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
12. [Screenshots Index](#12-screenshots-index)

---

## 1. Server Preparation — Disks

### VirtualBox VM Setup

Before starting the installation, create the VM with:

- **RAM:** 2 GB minimum
- **Disk 1:** 20 GB → mount point `/`
- **Disk 2:** 10 GB → mount point `/home`
- **Network Adapter 1:** NAT
- **Network Adapter 2:** Internal Network — `intnet`

### Disk Partitioning During Installation

During the Ubuntu Server 22.04 installer, choose **Custom storage layout**:

- `/dev/sda` (20 GB) → mount point `/`
- `/dev/sdb` (10 GB) → mount point `/home`

### Verify After Boot

```bash
lsblk
df -h
```

> 📸  
> ![lsblk showing sda at / and sdb at /home](Images/images-001_lsblk_discos.png)

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

> ⚠️ Remove any line with `127.0.1.1` pointing to the old hostname.

> 📸 `images/images-002_hostname_y_etc_hosts.png` — hostnamectl and /etc/hosts

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
ip a
```

> 📸 `images/images-003-Netplan_configuration_srv1.png` — netplan file and ip a output

### 2.4 Disable systemd-resolved

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

> 📸 `images/images-004-resolv.conf_srv1.png` — /etc/resolv.conf content

---

## 3. Samba AD DC — Installation and Provisioning

### 3.1 Install Packages

> ⚠️ **Critical:** Always install packages BEFORE provisioning.

```bash
sudo apt install -y samba samba-ad-dc krb5-user samba-dsdb-modules samba-vfs-modules winbind
```

When prompted for the Kerberos realm, enter `DEATHSTAR.LOCAL` (uppercase).

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

> 📸 `images/images-006-Domain_Provision_srv1.png` — full output of samba-tool domain provision

### 3.4 Copy Kerberos Config and Enable Service

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
sudo systemctl status samba-ad-dc
```

> 📸 `images/images-005-kerberos_reino.png` — Kerberos realm configuration

---

## 4. Domain and Kerberos Verification

### 4.1 Check DNS Records

```bash
host -t A srv01.deathstar.local
host -t SRV _ldap._tcp.deathstar.local
host -t SRV _kerberos._tcp.deathstar.local
```

### 4.2 Check Active Ports

```bash
sudo ss -tulnp | grep -E ':(53|88|389|445|636|3268)'
```

### 4.3 Remove Duplicate DNS Entry (NAT IP)

If `host -t A srv01.deathstar.local` shows the NAT IP (10.0.2.15), remove it:

```bash
sudo samba-tool dns delete 127.0.0.1 deathstar.local srv01 A 10.0.2.15 -U Administrator%Admin_21
```

### 4.4 Test Kerberos

> ⚠️ Always use UPPERCASE for the realm.

```bash
kinit Administrator@DEATHSTAR.LOCAL
klist
```

> 📸 `images/images-008_Kerberos_pruebas_srv1.png` — klist with Administrator ticket

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

> 📸 `images/images-009_creacion_usuarios_carpetas.png` — user list, group list and jedis members

---

## 6. Shared Folders with Permissions

### 6.1 Create Directories

```bash
sudo mkdir -p /deathstar/blueprints
sudo mkdir -p /sw/family
sudo mkdir -p /sw/force
```

### 6.2 Set Permissions

```bash
# blueprints — private, leia only
sudo chown root:root /deathstar/blueprints
sudo chmod 770 /deathstar/blueprints
setfacl -m u:leia:rwx /deathstar/blueprints

# family — public, anakin denied
sudo chown root:"Domain Users" /sw/family
sudo chmod 775 /sw/family
setfacl -m u:anakin:--- /sw/family

# force — public, read-only, leia denied
sudo chown root:"Domain Users" /sw/force
sudo chmod 775 /sw/force
setfacl -m u:anakin:r-x /sw/force
setfacl -m u:yoda:r-x /sw/force
setfacl -m g:jedis:r-x /sw/force
setfacl -m u:leia:--- /sw/force
```

### 6.3 Configure Samba Shares

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

> 📸 `images/images-010_smb.conf.png` — smb.conf shares configuration

### 6.4 Install Apache for Web Access

```bash
sudo apt install -y apache2
sudo ln -s /sw/family /var/www/html/family
sudo ln -s /sw/force /var/www/html/force
sudo systemctl enable --now apache2
```

### 6.5 Restart Samba and Verify

```bash
sudo systemctl restart samba-ad-dc
```

```bash
# blueprints: leia in, anakin denied
smbclient //localhost/blueprints -U leia%Admin_21
smbclient //localhost/blueprints -U anakin%Admin_21

# family: anakin denied, leia in
smbclient //localhost/family -U anakin%Admin_21
smbclient //localhost/family -U leia%Admin_21

# force: leia denied, anakin and yoda in
smbclient //localhost/force -U leia%Admin_21
smbclient //localhost/force -U anakin%Admin_21
smbclient //localhost/force -U yoda%Admin_21
```

> 📸 `images/images-014_comprobaciones_permisos_usuarios.png` — smbclient access verification per user

---

## 7. Linux Client — uc01

### 7.1 Network Configuration

Configure network via GUI (NetworkManager):

- Address: `192.168.1.3`
- Netmask: `255.255.255.0`
- DNS: `192.168.1.1`

```bash
sudo nano /etc/resolv.conf
```

```
nameserver 192.168.1.1
search deathstar.local
```

> 📸 `images/images-015_Configuracion_Red_Cliente.png` — network settings on uc01
> 📸 `images/images-016_resolv.conf_client.png` — /etc/resolv.conf on client

### 7.2 Install Required Packages

```bash
sudo apt install -y realmd sssd sssd-tools adcli samba-common-bin packagekit krb5-user
```

> 📸 `images/images-017_Instalacion_realmssd_Client.png` — package installation

### 7.3 Discover and Join Domain

```bash
realm discover DEATHSTAR.LOCAL
```

> 📸 `images/images-018_Realm_Discover.png` — realm discover output

```bash
sudo realm join DEATHSTAR.LOCAL -U Administrator
```

> 📸 `images/images-019_Realm_Join.png` — realm join with no errors

### 7.4 Configure SSSD and Verify

```bash
sudo nano /etc/sssd/sssd.conf
# Set: use_fully_qualified_names = False

sudo systemctl restart sssd
sudo pam-auth-update --enable mkhomedir
```

```bash
realm list
id leia@DEATHSTAR.LOCAL
id anakin@DEATHSTAR.LOCAL
kinit leia@DEATHSTAR.LOCAL
```

> 📸 `images/images-020_Prueba_Usuario_Cliente.png` — realm list and id commands
> 📸 `images/images-021_Prueba_Usuarios_Cliente_v2.png` — domain user login on client GUI

---

## 8. AWS Server — bespin07 (cloud07.city)

### 8.1 AWS Infrastructure Setup

**Create VPC:**

- Name: `LAB07-VPC`
- CIDR: `10.0.0.0/16`
- 1 public subnet, no private subnets
- Enable DNS hostnames and resolution

> 📸 `images/images-022_Creacion_VPC.png` — VPC creation
> 📸 `images/images-023_Verificacion_VPC.png` — VPC details
> 📸 `images/images-024_Habilitar_IP_Automatica_VPC.png` — enabling auto-assign public IP

**Create Security Group (LAB07-SG)** with these inbound rules:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| SSH | TCP | 22 | 0.0.0.0/0 |
| DNS (TCP) | TCP | 53 | 10.0.0.0/20 |
| DNS (UDP) | UDP | 53 | 10.0.0.0/20 |
| Custom TCP | TCP | 88 | 10.0.0.0/20 |
| Custom UDP | UDP | 88 | 10.0.0.0/20 |
| Custom TCP | TCP | 389 | 10.0.0.0/20 |
| Custom TCP | TCP | 636 | 10.0.0.0/20 |
| Custom TCP | TCP | 445 | 10.0.0.0/20 |
| Custom TCP | TCP | 3268 | 10.0.0.0/20 |
| Custom TCP | TCP | 3269 | 10.0.0.0/20 |
| RDP | TCP | 3389 | 0.0.0.0/0 |
| All traffic | All | All | 10.0.0.0/20 |

> 📸 `images/images-025_Grupos_De_Seguridad_Y_Reglas.png` — security group rules

**Launch EC2 Instance:**

- Name: `Bespin07`
- AMI: Ubuntu Server 24.04 LTS
- Type: t3.small
- Key pair: vockey
- VPC: LAB07-VPC
- Security group: LAB07-SG
- Storage: 30 GB gp3
- Private IP: `10.0.1.10` (static)

Assign Elastic IP and associate to instance.

### 8.2 SSH Connection from Windows

Fix PEM file permissions:

```
icacls C:\labsuser.pem /inheritance:r
icacls C:\labsuser.pem /remove "BUILTIN\Usuarios"
icacls C:\labsuser.pem /remove "NT AUTHORITY\Usuarios autentificados"
icacls C:\labsuser.pem /grant:r "migue:R"
```

Connect:

```
ssh -i C:\labsuser.pem ubuntu@100.55.166.123
```

> 📸 `images/images-026_ssh_desde_windows_a_ec2.png` — SSH session from Windows to EC2

### 8.3 Server Configuration

```bash
sudo hostnamectl set-hostname bespin07.cloud07.city
sudo nano /etc/hosts
```

```
127.0.0.1       localhost
10.0.1.10       bespin07.cloud07.city bespin07
```

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo unlink /etc/resolv.conf
sudo nano /etc/resolv.conf
```

```
nameserver 127.0.0.1
nameserver 8.8.8.8
search cloud07.city
```

> 📸 `images/images-026_ssh_hostname_etc_hosts_bespin07.png` — hostname and hosts on bespin07

### 8.4 Install Samba and Provision Domain

```bash
sudo apt install -y samba samba-ad-dc krb5-user samba-dsdb-modules samba-vfs-modules winbind
```

Realm: `CLOUD07.CITY`

```bash
sudo systemctl stop smbd nmbd winbind 2>/dev/null || true
sudo systemctl disable smbd nmbd winbind 2>/dev/null || true
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

> 📸 `images/images-027_Aprovisionamiento_bespin01.png` — domain provision output

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
```

> 📸 `images/images-029_UnMask_Status_bespin01.png` — samba-ad-dc status active

```bash
host -t A bespin07.cloud07.city 127.0.0.1
kinit Administrator@CLOUD07.CITY
klist
```

> 📸 `images/images-028_Dns_Kinit_Bespin07.png` — DNS check and Kerberos ticket

### 8.5 Create Users and Trap Share

```bash
sudo samba-tool user create lando Admin_21
sudo samba-tool user create boba Admin_21
sudo samba-tool user list
```

> 📸 `images/images-030_Creacion_Usuarios_bespin01.png` — user list on bespin07

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

# lando should succeed
smbclient //localhost/trap -U lando%Admin_21

# boba should be denied
smbclient //localhost/trap -U boba%Admin_21
```

> 📸 `images/images-031_Creacion_Carpetas_bespin01.png` — mkdir and permissions
> 📸 `images/images-031_Creacion_Carpetas_Y_Comprobaciones_bespin01.png` — lando in, boba denied

---

## 9. SSH Access and Process Control

### 9.1 Connect from Physical Machine to srv01

```
ssh administrador@127.0.0.1 -p 2222
```

### 9.2 Install and Launch sl

```bash
sudo apt install -y sl
sl
```

### 9.3 Pause and Resume with POSIX Signals

Open a second SSH terminal and:

```bash
# Find PID
pgrep -x sl

# SIGSTOP (19) — pause the process
kill -19 $(pgrep -x sl)
```

> 📸 `images/images-011_Sl_Stop.png` — sl process paused with SIGSTOP

```bash
# SIGCONT (18) — resume the process
kill -18 $(pgrep -x sl)
```

> 📸 `images/images-012_Sl_Continue.png` — sl process resumed with SIGCONT

---

## 10. Scheduled Task with Cron

### 10.1 Create the Script

```bash
sudo nano /usr/local/bin/r2d2.sh
```

```bash
#!/bin/bash
# r2d2.sh — Creates luke.sky file
# Scheduled: every day at 10:30

touch /home/administrador/luke.sky
echo "luke.sky created on $(date)" >> /var/log/r2d2.log
```

```bash
sudo chmod +x /usr/local/bin/r2d2.sh
```

> 📸 `images/images-013_script_r2d2.png` — cat of r2d2.sh

### 10.2 Add to crontab

```bash
crontab -e
```

Add:

```
30 10 * * * /usr/local/bin/r2d2.sh
```

```bash
crontab -l
```

> 📸 `images/images-013_script.png` — crontab -l showing 10:30 task

### 10.3 Test Manually

```bash
/usr/local/bin/r2d2.sh
ls -la /home/administrador/luke.sky
cat /var/log/r2d2.log
```

---

## 11. Trust Relationship Between Domains

A bidirectional trust is established between `DEATHSTAR.LOCAL` (srv01) and `LAB07.LAN` (ls07).

### 11.1 Verify Connectivity Between DCs

```bash
# From srv01
ping -c 4 192.168.1.2

# From ls07
ping -c 4 192.168.1.1
```

### 11.2 Configure DNS Forwarding

**On srv01** — add to `[global]` in `/etc/samba/smb.conf`:

```ini
dns forwarder = 192.168.1.2 8.8.8.8
```

**On ls07** — add to `[global]` in `/etc/samba/smb.conf`:

```ini
dns forwarder = 192.168.1.1 8.8.8.8
```

Restart Samba on both:

```bash
sudo systemctl restart samba-ad-dc
```

> 📸 `images/images-032_ParaElTrust_Añado_DnsForwarder_a_smb-conf_srv01.png` — dns forwarder on srv01
> 📸 `images/images-033_ParaElTrust_Añado_DnsForwarder_a_smb-conf_ls07(servidor2).png` — dns forwarder on ls07

Verify cross-resolution:

```bash
# From srv01
host -t SRV _ldap._tcp.lab07.lan

# From ls07
host -t SRV _ldap._tcp.deathstar.local
```

### 11.3 Create the Trust

Run on srv01:

```bash
sudo samba-tool domain trust create LAB07.LAN \
  --type=external \
  --direction=both \
  -U Administrator
```

> 📸 `images/images-034_Trust_Create.png` — trust create output with WERR_OK

### 11.4 Verify the Trust

```bash
sudo samba-tool domain trust list
sudo samba-tool domain trust validate LAB07.LAN -U Administrator
```

> ⚠️ A `NETLOGON_CONTROL_TC_VERIFY` warning during remote validation is normal with Samba 4. What matters is `TRUST[WERR_OK]` in LocalValidation.

> 📸 `images/images-035_Trust_Comprobaciones_srv01.png` — trust list and validate on srv01
> 📸 `images/images-036_Trust_Comprobaciones_ls07(SERVER2).png` — trust list on ls07

### 11.5 Test Cross-Domain Authentication

```bash
kinit Administrator@LAB07.LAN
klist
```

---

## 12. Screenshots Index

| No. | File | Section | What it shows |
|-----|------|---------|---------------|
| 01 | `images-001_lsblk_discos` | Disks | lsblk with sda at / and sdb at /home |
| 02 | `images-002_hostname_y_etc_hosts` | Network | hostnamectl and /etc/hosts |
| 03 | `images-003-Netplan_configuration_srv1` | Network | Netplan config |
| 04 | `images-004-resolv.conf_srv1` | Network | /etc/resolv.conf on srv01 |
| 05 | `images-005-kerberos_reino` | Kerberos | Kerberos realm config |
| 06 | `images-006-Domain_Provision_srv1` | Samba | Full domain provision output |
| 07 | `images-007-smb.conf_srv1` | Samba | smb.conf configuration |
| 08 | `images-008_Kerberos_pruebas_srv1` | Kerberos | klist with Administrator ticket |
| 09 | `images-009_creacion_usuarios_carpetas` | Users | user list and group jedis |
| 10 | `images-010_smb.conf` | Shares | smb.conf shares section |
| 11 | `images-011_Sl_Stop` | SSH | sl paused with SIGSTOP |
| 12 | `images-012_Sl_Continue` | SSH | sl resumed with SIGCONT |
| 13 | `images-013_script` | Cron | crontab -l at 10:30 |
| 14 | `images-013_script_r2d2` | Cron | cat of r2d2.sh |
| 15 | `images-014_comprobaciones_permisos_usuarios` | Shares | smbclient access verification |
| 16 | `images-015_Configuracion_Red_Cliente` | Client | network settings on uc01 |
| 17 | `images-016_resolv.conf_client` | Client | /etc/resolv.conf on client |
| 18 | `images-017_Instalacion_realmssd_Client` | Client | package installation |
| 19 | `images-018_Realm_Discover` | Client | realm discover output |
| 20 | `images-019_Realm_Join` | Client | realm join success |
| 21 | `images-020_Prueba_Usuario_Cliente` | Client | realm list and id commands |
| 22 | `images-021_Prueba_Usuarios_Cliente_v2` | Client | domain user login on GUI |
| 23 | `images-022_Creacion_VPC` | AWS | VPC creation |
| 24 | `images-023_Verificacion_VPC` | AWS | VPC details |
| 25 | `images-024_Habilitar_IP_Automatica_VPC` | AWS | enabling auto-assign public IP |
| 26 | `images-025_Grupos_De_Seguridad_Y_Reglas` | AWS | security group rules |
| 27 | `images-026_ssh_desde_windows_a_ec2` | AWS | SSH from Windows to EC2 |
| 28 | `images-026_ssh_hostname_etc_hosts_bespin07` | AWS | hostname and hosts on bespin07 |
| 29 | `images-027_Aprovisionamiento_bespin01` | AWS | domain provision on bespin07 |
| 30 | `images-028_Dns_Kinit_Bespin07` | AWS | DNS and Kerberos on bespin07 |
| 31 | `images-029_UnMask_Status_bespin01` | AWS | samba-ad-dc status active |
| 32 | `images-030_Creacion_Usuarios_bespin01` | AWS | user list on bespin07 |
| 33 | `images-031_Creacion_Carpetas_bespin01` | AWS | mkdir and permissions |
| 34 | `images-031_Creacion_Carpetas_Y_Comprobaciones_bespin01` | AWS | lando in, boba denied |
| 35 | `images-032_ParaElTrust_Añado_DnsForwarder_a_smb-conf_srv01` | Trust | dns forwarder on srv01 |
| 36 | `images-033_ParaElTrust_Añado_DnsForwarder_a_smb-conf_ls07(servidor2)` | Trust | dns forwarder on ls07 |
| 37 | `images-034_Trust_Create` | Trust | trust create with WERR_OK |
| 38 | `images-035_Trust_Comprobaciones_srv01` | Trust | trust list and validate on srv01 |
| 39 | `images-036_Trust_Comprobaciones_ls07(SERVER2)` | Trust | trust list on ls07 |
