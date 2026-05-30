# 📖 Full Documentation — DeathStar S.A.

> Step-by-step guide to set up an Ubuntu Server as an Active Directory Domain Controller (Samba 4), join a Linux client, configure shared folders, a secondary server, SSH process control, a cron task, and a trust relationship between domains.
>
> ⚠️ Replace `XX` with your classroom number everywhere in this document.

---

## Table of Contents

1. [Server Preparation — Disks](#1-server-preparation--disks)
2. [Network and Hostname — srv01](#2-network-and-hostname--srvwxx)
3. [NTP Time Synchronisation](#3-ntp-time-synchronisation)
4. [Samba AD DC — Installation and Provisioning](#4-samba-ad-dc--installation-and-provisioning)
5. [Domain and Kerberos Verification](#5-domain-and-kerberos-verification)
6. [Domain Users and Groups](#6-domain-users-and-groups)
7. [Shared Folders with Permissions](#7-shared-folders-with-permissions)
8. [Linux Client — uc01](#8-linux-client--ucxx)
9. [Secondary Server — bespin01 (cloud01.city)](#9-secondary-server--bespinxx-cloudxxcity)
10. [SSH Access and Process Control](#10-ssh-access-and-process-control)
11. [Scheduled Task with Cron](#11-scheduled-task-with-cron)
12. [Trust Relationship Between Domains](#12-trust-relationship-between-domains)
13. [Screenshots Index](#13-screenshots-index)

---

## 1. Server Preparation — Disks

### VirtualBox VM Setup

Before starting the installation, create the VM with:

- **RAM:** 2 GB minimum
- **Disk 1:** 20 GB
- **Disk 2:** 10 GB (add via *Settings → Storage → Add Hard Disk*)
- **Network Adapter 1:** NAT
- **Network Adapter 2:** Internal Network — name it exactly `intnet`

### Disk Partitioning During Installation

During the Ubuntu Server 22.04 installer, when it reaches storage configuration, choose **Custom storage layout**:

- `/dev/sda` (20 GB) → mount point `/`
- `/dev/sdb` (10 GB) → mount point `/home`

The installer writes `/etc/fstab` automatically. No extra steps needed.

### Verify After Boot

```bash
lsblk
df -h
cat /etc/fstab
```

Expected `lsblk` output:
```
sda    20G  disk
└─sda1 20G  part  /
sdb    10G  disk
└─sdb1 10G  part  /home
```

> 📸 `images/01_lsblk_disks.png` — lsblk showing sda at / and sdb at /home
> 📸 `images/02_df_home.png` — df -h confirming /home on secondary disk

---

## 2. Network and Hostname — srv01

### 2.1 Set Hostname

```bash
sudo hostnamectl set-hostname srv01.deathstar.local
hostname -f
# Expected: srv01.deathstar.local
```

> 📸 `images/03_hostname.png` — hostnamectl and hostname -f output

### 2.2 Edit /etc/hosts

```bash
sudo nano /etc/hosts
```

```
127.0.0.1       localhost
192.168.1.1     srv01.deathstar.local srv01
```

> ⚠️ Remove any line with `127.0.1.1` pointing to the old hostname.

```bash
cat /etc/hosts
```

> 📸 `images/04_hosts.png` — /etc/hosts with the internal IP

### 2.3 Configure Network Interfaces (Netplan)

The server has two adapters:

| Adapter | VirtualBox Type | Interface | IP | Purpose |
|---------|-----------------|-----------|-----|---------|
| Adapter 1 | NAT | `enp0s3` | `172.30.20.10/24` (static) | Internet / packages |
| Adapter 2 | Internal Network (`intnet`) | `enp0s8` | `192.168.1.1/24` (static) | AD domain traffic |

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 172.30.20.10/24
      routes:
        - to: default
          via: 172.30.20.1
      nameservers:
        addresses:
          - 8.8.8.8
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

> `enp0s3` handles internet access via the lab gateway. `enp0s8` is the AD internal network — DNS points to `127.0.0.1` because Samba will manage domain DNS on this server after provisioning.

```bash
sudo netplan apply
ip a
ping -c 4 8.8.8.8
```

> 📸 `images/05_netplan.png` — netplan file content
> 📸 `images/06_ip_a.png` — ip a showing both interfaces with correct IPs
> 📸 `images/07_ping_internet.png` — successful ping to 8.8.8.8

### 2.4 Disable IPv6

IPv6 can cause conflicts with Samba and Kerberos name resolution.

```bash
sudo nano /etc/sysctl.conf
```

Add at the end:

```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

```bash
sudo sysctl -p
ip a | grep inet6
# Should return nothing
```

> 📸 `images/08_ipv6_disabled.png` — sysctl -p and ip a with no inet6 addresses

### 2.5 Disable systemd-resolved

⚠️ **Critical:** Samba needs port 53 free for its own DNS. Ubuntu 22.04 runs `systemd-resolved` by default which occupies that port.

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

```bash
sudo ss -tulnp | grep :53
# Must return nothing
```

> 📸 `images/09_resolved_disabled.png` — ss -tulnp with nothing on port 53

---

## 3. NTP Time Synchronisation

Kerberos fails if clocks differ by more than 5 minutes. NTP is mandatory. We use classic `ntpd`, not chrony.

### 3.1 Remove chrony

```bash
sudo systemctl stop chronyd 2>/dev/null || true
sudo apt remove -y chrony
```

### 3.2 Install and Configure NTP

```bash
sudo apt install -y ntp
sudo nano /etc/ntp.conf
```

Ensure these lines are present (add if missing):

```
pool ntp.ubuntu.com iburst
pool pool.ntp.org iburst

restrict default nomodify nopeer noquery limited kod
restrict 127.0.0.1
restrict ::1
restrict 192.168.1.0 mask 255.255.255.0 nomodify nopeer
```

```bash
sudo systemctl restart ntp
sudo systemctl enable ntp
```

### 3.3 Configure AppArmor for Samba NTP Signing

Samba generates `/var/lib/samba/ntp_signd/` so that domain clients can sync time against this DC. We need to allow ntpd access to it:

```bash
sudo nano /etc/apparmor.d/usr.sbin.ntpd
```

Add inside the existing block:

```
/var/lib/samba/ntp_signd r,
/var/lib/samba/ntp_signd/** rw,
```

```bash
sudo systemctl restart apparmor
sudo systemctl restart ntp
```

### 3.4 Verify Sync

```bash
ntpq -p
timedatectl
```

The `*` in `ntpq -p` marks the active time source. `timedatectl` should show `System clock synchronized: yes`.

> 📸 `images/10_ntp_sync.png` — ntpq -p and timedatectl showing active sync

---

## 4. Samba AD DC — Installation and Provisioning

### 4.1 Install Packages

```bash
sudo apt install -y samba krb5-user samba-dsdb-modules samba-vfs-modules winbind
```

> When prompted for the Kerberos realm, enter `DEATHSTAR.LOCAL` (uppercase).

### 4.2 Stop Default Services and Clean Config

```bash
sudo systemctl stop smbd nmbd winbind 2>/dev/null || true
sudo systemctl disable smbd nmbd winbind 2>/dev/null || true
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig 2>/dev/null || true
sudo mv /etc/krb5.conf /etc/krb5.conf.bak 2>/dev/null || true
```

### 4.3 Provision the Domain

```bash
sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=DEATHSTAR.LOCAL \
  --domain=DEATHSTAR \
  --adminpass='P@ssw0rd2024!' \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL
```

| Flag | Meaning |
|------|---------|
| `--use-rfc2307` | Enables POSIX/UNIX attributes for Linux compatibility |
| `--realm` | Kerberos realm — must be UPPERCASE |
| `--domain` | NetBIOS domain name |
| `--adminpass` | Administrator password (must meet complexity requirements) |
| `--server-role=dc` | This server becomes a Domain Controller |
| `--dns-backend=SAMBA_INTERNAL` | Samba manages its own DNS |

> 📸 `images/11_domain_provision.png` — full output of samba-tool domain provision

### 4.4 Copy Kerberos Config

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

### 4.5 Enable the Service

```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
sudo systemctl status samba-ad-dc
```

> 📸 `images/12_samba_status.png` — systemctl status samba-ad-dc showing active (running)

### 4.6 Verify AD Ports Are Listening

```bash
sudo ss -tulnp | grep -E ':(53|88|389|445|636|3268)'
```

All these must appear:

```
53   DNS
88   Kerberos
389  LDAP
445  SMB
636  LDAPS
3268 Global Catalog
```

> 📸 `images/13_ad_ports.png` — ss output showing all AD ports active

---

## 5. Domain and Kerberos Verification

### 5.1 Check DNS Records

```bash
host -t SRV _kerberos._tcp.deathstar.local 127.0.0.1
host -t SRV _ldap._tcp.deathstar.local 127.0.0.1
host -t A srv01.deathstar.local 127.0.0.1
```

> 📸 `images/14_dns_check.png` — host commands returning correct SRV and A records

### 5.2 Obtain a Kerberos Ticket

```bash
kinit administrator@DEATHSTAR.LOCAL
# Password: P@ssw0rd2024!
klist
kdestroy
```

> 📸 `images/15_kinit_klist.png` — klist showing the administrator ticket

### 5.3 Test SMB Access

```bash
smbclient -L localhost -U administrator%P@ssw0rd2024!
```

> 📸 `images/16_smbclient.png` — share listing via smbclient

---

## 6. Domain Users and Groups

> Run all commands on `srv01`.

### 6.1 Create Users

```bash
sudo samba-tool user add leia P@ssw0rd2024!
sudo samba-tool user add anakin P@ssw0rd2024!
sudo samba-tool user add yoda P@ssw0rd2024!
```

### 6.2 Create Group and Add Members

```bash
sudo samba-tool group add jedis
sudo samba-tool group addmembers jedis anakin
sudo samba-tool group addmembers jedis yoda
```

### 6.3 Verify

```bash
sudo samba-tool user list
sudo samba-tool group listmembers jedis
```

> 📸 `images/17_users_groups.png` — user list and jedis group members

---

## 7. Shared Folders with Permissions

### 7.1 Create Directory Structure

```bash
sudo mkdir -p /deathstar/blueprints
sudo mkdir -p /sw/family
sudo mkdir -p /sw/force
```

---

### 7.2 Share `blueprints` — leia only, not public

**Exam:** `/deathstar/blueprints` — only `leia` can access, must not be public.

```bash
sudo chown root:"Domain Users" /deathstar/blueprints
sudo chmod 770 /deathstar/blueprints
```

Add to `/etc/samba/smb.conf`:

```bash
sudo nano /etc/samba/smb.conf
```

```ini
[blueprints]
    path = /deathstar/blueprints
    browseable = no
    read only = no
    valid users = DEATHSTAR\leia
    create mask = 0660
    directory mask = 0770
```

> 📸 `images/18_smb_blueprints.png` — blueprints section in smb.conf

---

### 7.3 Share `family` — anakin cannot access, public and visible in browser

**Exam:** `/sw/family` — `anakin` must NOT access, public, visible in browser.

```bash
sudo chown root:"Domain Users" /sw/family
sudo chmod 775 /sw/family
```

**Install Apache:**

```bash
sudo apt install -y apache2
```

**Configure Apache alias:**

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

Add inside the `<VirtualHost *:80>` block:

```apache
Alias /family /sw/family
<Directory /sw/family>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

```bash
sudo systemctl reload apache2
```

**Add to smb.conf:**

```ini
[family]
    path = /sw/family
    browseable = yes
    read only = no
    guest ok = yes
    invalid users = DEATHSTAR\anakin
    create mask = 0664
    directory mask = 0775
```

Verify in browser: `http://192.168.1.1/family`

> 📸 `images/19_family_web.png` — browser showing /sw/family at http://192.168.1.1/family

---

### 7.4 Share `force` — anakin + yoda + jedis can access; leia cannot; public, web, read-only

**Exam:** `/sw/force` — `anakin`, `yoda` and group `jedis` can access; `leia` cannot; public, visible in browser, read-only.

```bash
sudo chown root:"Domain Users" /sw/force
sudo chmod 755 /sw/force
```

**Add Apache alias (same file as family):**

```apache
Alias /force /sw/force
<Directory /sw/force>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

```bash
sudo systemctl reload apache2
```

**Add to smb.conf:**

```ini
[force]
    path = /sw/force
    browseable = yes
    read only = yes
    guest ok = yes
    valid users = DEATHSTAR\anakin, DEATHSTAR\yoda, @DEATHSTAR\jedis
    invalid users = DEATHSTAR\leia
```

Verify in browser: `http://192.168.1.1/force`

> 📸 `images/20_force_web.png` — browser showing /sw/force at http://192.168.1.1/force

---

### 7.5 Check Config and Restart Samba

```bash
testparm
sudo systemctl restart samba-ad-dc

# Test per-user access
smbclient -L localhost -U leia%P@ssw0rd2024!
smbclient -L localhost -U anakin%P@ssw0rd2024!
```

> 📸 `images/21_shares_verify.png` — smbclient showing correct access per user

---

## 8. Linux Client — uc01

> All steps in this section are performed on the **client machine** `uc01`.

### 8.1 Configure Network

| Adapter | VirtualBox Type | Interface | IP |
|---------|-----------------|-----------|-----|
| Adapter 1 | NAT | `enp0s3` | `172.30.20.30/24` (static) |
| Adapter 2 | Internal Network (`intnet`) | `enp0s8` | `192.168.1.3/24` (static) |

> ⚠️ The internal network name `intnet` must be identical to the one used on srv01.

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 172.30.20.30/24
      routes:
        - to: default
          via: 172.30.20.1
      nameservers:
        addresses:
          - 8.8.8.8
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.1.3/24
      nameservers:
        addresses:
          - 192.168.1.1      # DC = srv01
          - 8.8.8.8
        search:
          - deathstar.local
```

```bash
sudo netplan apply
ping -c 4 192.168.1.1
```

### 8.2 Disable IPv6

```bash
sudo nano /etc/sysctl.conf
```

```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

```bash
sudo sysctl -p
```

> 📸 `images/22_client_ipv6_disabled.png` — ip a on client with no inet6

### 8.3 Install Packages

```bash
sudo apt remove -y chrony 2>/dev/null || true
sudo apt install -y realmd sssd sssd-tools krb5-user samba-common packagekit adcli ntp
```

Configure NTP to sync against the DC:

```bash
sudo nano /etc/ntp.conf
```

Add:

```
server 192.168.1.1 iburst
```

```bash
sudo systemctl restart ntp
```

### 8.4 Discover and Join the Domain

```bash
realm discover DEATHSTAR.LOCAL
```

> 📸 `images/23_realm_discover.png` — realm discover output

```bash
sudo realm join --user=Administrator DEATHSTAR.LOCAL
# Password: P@ssw0rd2024!
```

> 📸 `images/24_realm_join.png` — realm join output (no errors = success)

### 8.5 Verify Domain Join

```bash
realm list
id leia@DEATHSTAR.LOCAL
```

> 📸 `images/25_realm_list_id.png` — realm list and id showing the domain user

### 8.6 Optional — Allow Login Without Domain Suffix

```bash
sudo nano /etc/sssd/sssd.conf
```

Change:

```ini
use_fully_qualified_names = False
```

```bash
sudo systemctl restart sssd
```

---

## 9. Secondary Server — bespin01 (cloud01.city)

> All steps in this section are performed on the second server `bespin01`. This simulates an AWS cloud instance with its own independent domain.

### 9.1 Hostname and Hosts

```bash
sudo hostnamectl set-hostname bespin01.cloud01.city
sudo nano /etc/hosts
```

```
127.0.0.1       localhost
192.168.1.2     bespin01.cloud01.city bespin01
```

### 9.2 Network Configuration

| Adapter | VirtualBox Type | Interface | IP |
|---------|-----------------|-----------|-----|
| Adapter 1 | NAT | `enp0s3` | `172.30.20.20/24` (static) |
| Adapter 2 | Internal Network (`intnet`) | `enp0s8` | `192.168.1.2/24` (static) |

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 172.30.20.20/24
      routes:
        - to: default
          via: 172.30.20.1
      nameservers:
        addresses:
          - 8.8.8.8
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.1.2/24
      nameservers:
        addresses:
          - 127.0.0.1
          - 8.8.8.8
        search:
          - cloud01.city
```

```bash
sudo netplan apply
ping -c 4 192.168.1.1   # verify connection to srv01
```

### 9.3 Disable systemd-resolved and Configure DNS

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo unlink /etc/resolv.conf
sudo nano /etc/resolv.conf
```

```
nameserver 127.0.0.1
nameserver 8.8.8.8
search cloud01.city
```

```bash
sudo ss -tulnp | grep :53
# Must be empty
```

### 9.4 NTP

```bash
sudo apt remove -y chrony 2>/dev/null || true
sudo apt install -y ntp
```

Add to `/etc/ntp.conf`:

```
pool ntp.ubuntu.com iburst
```

```bash
sudo systemctl restart ntp && sudo systemctl enable ntp
```

### 9.5 Install and Provision Samba

```bash
sudo apt install -y samba krb5-user samba-dsdb-modules samba-vfs-modules
sudo systemctl stop smbd nmbd winbind 2>/dev/null || true
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig 2>/dev/null || true
sudo mv /etc/krb5.conf /etc/krb5.conf.bak 2>/dev/null || true

sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=CLOUD01.CITY \
  --domain=CLOUD01 \
  --adminpass='P@ssw0rd2024!' \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL

sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
```

> 📸 `images/26_bespin_provision.png` — samba-tool domain provision output for cloud01.city

### 9.6 Create Users and Directory with Your Initials

```bash
sudo samba-tool user add lando P@ssw0rd2024!
sudo samba-tool user add boba P@ssw0rd2024!

# Create trap folder and a subfolder with your initials (replace ABC)
sudo mkdir -p /city/trap/ABC
```

### 9.7 Configure Share `trap` — lando only, not boba

**Exam:** `/city/trap` — only `lando` can access, `boba` cannot. Create a subfolder with your initials inside.

```bash
sudo chown root:"Domain Users" /city/trap
sudo chmod 770 /city/trap
```

Add to `/etc/samba/smb.conf` on bespin01:

```ini
[trap]
    path = /city/trap
    browseable = no
    read only = no
    valid users = CLOUD01\lando
    invalid users = CLOUD01\boba
    create mask = 0660
    directory mask = 0770
```

```bash
sudo systemctl restart samba-ad-dc

# lando should succeed
smbclient //localhost/trap -U lando%P@ssw0rd2024!

# boba should be denied
smbclient //localhost/trap -U boba%P@ssw0rd2024!
```

> 📸 `images/27_trap_lando.png` — lando accessing trap successfully
> 📸 `images/28_trap_boba.png` — boba denied access to trap

---

## 10. SSH Access and Process Control

### 10.1 Verify SSH is Running on srv01

```bash
sudo systemctl status ssh
# If not installed:
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

### 10.2 Connect from the Physical Machine

From your own computer's terminal:

```bash
ssh your_user@172.30.20.10
```

> 📸 `images/29_ssh_connection.png` — SSH session established from the physical machine

### 10.3 Install and Launch `sl`

```bash
sudo apt install -y sl
sl
```

### 10.4 Pause and Resume with POSIX Signals

**Method 1 — Ctrl+Z / fg (same terminal):**

```bash
# While sl is running, press Ctrl+Z
jobs
# [1]+  Stopped    sl

fg %1    # resume
```

**Method 2 — kill signals (second terminal, preferred for demonstrating POSIX signals):**

```bash
# In a second terminal, find the PID
pgrep -x sl

# SIGSTOP (19) — pauses the process (kernel-level, cannot be ignored)
kill -19 $(pgrep -x sl)

# SIGCONT (18) — resumes the process
kill -18 $(pgrep -x sl)
```

> 📸 `images/30_sl_signals.png` — sl launched, paused and resumed with signals

---

## 11. Scheduled Task with Cron

### 11.1 Create the Script

```bash
sudo nano /usr/local/bin/r2d2.sh
```

```bash
#!/bin/bash
# r2d2.sh — Creates luke.sky file
# Scheduled: every day at 10:30

touch /home/administrator/luke.sky
echo "luke.sky created on $(date)" >> /var/log/r2d2.log
```

```bash
sudo chmod +x /usr/local/bin/r2d2.sh
```

> 📸 `images/31_r2d2_script.png` — cat of r2d2.sh

### 11.2 Add to crontab

```bash
crontab -e
```

Add at the end:

```
30 10 * * * /usr/local/bin/r2d2.sh
```

```bash
crontab -l
```

> 📸 `images/32_crontab.png` — crontab -l showing the 10:30 task

### 11.3 Test Manually

```bash
/usr/local/bin/r2d2.sh
ls -la /home/administrator/luke.sky
cat /var/log/r2d2.log
```

> 📸 `images/33_luke_sky.png` — luke.sky file created and r2d2.log content

---

## 12. Trust Relationship Between Domains

A bidirectional trust is established between `DEATHSTAR.LOCAL` (srv01) and `LAB01.LAN` (the lab practices server).

### 12.1 Set Up lab01

The `lab01` VM follows the same setup as `srv01` with different values:

| Adapter | Interface | IP |
|---------|-----------|-----|
| Adapter 1 (NAT) | `enp0s3` | `172.30.20.40/24` |
| Adapter 2 (intnet) | `enp0s8` | `192.168.1.4/24` |

```bash
sudo hostnamectl set-hostname lab01.lab01.lan
```

`/etc/hosts`:
```
127.0.0.1     localhost
192.168.1.4   lab01.lab01.lan lab01
```

Netplan `enp0s8`: IP `192.168.1.4/24`, DNS `127.0.0.1`, search `lab01.lan`

Provision:

```bash
sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=LAB01.LAN \
  --domain=LAB01 \
  --adminpass='P@ssw0rd2024!' \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL
```

### 12.2 Verify Connectivity Between DCs

From srv01:

```bash
ping -c 4 192.168.1.4
host -t SRV _kerberos._tcp.lab01.lan 192.168.1.4
```

From lab01:

```bash
ping -c 4 192.168.1.1
host -t SRV _kerberos._tcp.deathstar.local 192.168.1.1
```

### 12.3 Configure DNS Forwarding

**On srv01** — add to `[global]` in `/etc/samba/smb.conf`:

```ini
dns forwarder = 192.168.1.4
```

**On lab01** — add to `[global]` in `/etc/samba/smb.conf`:

```ini
dns forwarder = 192.168.1.1
```

Restart Samba on both:

```bash
sudo systemctl restart samba-ad-dc
```

Verify cross-resolution:

```bash
# From srv01
host -t SRV _ldap._tcp.lab01.lan

# From lab01
host -t SRV _ldap._tcp.deathstar.local
```

### 12.4 Create the Trust

Run on srv01:

```bash
sudo samba-tool domain trust create LAB01.LAN \
  --type=forest \
  --direction=both \
  -U administrator%P@ssw0rd2024! \
  --remote-admin="administrator%P@ssw0rd2024!"
```

> 📸 `images/34_trust_create.png` — samba-tool domain trust create output

### 12.5 Verify the Trust

```bash
sudo samba-tool domain trust list
sudo samba-tool domain trust validate LAB01.LAN -U administrator%P@ssw0rd2024!
```

> ⚠️ A netlogon warning during validation is normal with Samba 4. What matters is `TRUST[WERR_OK]`.

> 📸 `images/35_trust_verify.png` — trust list and validate with WERR_OK

### 12.6 Test Cross-Domain Authentication

```bash
kinit administrator@LAB01.LAN
klist
```

> 📸 `images/36_trust_kinit.png` — Kerberos ticket for LAB01.LAN user obtained from DEATHSTAR.LOCAL

---

## 13. Screenshots Index

> Screenshots are taken progressively during the lab and placed in the `images/` folder.

| No. | File | Section | What it shows |
|-----|------|---------|---------------|
| 01 | `01_lsblk_disks.png` | Disks | lsblk with sda at / and sdb at /home |
| 02 | `02_df_home.png` | Disks | df -h confirming /home on secondary disk |
| 03 | `03_hostname.png` | Network | hostnamectl and hostname -f |
| 04 | `04_hosts.png` | Network | /etc/hosts with internal IP |
| 05 | `05_netplan.png` | Network | Netplan config — enp0s3 172.30.20.10, enp0s8 192.168.1.1 |
| 06 | `06_ip_a.png` | Network | ip a showing both interfaces |
| 07 | `07_ping_internet.png` | Network | Successful ping to 8.8.8.8 |
| 08 | `08_ipv6_disabled.png` | Network | ip a with no inet6 addresses |
| 09 | `09_resolved_disabled.png` | Network | ss -tulnp with nothing on port 53 |
| 10 | `10_ntp_sync.png` | NTP | ntpq -p and timedatectl with active sync |
| 11 | `11_domain_provision.png` | Samba | Full samba-tool domain provision output |
| 12 | `12_samba_status.png` | Samba | systemctl status samba-ad-dc active |
| 13 | `13_ad_ports.png` | Samba | ss showing ports 53,88,389,445,636,3268 |
| 14 | `14_dns_check.png` | Kerberos | host commands resolving SRV and A records |
| 15 | `15_kinit_klist.png` | Kerberos | klist with administrator ticket |
| 16 | `16_smbclient.png` | Kerberos | smbclient listing shares |
| 17 | `17_users_groups.png` | Users | samba-tool user list and jedis members |
| 18 | `18_smb_blueprints.png` | Shares | blueprints section of smb.conf |
| 19 | `19_family_web.png` | Shares | Browser showing /sw/family |
| 20 | `20_force_web.png` | Shares | Browser showing /sw/force |
| 21 | `21_shares_verify.png` | Shares | smbclient access verification per user |
| 22 | `22_client_ipv6_disabled.png` | Client | ip a on uc01 with no inet6 |
| 23 | `23_realm_discover.png` | Client | realm discover DEATHSTAR.LOCAL |
| 24 | `24_realm_join.png` | Client | realm join with no errors |
| 25 | `25_realm_list_id.png` | Client | realm list and id leia@DEATHSTAR.LOCAL |
| 26 | `26_bespin_provision.png` | bespin01 | Provisioning cloud01.city |
| 27 | `27_trap_lando.png` | bespin01 | lando accessing trap share |
| 28 | `28_trap_boba.png` | bespin01 | boba denied access to trap |
| 29 | `29_ssh_connection.png` | SSH | SSH session from physical machine |
| 30 | `30_sl_signals.png` | SSH | sl paused and resumed with POSIX signals |
| 31 | `31_r2d2_script.png` | Cron | cat of r2d2.sh |
| 32 | `32_crontab.png` | Cron | crontab -l with task at 10:30 |
| 33 | `33_luke_sky.png` | Cron | luke.sky created and r2d2.log |
| 34 | `34_trust_create.png` | Trust | samba-tool domain trust create output |
| 35 | `35_trust_verify.png` | Trust | trust list and validate WERR_OK |
| 36 | `36_trust_kinit.png` | Trust | Cross-domain Kerberos ticket |
