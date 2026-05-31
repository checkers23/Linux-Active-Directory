# 📖 Full Documentation — DeathStar S.A.

> Step-by-step guide to set up an Ubuntu Server as an Active Directory Domain Controller (Samba 4), join a Linux client, configure shared folders, a secondary AWS server, SSH process control, a cron task, and a trust relationship between domains.
>
> **Repository:** https://github.com/checkers23/Linux-Active-Directory

---

## Table of Contents

1. [Server Preparation — Disks](#1-server-preparation--disks)
2. [Network and Hostname — srv01](#2-network-and-hostname--srv01)
3. [Samba AD DC — Installation and Provisioning](#3-samba-ad-dc--installation-and-provisioning)
4. [Domain and Kerberos Verification](#4-domain-and-kerberos-verification)
5. [Domain Users and Groups](#5-domain-users-and-groups)
6. [Shared Folders with Permissions](#6-shared-folders-with-permissions)
7. [Linux Client — uc01](#7-linux-client--uc01)
8. [AWS Server — bespin07 (cloud07.city)](#8-aws-server--bespin07-cloud07city)
9. [SSH Access and Process Control](#9-ssh-access-and-process-control)
10. [Scheduled Task with Cron](#10-scheduled-task-with-cron)
11. [Trust Relationship Between Domains](#11-trust-relationship-between-domains)

---

## 1. Server Preparation — Disks

Before starting the installation, create the VM in VirtualBox with two disks:

- **Disk 1:** 20 GB → mount point `/`
- **Disk 2:** 10 GB → mount point `/home`

During the Ubuntu Server installer, choose **Custom storage layout** and assign each disk to its mount point.

### Verify After Boot

```bash
lsblk
df -h
```

![lsblk showing two disks](images-001_lsblk_discos.png)

---

## 2. Network and Hostname — srv01

### 2.1 Set Hostname

```bash
sudo hostnamectl set-hostname srv01.deathstar.local
hostname -f
```

### 2.2 Edit /etc/hosts

```bash
sudo nano /etc/hosts
```

```
127.0.0.1       localhost
192.168.1.1     srv01.deathstar.local srv01
```

> ⚠️ Remove any line with `127.0.1.1`. During setup the IP was initially set to `192.168.10.1` and later corrected to `192.168.1.1`.

![hostname and /etc/hosts configuration](images-002_hostname_y_etc_hosts.png)

### 2.3 Configure Network Interfaces (Netplan)

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: yes
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.1.1/24
      nameservers:
        addresses:
          - 127.0.0.1
          - 8.8.8.8
        search:
          - deathstar.local
```

```bash
sudo netplan apply
```

![Netplan configuration file](images-003-Netplan_configuration_srv1.png)

### 2.4 Disable systemd-resolved and Configure DNS

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo unlink /etc/resolv.conf
sudo nano /etc/resolv.conf
```

```
nameserver 127.0.0.1
nameserver 8.8.8.8
search deathstar.local
```

![resolv.conf configuration](images-004-resolv_conf_srv1.png)

---

## 3. Samba AD DC — Installation and Provisioning

### 3.1 Install Packages

> ⚠️ **Critical:** Always install packages BEFORE provisioning. Installing samba-ad-dc after provisioning causes the domain to be incomplete.

```bash
sudo apt install -y samba samba-ad-dc krb5-user samba-dsdb-modules samba-vfs-modules winbind
```

When prompted for the Kerberos realm, enter `DEATHSTAR.LOCAL` in uppercase.

![Kerberos realm configuration during install](images-005-kerberos_reino.png)

### 3.2 Stop Default Services and Clean Config

```bash
sudo systemctl stop smbd nmbd winbind 2>/dev/null || true
sudo systemctl disable smbd nmbd winbind 2>/dev/null || true
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
sudo mv /etc/krb5.conf /etc/krb5.conf.bak
```

### 3.3 Provision the Domain

```bash
sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=DEATHSTAR.LOCAL \
  --domain=DEATHSTAR \
  --adminpass='Admin_21' \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL
```

![Domain provision output](images-006-Domain_Provision_srv1.png)

### 3.4 Copy Kerberos Config and Enable Service

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
sudo systemctl status samba-ad-dc
```

### 3.5 smb.conf Result

![smb.conf after provisioning](images-007-smb_conf_srv1.png)

---

## 4. Domain and Kerberos Verification

```bash
sudo samba-tool domain level show
sudo samba-tool domain info 127.0.0.1
host -t A srv01.deathstar.local
host -t SRV _ldap._tcp.deathstar.local
host -t SRV _kerberos._tcp.deathstar.local
```

### Remove Duplicate DNS Entry (NAT IP)

If `host -t A` shows the NAT IP `10.0.2.15`, remove it:

```bash
sudo samba-tool dns delete 127.0.0.1 deathstar.local srv01 A 10.0.2.15 -U Administrator%Admin_21
```

### Test Kerberos

> ⚠️ Always use UPPERCASE for the realm. `kinit administrator@deathstar.local` will fail — use `kinit Administrator@DEATHSTAR.LOCAL`.

```bash
kinit Administrator@DEATHSTAR.LOCAL
klist
```

![Kerberos verification and domain info](images-008_Kerberos_pruebas_srv1.png)

---

## 5. Domain Users and Groups

```bash
sudo samba-tool user create leia Admin_21
sudo samba-tool user create anakin Admin_21
sudo samba-tool user create yoda Admin_21
sudo samba-tool group add jedis
sudo samba-tool group addmembers jedis anakin,yoda
sudo samba-tool user list
sudo samba-tool group list
sudo samba-tool group listmembers jedis
```

![User and group creation](images-009_creacion_usuarios.png)

![Users, groups and folder creation](images-009_creacion_usuarios_carpetas.png)

---

## 6. Shared Folders with Permissions

### 6.1 Create Directories

```bash
sudo mkdir -p /deathstar/blueprints
sudo mkdir -p /sw/family
sudo mkdir -p /sw/force
```

### 6.2 Set Permissions with ACLs

```bash
# blueprints — private, leia only
sudo chown root:root /deathstar/blueprints
sudo chmod 770 /deathstar/blueprints
sudo setfacl -m u:leia:rwx /deathstar/blueprints

# family — public, anakin denied
sudo chown root:"Domain Users" /sw/family
sudo chmod 775 /sw/family
sudo setfacl -m u:anakin:--- /sw/family

# force — public, read-only, leia denied
sudo chown root:"Domain Users" /sw/force
sudo chmod 775 /sw/force
sudo setfacl -m u:anakin:r-x /sw/force
sudo setfacl -m u:yoda:r-x /sw/force
sudo setfacl -m g:jedis:r-x /sw/force
sudo setfacl -m u:leia:--- /sw/force
```

### 6.3 Configure Samba Shares in smb.conf

```bash
sudo nano /etc/samba/smb.conf
```

Add at the end:

```ini
[blueprints]
    path = /deathstar/blueprints
    browseable = no
    read only = no
    valid users = DEATHSTAR\leia

[family]
    path = /sw/family
    browseable = yes
    read only = no
    invalid users = DEATHSTAR\anakin
    guest ok = yes

[force]
    path = /sw/force
    browseable = yes
    read only = yes
    valid users = DEATHSTAR\anakin DEATHSTAR\yoda @DEATHSTAR\jedis
    invalid users = DEATHSTAR\leia
    guest ok = yes
```

![smb.conf with all shares configured](images-010_smb_conf.png)

### 6.4 Install Apache for Web Access

```bash
sudo apt install -y apache2
sudo ln -s /sw/family /var/www/html/family
sudo ln -s /sw/force /var/www/html/force
sudo systemctl enable --now apache2
```

### 6.5 Restart Samba and Verify Access

```bash
sudo systemctl restart samba-ad-dc

smbclient //localhost/blueprints -U leia%Admin_21
smbclient //localhost/blueprints -U anakin%Admin_21
smbclient //localhost/family -U anakin%Admin_21
smbclient //localhost/family -U leia%Admin_21
smbclient //localhost/force -U leia%Admin_21
smbclient //localhost/force -U anakin%Admin_21
smbclient //localhost/force -U yoda%Admin_21
```

![Access verification per user — leia in blueprints, anakin denied, force permissions correct](images-014_comprobaciones_permisos_usuarios.png)

---

## 7. Linux Client — uc01

### 7.1 Network Configuration via GUI

Open Settings → Network → IPv4 → Manual:

- Address: `192.168.1.3`
- Netmask: `255.255.255.0`
- DNS: `192.168.1.1`

![Network configuration on uc01](images-015_Configuracion_Red_Cliente.png)

### 7.2 Configure resolv.conf

```bash
sudo nano /etc/resolv.conf
```

```
nameserver 192.168.1.1
search deathstar.local
```

![resolv.conf on client](images-016_resolv_conf_client.png)

### 7.3 Install Required Packages

```bash
sudo apt install -y realmd sssd sssd-tools adcli samba-common-bin packagekit krb5-user
```

![Package installation on client](images-017_Instalacion_realmssd_Client.png)

### 7.4 Discover the Domain

```bash
realm discover DEATHSTAR.LOCAL
```

![realm discover output](images-018_Realm_Discover.png)

### 7.5 Join the Domain

```bash
sudo realm join DEATHSTAR.LOCAL -U Administrator
realm list
id leia@DEATHSTAR.LOCAL
id anakin@DEATHSTAR.LOCAL
```

![realm join and verification](images-019_Realm_Join.png)

### 7.6 Configure SSSD

```bash
sudo nano /etc/sssd/sssd.conf
# Set: use_fully_qualified_names = False
sudo systemctl restart sssd
sudo pam-auth-update --enable mkhomedir
```

### 7.7 Test Domain Users

```bash
kinit leia@DEATHSTAR.LOCAL
kinit anakin@DEATHSTAR.LOCAL
kinit yoda@DEATHSTAR.LOCAL
```

![Kerberos tickets for domain users on client](images-020_Prueba_Usuario_Cliente.png)

### 7.8 Graphical Login with Domain User

Log out and log in as `leia@DEATHSTAR.LOCAL` with password `Admin_21`.

![leia logged in on uc01 via domain — whoami shows leia@deathstar.local](images-021_Prueba_Usuarios_Cliente_v2.png)

---

## 8. AWS Server — bespin07 (cloud07.city)

### 8.1 Create VPC in AWS Academy

- Name: `LAB07-VPC`
- CIDR: `10.0.0.0/16`
- 1 public subnet, DNS enabled

![VPC creation form](images-022_Creacion_VPC.png)

![VPC details and resource map](images-023_Verificacion_VPC.png)

### 8.2 Enable Auto-assign Public IP on Subnet

VPC → Subnets → Select LAB07-VPC-subnet-public1 → Actions → Edit subnet settings → Enable auto-assign IPv4.

![Auto-assign public IP enabled](images-024_Habilitar_IP_Automatica_VPC.png)

### 8.3 Create Security Group (LAB07-SG)

Inbound rules: SSH(22), DNS TCP/UDP(53), Kerberos TCP/UDP(88), LDAP(389), SMB(445), LDAPS(636), Global Catalog(3268/3269), RDP(3389), All traffic from VPC(10.0.0.0/20).

![Security group LAB07-SG with all rules](images-025_Grupos_De_Seguridad_Y_Reglas.png)

### 8.4 SSH Connection from Windows

Fix PEM file permissions and connect:

```
icacls C:\labsuser.pem /inheritance:r
icacls C:\labsuser.pem /remove "BUILTIN\Usuarios"
icacls C:\labsuser.pem /remove "NT AUTHORITY\Usuarios autentificados"
ssh -i C:\labsuser.pem ubuntu@100.55.166.123
```

![SSH from Windows to EC2 instance — connected as ubuntu](images-026_ssh_desde_windows_a_ec2.png)

### 8.5 Configure Hostname and Hosts

```bash
sudo hostnamectl set-hostname bespin07.cloud07.city
sudo nano /etc/hosts
# Add: 10.0.1.10   bespin07.cloud07.city bespin07
```

![hostname and /etc/hosts on bespin07](images-026_ssh_hostname_etc_hosts_bespin07.png)

### 8.6 Install Samba and Provision Domain

```bash
sudo apt install -y samba samba-ad-dc krb5-user samba-dsdb-modules samba-vfs-modules winbind

sudo systemctl stop smbd nmbd winbind 2>/dev/null || true
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
sudo mv /etc/krb5.conf /etc/krb5.conf.bak

sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=CLOUD07.CITY \
  --domain=CLOUD07 \
  --adminpass='Admin_21' \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL
```

![Domain provision on bespin07](images-027_Aprovisionamiento_bespin01.png)

### 8.7 Enable Service and Verify

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
sudo systemctl status samba-ad-dc
```

![samba-ad-dc active and running on bespin07](images-029_UnMask_Status_bespin01.png)

```bash
host -t A bespin07.cloud07.city 127.0.0.1
kinit Administrator@CLOUD07.CITY
klist
```

![DNS check and Kerberos ticket on bespin07](images-028_Dns_Kinit_Bespin07.png)

### 8.8 Create Users and Trap Share

```bash
sudo samba-tool user create lando Admin_21
sudo samba-tool user create boba Admin_21
sudo samba-tool user list
```

![lando and boba created on bespin07](images-030_Creacion_Usuarios_bespin01.png)

```bash
sudo mkdir -p /city/trap/MPI
sudo chown root:"Domain Users" /city/trap
sudo chmod 770 /city/trap
```

Add to `/etc/samba/smb.conf`:

```ini
[trap]
    path = /city/trap
    browseable = no
    read only = no
    valid users = CLOUD07\lando
    invalid users = CLOUD07\boba
```

```bash
sudo systemctl restart samba-ad-dc
```

![mkdir /city/trap/MPI and permissions](images-031_Creacion_Carpetas_bespin01.png)

```bash
smbclient //localhost/trap -U lando%Admin_21
smbclient //localhost/trap -U boba%Admin_21
```

![lando accesses trap, boba gets ACCESS DENIED](images-031_Creacion_Carpetas_Y_Comprobaciones_bespin01.png)

---

## 9. SSH Access and Process Control

### 9.1 Connect from Physical Machine to srv01

```
ssh administrador@127.0.0.1 -p 2222
```

Port forwarding configured in VirtualBox: Host 2222 → Guest 22.

### 9.2 Install and Launch sl

```bash
sudo apt install -y sl
sl
```

### 9.3 Pause and Resume with POSIX Signals

Open a second SSH terminal and:

```bash
# SIGSTOP (19) — pause the process
kill -19 $(pgrep -x sl)
```

![sl train stopped with SIGSTOP — [1]+ Stopped sl visible](images-011_Sl_Stop.png)

```bash
# SIGCONT (18) — resume the process
kill -18 $(pgrep -x sl)
```

![sl train resumed with SIGCONT](images-012_Sl_Continue.png)

---

## 10. Scheduled Task with Cron

### 10.1 Create the Script

```bash
sudo nano /usr/local/bin/r2d2.sh
```

```bash
#!/bin/bash
touch /home/administrador/luke.sky
echo "luke.sky created on $(date)" >> /var/log/r2d2.log
```

```bash
sudo chmod +x /usr/local/bin/r2d2.sh
```

![r2d2.sh script content](images-013_script_r2d2.png)

### 10.2 Add to crontab at 10:30

```bash
crontab -e
# Add: 30 10 * * * /usr/local/bin/r2d2.sh
crontab -l
```

### 10.3 Test Manually

```bash
sudo /usr/local/bin/r2d2.sh
ls -la /home/administrador/luke.sky
```

![crontab at 10:30, script execution and luke.sky file created](images-013_script.png)

---

## 11. Trust Relationship Between Domains

A bidirectional trust is established between `DEATHSTAR.LOCAL` (srv01, 192.168.1.1) and `LAB07.LAN` (ls07, 192.168.1.2).

### 11.1 Configure DNS Forwarding

**On srv01** — add to `[global]` in `/etc/samba/smb.conf`:

```ini
dns forwarder = 192.168.1.2 8.8.8.8
```

![dns forwarder added to srv01 smb.conf](images-032_ParaElTrust_DnsForwarder_srv01.png)

**On ls07** — add to `[global]` in `/etc/samba/smb.conf`:

```ini
dns forwarder = 192.168.1.1 8.8.8.8
```

![dns forwarder added to ls07 smb.conf](images-033_ParaElTrust_DnsForwarder_ls07.png)

Restart Samba on both servers and verify cross-resolution:

```bash
# From srv01
host -t SRV _ldap._tcp.lab07.lan

# From ls07
host -t SRV _ldap._tcp.deathstar.local
```

### 11.2 Create the Trust

Run on srv01:

```bash
sudo samba-tool domain trust create LAB07.LAN \
  --type=external \
  --direction=both \
  -U Administrator
```

![Trust creation with WERR_OK on both directions](images-034_Trust_Create.png)

### 11.3 Verify the Trust

```bash
sudo samba-tool domain trust list
sudo samba-tool domain trust validate LAB07.LAN -U Administrator
kinit Administrator@LAB07.LAN
klist
```

> ⚠️ The `NETLOGON_CONTROL_TC_VERIFY` warning during remote validation is normal with Samba 4. What matters is `TRUST[WERR_OK]` in LocalValidation and the successful cross-domain kinit.

![Trust list Direction[BOTH], validate WERR_OK, and cross-domain kinit to LAB07.LAN](images-035_Trust_Comprobaciones_srv01.png)

**Verification from ls07:**

```bash
sudo samba-tool domain trust list
```

![ls07 shows deathstar.local in trust list — Direction[BOTH]](images-036_Trust_Comprobaciones_ls07.png)

---

*Documentation by MPI — DeathStar S.A. | https://github.com/checkers23/Linux-Active-Directory*
