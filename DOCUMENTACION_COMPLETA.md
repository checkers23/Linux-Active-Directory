# 📖 Complete Documentation — DeathStar S.A.

> Step-by-step guide for deploying Ubuntu Server as a Samba 4 Active Directory Domain Controller, joining a Linux client to the domain, configuring shared folders, setting up a secondary server, SSH process management, scheduled tasks and a domain trust relationship.

---

## Table of Contents

1. [Server Setup and Disks](#1-server-setup-and-disks)
2. [Network and Hostname Configuration](#2-network-and-hostname-configuration)
3. [NTP Time Synchronization](#3-ntp-time-synchronization)
4. [Samba AD DC Installation and Provisioning](#4-samba-ad-dc-installation-and-provisioning)
5. [Domain and Kerberos Verification](#5-domain-and-kerberos-verification)
6. [Joining the Linux Client to the Domain](#6-joining-the-linux-client-to-the-domain)
7. [Domain Users and Groups](#7-domain-users-and-groups)
8. [Shared Folders and Permissions](#8-shared-folders-and-permissions)
9. [Secondary Server — bespin01 (cloud01.city)](#9-secondary-server--bespin01-cloud01city)
10. [SSH Access and Process Management](#10-ssh-access-and-process-management)
11. [Scheduled Task with Cron](#11-scheduled-task-with-cron)
12. [Domain Trust Relationship](#12-domain-trust-relationship)

---

## 1. Server Setup and Disks

During the Ubuntu Server 22.04 installation, two virtual disks were added in VirtualBox and assigned at the partitioning step:

- **Disk 1** (`/dev/sda`): 20 GB — mounted as `/`
- **Disk 2** (`/dev/sdb`): 10 GB — mounted as `/home`

By doing this during the installer, Ubuntu automatically writes the correct entries to `/etc/fstab`, so no manual mounting is needed afterwards. To confirm everything is correctly set up, I ran the following commands:

```bash
lsblk
df -h
cat /etc/fstab
```

`lsblk` shows both block devices and their mount points, while `df -h` confirms that `/home` is being served from the secondary disk as expected.

> 📸 **Screenshot:** `images/01_lsblk_discos.png` — Output of lsblk showing /dev/sda at / and /dev/sdb at /home.
> 📸 **Screenshot:** `images/02_df_home.png` — Output of df -h confirming /home is mounted on the secondary disk.

---

## 2. Network and Hostname Configuration

### 2.1 Setting the Hostname

The first thing I configured was the hostname, since Samba and Kerberos both rely on being able to resolve the server's FQDN (Fully Qualified Domain Name) correctly. I set it to `srv01.deathstar.local` to match the domain we will be provisioning:

```bash
sudo hostnamectl set-hostname srv01.deathstar.local
hostname -f
# Expected output: srv01.deathstar.local
```

> 📸 **Screenshot:** `images/03_hostname.png` — Output of hostnamectl and hostname -f showing the correct name.

### 2.2 Editing `/etc/hosts`

Next I edited `/etc/hosts` to make sure the server can resolve its own FQDN to its internal IP address without depending on an external DNS. This is important because during the Samba provisioning process the tool queries the local hostname, and if it resolves incorrectly the provisioning will fail:

```bash
sudo nano /etc/hosts
```

```
127.0.0.1       localhost
192.168.10.1    srv01.deathstar.local srv01
```

I also removed the line that Ubuntu adds by default associating `127.0.1.1` with the old hostname, since having two entries for the same hostname causes confusion.

```bash
cat /etc/hosts
```

> 📸 **Screenshot:** `images/04_hosts.png` — Contents of /etc/hosts with the server's internal IP.

### 2.3 Configuring Network Interfaces with Netplan

The server has two network adapters configured in VirtualBox:

- **Adapter 1 (enp0s3)** — NAT: used to reach the internet and install packages. VirtualBox assigns the IP automatically via DHCP.
- **Adapter 2 (enp0s8)** — Internal Network (`intnet`): used for all domain traffic (AD, Kerberos, LDAP, DNS). Static IP `192.168.10.1/24`.

> In VirtualBox: VM Settings → Network → Adapter 1: NAT / Adapter 2: Internal Network (name: `intnet`)

I edited the Netplan configuration file to reflect this setup:

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
        - 192.168.10.1/24
      nameservers:
        addresses:
          - 127.0.0.1
          - 8.8.8.8
        search:
          - deathstar.local
```

The DNS for `enp0s8` is pointed to `127.0.0.1` because once Samba is provisioned, this server will be its own DNS resolver for the `deathstar.local` domain. I then applied the configuration and verified connectivity:

```bash
sudo netplan apply
ip a
ip route

# Test internet access through NAT
ping -c 4 8.8.8.8

# Test internal network
ping -c 4 192.168.10.1
```

> 📸 **Screenshot:** `images/05_netplan_yaml.png` — Contents of the netplan file with both interfaces.
> 📸 **Screenshot:** `images/06_ip_a.png` — Output of ip a showing enp0s3 (NAT) and enp0s8 with 192.168.10.1.
> 📸 **Screenshot:** `images/07_ping_internet.png` — Successful ping to 8.8.8.8 confirming internet access.

### 2.4 Disabling IPv6

IPv6 can cause problems with Samba and Kerberos because the system may try to resolve hostnames over IPv6 before IPv4, leading to unexpected failures. Since this is a purely IPv4 lab environment, I disabled it by adding the following lines to `/etc/sysctl.conf`:

```bash
sudo nano /etc/sysctl.conf
```

```
# Disable IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

I applied the changes immediately without rebooting using:

```bash
sudo sysctl -p
```

To confirm it worked, I checked that no `inet6` addresses appear in the network interfaces:

```bash
ip a | grep inet6
```

No output means IPv6 is correctly disabled.

> 📸 **Screenshot:** `images/08_ipv6_desactivado.png` — Output of sysctl -p and ip a with no inet6 addresses.

### 2.5 Disabling systemd-resolved

⚠️ **This step is critical.** Samba needs port 53 to run its own internal DNS server. Ubuntu 22.04 runs `systemd-resolved` by default, which binds to port 53 and would block Samba from using it. I stopped and disabled it, then created a static `resolv.conf` pointing to localhost (which will be handled by Samba after provisioning):

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

# Remove the symlink that systemd-resolved created
sudo unlink /etc/resolv.conf

# Create a static resolv.conf
sudo nano /etc/resolv.conf
```

```
nameserver 127.0.0.1
nameserver 8.8.8.8
search deathstar.local
```

I then verified that port 53 is now free:

```bash
sudo ss -tulnp | grep :53
```

If this returns no output, the port is free and Samba will be able to bind to it.

> 📸 **Screenshot:** `images/09_resolved_disabled.png` — Output of ss -tulnp with nothing listening on port 53.

---

## 3. NTP Time Synchronization

Kerberos authentication — which is the backbone of Active Directory — is extremely sensitive to time differences. If the clock difference between a client and the server exceeds 5 minutes, Kerberos will refuse to issue tickets and authentication will fail completely. This is why time synchronization is a mandatory step before provisioning the domain.

The professor specifically asked us to use classic `ntpd` instead of `chrony`, since ntpd integrates better with Samba's NTP signing infrastructure.

### 3.1 Removing chrony

Ubuntu 22.04 installs `chrony` by default, so I removed it first to avoid conflicts:

```bash
sudo systemctl stop chronyd
sudo systemctl disable chronyd
sudo apt remove -y chrony
```

### 3.2 Installing and Configuring ntpd

I installed the classic NTP daemon and configured it to sync from public NTP servers, while also allowing clients on the internal network to synchronize against this server (since the DC should act as the NTP source for all domain members):

```bash
sudo apt install -y ntp
sudo nano /etc/ntp.conf
```

```
# Public NTP servers
pool ntp.ubuntu.com iburst
pool pool.ntp.org iburst

# Allow clients on the internal network to sync from this server
restrict default nomodify nopeer noquery limited kod
restrict 127.0.0.1
restrict ::1
restrict 192.168.10.0 mask 255.255.255.0 nomodify nopeer
```

```bash
sudo systemctl restart ntp
sudo systemctl enable ntp
```

### 3.3 Configuring AppArmor for Samba NTP Signing

Samba automatically creates `/var/lib/samba/ntp_signd/` to sign NTP packets, which is needed so that Windows clients can authenticate their NTP requests when joined to the domain. For this to work, I had to grant ntpd permission to access that directory through AppArmor:

```bash
sudo nano /etc/apparmor.d/usr.sbin.ntpd
```

Inside the existing block, I added:

```
/var/lib/samba/ntp_signd r,
/var/lib/samba/ntp_signd/** rw,
```

```bash
sudo systemctl restart apparmor
sudo systemctl restart ntp
```

### 3.4 Verifying Synchronization

I confirmed that ntpd is syncing correctly using `ntpq -p`, where the `*` symbol marks the active sync source:

```bash
ntpq -p
timedatectl
```

`timedatectl` should show `System clock synchronized: yes` and `NTP service: active`.

> 📸 **Screenshot:** `images/10_ntp_sync.png` — Output of ntpq -p and timedatectl showing active synchronization.

---

## 4. Samba AD DC Installation and Provisioning

### 4.1 Installing Required Packages

I installed Samba along with the Kerberos client and the necessary Samba modules for AD DC functionality:

```bash
sudo apt install -y samba krb5-user samba-dsdb-modules samba-vfs-modules winbind
```

When the installer asks for the default Kerberos realm, I entered `DEATHSTAR.LOCAL` in uppercase, as Kerberos realms are case-sensitive.

### 4.2 Stopping Existing Services and Cleaning Up

Before provisioning, I stopped any running Samba services that might conflict and backed up the default configuration files, since the provisioning process will replace them:

```bash
sudo systemctl stop smbd nmbd winbind 2>/dev/null || true
sudo systemctl disable smbd nmbd winbind 2>/dev/null || true

sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig 2>/dev/null || true
sudo mv /etc/krb5.conf /etc/krb5.conf.bak 2>/dev/null || true
```

### 4.3 Provisioning the Domain

This is the central step. `samba-tool domain provision` creates the entire AD database, the Kerberos KDC, the internal DNS zone and a working `smb.conf` tailored for an AD DC:

```bash
sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=DEATHSTAR.LOCAL \
  --domain=DEATHSTAR \
  --adminpass='P@ssw0rd2024!' \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL
```

The `--use-rfc2307` flag enables POSIX/UNIX attributes in the LDAP schema, which allows Linux systems to resolve domain users as local POSIX users. The `--dns-backend=SAMBA_INTERNAL` option means Samba will manage DNS itself, which is why we freed up port 53 earlier.

> 📸 **Screenshot:** `images/11_domain_provision.png` — Full output of the samba-tool domain provision command.

### 4.4 Copying the Kerberos Configuration

The provisioning tool generates a ready-made `krb5.conf` inside `/var/lib/samba/private/`. I copied it to the system location so that Kerberos tools like `kinit` work correctly:

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

### 4.5 Starting the Service

I unmasked the `samba-ad-dc` service (it is masked by default when the individual smbd/nmbd packages are installed), then enabled and started it:

```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
sudo systemctl status samba-ad-dc
```

> 📸 **Screenshot:** `images/12_samba_status.png` — Output of systemctl status samba-ad-dc showing active (running).

### 4.6 Verifying All AD Ports Are Listening

To confirm the DC is fully operational I checked that all the ports required by Active Directory are open and listening:

```bash
sudo ss -tulnp | grep -E ':(53|88|389|445|636|3268)'
```

All six ports must appear: 53 (DNS), 88 (Kerberos), 389 (LDAP), 445 (SMB), 636 (LDAPS) and 3268 (Global Catalog).

> 📸 **Screenshot:** `images/13_puertos_ad.png` — Output of ss showing all AD ports active.

---

## 5. Domain and Kerberos Verification

### 5.1 Checking the Domain Level

```bash
sudo samba-tool domain level show
```

### 5.2 Verifying Internal DNS

Samba's internal DNS should now be responding to SRV record queries, which are what clients use to discover the DC. I queried the local DNS to confirm:

```bash
host -t SRV _kerberos._tcp.deathstar.local 127.0.0.1
host -t SRV _ldap._tcp.deathstar.local 127.0.0.1
host -t A srv01.deathstar.local 127.0.0.1
```

All three should return valid results pointing to `192.168.10.1`.

> 📸 **Screenshot:** `images/14_dns_check.png` — Output of host commands showing correct SRV and A record resolution.

### 5.3 Obtaining a Kerberos Ticket

The definitive test of a working AD DC is getting a Kerberos ticket. I used `kinit` to request a ticket for the Administrator account, then listed it with `klist` and cleaned it up with `kdestroy`:

```bash
kinit administrator@DEATHSTAR.LOCAL
# Enter password: P@ssw0rd2024!

klist
kdestroy
```

A successful `klist` output shows the ticket was issued and has a valid expiry time.

> 📸 **Screenshot:** `images/15_kinit_klist.png` — Output of klist showing the Administrator Kerberos ticket.

### 5.4 Testing SMB Access

Finally I used `smbclient` to list the shares available on the server, which also confirms that SMB and authentication are working end-to-end:

```bash
smbclient -L localhost -U administrator%P@ssw0rd2024!
```

> 📸 **Screenshot:** `images/16_smbclient.png` — smbclient listing the server shares.

---

## 6. Joining the Linux Client to the Domain

> These steps are performed on the client machine `cli01` (Ubuntu Desktop 22.04).

### 6.1 Network Configuration

The client also has two network adapters in VirtualBox, configured exactly like the server: NAT on `enp0s3` for internet access, and internal network `intnet` on `enp0s8` for domain communication. It is essential that both the server and the client use the same `intnet` name, otherwise they will be on separate isolated networks and cannot communicate.

I configured `enp0s8` with a static IP and pointed its DNS to the DC at `192.168.10.1`, since the client needs to be able to resolve the domain's SRV records to find the DC:

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
        - 192.168.10.10/24
      nameservers:
        addresses:
          - 192.168.10.1
          - 8.8.8.8
        search:
          - deathstar.local
```

```bash
sudo netplan apply

# Verify connectivity with the DC
ping -c 4 192.168.10.1
```

### 6.2 Disabling IPv6 on the Client

Same as on the server, I disabled IPv6 to avoid resolution issues:

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
ip a | grep inet6
```

> 📸 **Screenshot:** `images/17_cliente_ipv6_desactivado.png` — Output of ip a on the client with no inet6 addresses.

### 6.3 Verifying DNS Resolution from the Client

Before attempting to join the domain I verified that the client can correctly resolve the domain's DNS records using the DC as resolver:

```bash
host -t A srv01.deathstar.local
host -t SRV _kerberos._tcp.deathstar.local
```

Both should return valid results. If they don't, the domain join will fail.

### 6.4 Installing Required Packages

I installed `realmd` and `sssd`, which together handle the domain discovery and join process, as well as the Kerberos client tools:

```bash
sudo apt install -y realmd sssd sssd-tools krb5-user samba-common packagekit adcli
```

When prompted for the Kerberos realm, I entered `DEATHSTAR.LOCAL`.

### 6.5 Configuring NTP on the Client

The client must also be time-synchronized. I removed `chrony` and configured `ntpd` to sync directly against the DC instead of a public server, which ensures the client and server always agree on time:

```bash
sudo systemctl stop chronyd
sudo systemctl disable chronyd
sudo apt remove -y chrony
sudo apt install -y ntp

sudo nano /etc/ntp.conf
```

```
server 192.168.10.1 iburst
```

```bash
sudo systemctl restart ntp
ntpq -p
```

### 6.6 Discovering and Joining the Domain

With DNS, NTP and packages all in order, I used `realm discover` to confirm the client can see the domain, then joined it with the Administrator account:

```bash
realm discover DEATHSTAR.LOCAL
```

> 📸 **Screenshot:** `images/18_realm_discover.png` — Output of realm discover showing DEATHSTAR.LOCAL information.

```bash
sudo realm join --user=Administrator DEATHSTAR.LOCAL
# Enter password: P@ssw0rd2024!
```

A successful join produces no error output.

> 📸 **Screenshot:** `images/19_realm_join.png` — Output of realm join with no errors.

### 6.7 Verifying the Domain Join

I confirmed the join was successful by listing the domain with `realm list` and checking that a domain user resolves correctly with `id`:

```bash
realm list
id leia@DEATHSTAR.LOCAL
```

> 📸 **Screenshot:** `images/20_realm_list_id.png` — Output of realm list and id leia@DEATHSTAR.LOCAL.

### 6.8 Allowing Login Without the Domain Suffix (Optional)

By default, sssd requires users to log in as `leia@DEATHSTAR.LOCAL`. To allow simply `leia`, I edited the sssd configuration:

```bash
sudo nano /etc/sssd/sssd.conf
```

```ini
use_fully_qualified_names = False
```

```bash
sudo systemctl restart sssd
```

---

## 7. Domain Users and Groups

> These steps are run on the server `srv01`.

I created the three users required by the project using `samba-tool`, then created the `jedis` group and added the appropriate members. These users will later be used to test access permissions on the shared folders:

```bash
sudo samba-tool user add leia P@ssw0rd2024!
sudo samba-tool user add anakin P@ssw0rd2024!
sudo samba-tool user add yoda P@ssw0rd2024!

sudo samba-tool group add jedis

sudo samba-tool group addmembers jedis anakin
sudo samba-tool group addmembers jedis yoda
```

I then verified everything was created correctly:

```bash
sudo samba-tool user list
sudo samba-tool group listmembers jedis
```

> 📸 **Screenshot:** `images/21_usuarios_grupos.png` — Output of samba-tool user list and group listmembers jedis.

---

## 8. Shared Folders and Permissions

I created the directory structure first, then configured each share in `smb.conf` with the appropriate access rules.

```bash
sudo mkdir -p /deathstar/blueprints
sudo mkdir -p /sw/family
sudo mkdir -p /sw/force
```

### 8.1 Share `blueprints` — Only leia can access, not public

This share contains sensitive information and should only be accessible to `leia`. I set the filesystem permissions and then defined the share in `smb.conf` with `valid users` so that only `leia` can authenticate to it. Setting `browseable = no` hides it from the network share list:

```bash
sudo chown root:users /deathstar/blueprints
sudo chmod 770 /deathstar/blueprints

sudo nano /etc/samba/smb.conf
```

```ini
[blueprints]
   path = /deathstar/blueprints
   browseable = no
   read only = no
   valid users = leia@DEATHSTAR.LOCAL, DEATHSTAR\leia
   create mask = 0660
   directory mask = 0770
```

> 📸 **Screenshot:** `images/22_smb_blueprints.png` — blueprints section in smb.conf.

### 8.2 Share `family` — anakin cannot access, public and visible in browser

The `family` share is public (guest access allowed) but explicitly blocks `anakin` using `invalid users`. I also installed Apache and configured an alias so the folder's contents are browsable from a web browser:

```bash
sudo chown root:users /sw/family
sudo chmod 775 /sw/family
sudo apt install -y apache2
```

Added to `smb.conf`:

```ini
[family]
   path = /sw/family
   browseable = yes
   read only = no
   guest ok = yes
   invalid users = anakin@DEATHSTAR.LOCAL, DEATHSTAR\anakin
   create mask = 0664
   directory mask = 0775
```

Added to `/etc/apache2/sites-available/000-default.conf` inside the VirtualHost block:

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

> 📸 **Screenshot:** `images/23_family_web.png` — Browser showing /sw/family contents at http://192.168.10.1/family.

### 8.3 Share `force` — anakin, yoda and jedis group can access, leia cannot, public, web, read-only

The `force` share is read-only and accessible to `anakin`, `yoda` and all members of the `jedis` group, but explicitly denies `leia`. I set it as public so it can be browsed without authentication, and also made it accessible via Apache:

```bash
sudo chown root:users /sw/force
sudo chmod 755 /sw/force
```

Added to `smb.conf`:

```ini
[force]
   path = /sw/force
   browseable = yes
   read only = yes
   guest ok = yes
   valid users = anakin@DEATHSTAR.LOCAL, yoda@DEATHSTAR.LOCAL, @jedis@DEATHSTAR.LOCAL
   invalid users = leia@DEATHSTAR.LOCAL
```

Added to the Apache config:

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

> 📸 **Screenshot:** `images/24_force_web.png` — Browser showing /sw/force contents at http://192.168.10.1/force.

### 8.4 Validating and Restarting Samba

Before restarting I always run `testparm` to catch any syntax errors in `smb.conf`, then restarted and tested access with different users to confirm the permissions are applied correctly:

```bash
testparm

sudo systemctl restart samba-ad-dc

smbclient -L localhost -U leia%P@ssw0rd2024!
smbclient -L localhost -U anakin%P@ssw0rd2024!
```

> 📸 **Screenshot:** `images/25_shares_verificacion.png` — smbclient output confirming which shares each user can see.

---

## 9. Secondary Server — bespin01 (cloud01.city)

> These steps are performed on the second server VM (`bespin01`), which will host a completely independent domain called `cloud01.city`.

### 9.1 Hostname and Hosts

Following the same process as on srv01, I set the hostname and updated `/etc/hosts`:

```bash
sudo hostnamectl set-hostname bespin01.cloud01.city

sudo nano /etc/hosts
```

```
127.0.0.1       localhost
192.168.10.2    bespin01.cloud01.city bespin01
```

### 9.2 Network Configuration

bespin01 has the same two-adapter setup as srv01: NAT on `enp0s3` and internal network `intnet` on `enp0s8`. The same `intnet` name ensures all VMs can communicate with each other:

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
        - 192.168.10.2/24
      nameservers:
        addresses:
          - 127.0.0.1
          - 8.8.8.8
        search:
          - cloud01.city
```

```bash
sudo netplan apply
ping -c 4 192.168.10.1
```

### 9.3 Disabling systemd-resolved and IPv6

Same as on srv01, I disabled both to avoid DNS port conflicts and IPv6 resolution issues:

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
# Disable IPv6
sudo nano /etc/sysctl.conf
# Add the three net.ipv6 lines
sudo sysctl -p

# Verify port 53 is free
sudo ss -tulnp | grep :53
```

### 9.4 NTP Configuration

I removed chrony and configured ntpd to sync from public NTP servers:

```bash
sudo systemctl stop chronyd && sudo apt remove -y chrony
sudo apt install -y ntp
```

Added to `/etc/ntp.conf`:
```
pool ntp.ubuntu.com iburst
pool pool.ntp.org iburst
```

```bash
sudo systemctl restart ntp && sudo systemctl enable ntp
ntpq -p
```

### 9.5 Provisioning the Domain

I provisioned bespin01 as a DC for the `cloud01.city` domain using the same process as srv01 but with different realm and domain names:

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

> 📸 **Screenshot:** `images/26_bespin_provision.png` — Full output of the provisioning of cloud01.city on bespin01.

### 9.6 Creating Users and the `trap` Share

I created the two users required for this domain and then created the `/city/trap` directory, including a subdirectory with my initials as specified in the task:

```bash
sudo samba-tool user add lando P@ssw0rd2024!
sudo samba-tool user add boba P@ssw0rd2024!

sudo mkdir -p /city/trap/abc
sudo chown root:users /city/trap
sudo chmod 770 /city/trap
```

I then configured the share so that only `lando` can access it and `boba` is explicitly denied:

```bash
sudo nano /etc/samba/smb.conf
```

```ini
[trap]
   path = /city/trap
   browseable = no
   read only = no
   valid users = lando@CLOUD01.CITY, CLOUD01\lando
   invalid users = boba@CLOUD01.CITY, CLOUD01\boba
   create mask = 0660
   directory mask = 0770
```

```bash
testparm
sudo systemctl restart samba-ad-dc

# Test: lando should succeed, boba should be denied
smbclient //localhost/trap -U lando%P@ssw0rd2024!
smbclient //localhost/trap -U boba%P@ssw0rd2024!
```

> 📸 **Screenshot:** `images/27_trap_lando.png` — Successful access by lando to the trap share.
> 📸 **Screenshot:** `images/28_trap_boba.png` — Access denied for boba on the trap share.

---

## 10. SSH Access and Process Management

### 10.1 Enabling SSH

I verified that the SSH server was active on srv01. If it hadn't been installed, the command to install it would be:

```bash
sudo systemctl status ssh

# If not installed:
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

### 10.2 Connecting from the Host Machine

From a terminal on my physical machine I connected to the server over the internal network:

```bash
ssh usuario@192.168.10.1
```

> 📸 **Screenshot:** `images/29_ssh_conexion.png` — Terminal showing the established SSH session.

### 10.3 Installing and Running `sl`

`sl` is a fun terminal program that displays an ASCII steam train. I installed it and launched it in the foreground to then demonstrate process control:

```bash
sudo apt install -y sl
sl
```

### 10.4 Pausing and Resuming the Process

I demonstrated two methods to stop and resume the process:

**Method 1 — Ctrl+Z and bg/fg (same terminal):**

With `sl` running, pressing `Ctrl+Z` sends a SIGTSTP signal which pauses the process and returns control to the shell:

```bash
jobs
# [1]+  Stopped    sl

bg %1    # resume in the background
fg %1    # bring back to foreground
```

**Method 2 — POSIX signals with kill (second terminal):**

This method demonstrates low-level process control. `kill -19` sends SIGSTOP, which is handled directly by the kernel and cannot be caught or ignored by the process. `kill -18` sends SIGCONT to resume it:

```bash
pgrep -x sl

kill -19 $(pgrep -x sl)   # freeze the process
kill -18 $(pgrep -x sl)   # resume the process
```

The key difference is that `Ctrl+Z` sends SIGTSTP, which a process could theoretically intercept and ignore, whereas SIGSTOP is unconditional.

> 📸 **Screenshot:** `images/30_sl_signals.png` — Terminal showing sl paused and resumed using both methods.

---

## 11. Scheduled Task with Cron

The task requires a script called `r2d2.sh` that creates a file called `luke.sky`, scheduled to run automatically at 10:30 every day.

### 11.1 Creating the Script

I created the script in `/usr/local/bin/` so it is available system-wide and gave it execute permissions:

```bash
sudo nano /usr/local/bin/r2d2.sh
```

```bash
#!/bin/bash
# r2d2.sh - Creates the luke.sky file

touch /home/administrador/luke.sky
echo "File luke.sky created on $(date)" >> /var/log/r2d2.log
```

```bash
sudo chmod +x /usr/local/bin/r2d2.sh
```

> 📸 **Screenshot:** `images/31_r2d2_script.png` — Contents of r2d2.sh shown with cat.

### 11.2 Scheduling with Cron

I opened the user's crontab and added the task. The cron format is `minute hour day month weekday command`:

```bash
crontab -e
```

```
30 10 * * * /usr/local/bin/r2d2.sh
```

```bash
crontab -l
```

> 📸 **Screenshot:** `images/32_crontab.png` — Output of crontab -l showing the task scheduled at 10:30.

### 11.3 Manual Test

I ran the script manually to verify it works correctly before waiting for the scheduled time:

```bash
/usr/local/bin/r2d2.sh
ls -la /home/administrador/luke.sky
cat /var/log/r2d2.log
```

> 📸 **Screenshot:** `images/33_luke_sky.png` — ls showing luke.sky was created and the r2d2 log file.

---

## 12. Domain Trust Relationship

A trust relationship allows users from one domain to authenticate to resources in another domain. I established a bidirectional trust between `DEATHSTAR.LOCAL` and `LAB01.LAN`, which is simulated using a fourth VM configured as a DC for that domain.

### 12.1 Setting Up the Fourth VM — DC for lab01.lan

I provisioned a fourth VM following exactly the same steps as srv01 (hostname, hosts, network, disable IPv6, disable systemd-resolved, NTP, Samba provisioning) but with the following different values:

```bash
sudo hostnamectl set-hostname lab01.lab01.lan
```

`/etc/hosts`:
```
127.0.0.1     localhost
192.168.10.3  lab01.lab01.lan lab01
```

`enp0s8` IP: `192.168.10.3/24`, DNS: `127.0.0.1`, search: `lab01.lan`

```bash
sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=LAB01.LAN \
  --domain=LAB01 \
  --adminpass='P@ssw0rd2024!' \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL
```

### 12.2 Verifying Connectivity Between the Two DCs

Before creating the trust, I confirmed that both DCs can reach each other over the internal network and resolve each other's Kerberos SRV records:

```bash
# From srv01
ping -c 4 192.168.10.3
host -t SRV _kerberos._tcp.lab01.lan 192.168.10.3

# From lab01
ping -c 4 192.168.10.1
host -t SRV _kerberos._tcp.deathstar.local 192.168.10.1
```

### 12.3 Configuring DNS Forwarding Between Domains

For the trust to work, each DC needs to be able to resolve the other domain's DNS records. I configured a DNS forwarder in each server's `smb.conf` so that queries for the other domain are forwarded to the correct DC:

On srv01 — add to the `[global]` section of `/etc/samba/smb.conf`:
```ini
dns forwarder = 192.168.10.3
```

On lab01 — add to the `[global]` section of `/etc/samba/smb.conf`:
```ini
dns forwarder = 192.168.10.1
```

```bash
# Restart Samba on both servers
sudo systemctl restart samba-ad-dc

# Verify cross-domain DNS resolution
# From srv01:
host -t SRV _ldap._tcp.lab01.lan
# From lab01:
host -t SRV _ldap._tcp.deathstar.local
```

### 12.4 Creating the Trust

With connectivity and DNS confirmed, I created the bidirectional trust from srv01:

```bash
sudo samba-tool domain trust create LAB01.LAN \
  --type=forest \
  --direction=both \
  -U administrator%P@ssw0rd2024! \
  --remote-admin="administrator%P@ssw0rd2024!"
```

> 📸 **Screenshot:** `images/34_trust_create.png` — Output of samba-tool domain trust create showing the trust was created.

### 12.5 Verifying the Trust

I verified the trust is listed and validated correctly:

```bash
sudo samba-tool domain trust list
sudo samba-tool domain trust validate LAB01.LAN -U administrator%P@ssw0rd2024!
```

> ⚠️ A netlogon warning during validation is normal in Samba 4. The important thing is to see `TRUST[WERR_OK]` in the output.

> 📸 **Screenshot:** `images/35_trust_verify.png` — Output of trust list and validate showing WERR_OK.

### 12.6 Testing Cross-Domain Authentication

As a final test I obtained a Kerberos ticket for a user from `LAB01.LAN` while logged into `srv01`, confirming that authentication crosses the trust correctly:

```bash
kinit administrator@LAB01.LAN
klist
```

> 📸 **Screenshot:** `images/36_trust_kinit.png` — Kerberos ticket for a LAB01.LAN user obtained from DEATHSTAR.LOCAL.

---

## Screenshot Reference

> Screenshots are taken during the lab and uploaded to the `images/` folder of the repository.

| Nº  | File                                 | Section        | What it shows                                                 |
|-----|--------------------------------------|----------------|---------------------------------------------------------------|
| 01  | `01_lsblk_discos.png`               | Disks          | lsblk with /dev/sda at / and /dev/sdb at /home               |
| 02  | `02_df_home.png`                    | Disks          | df -h confirming /home on secondary disk                      |
| 03  | `03_hostname.png`                   | Network        | hostnamectl and hostname -f showing correct FQDN              |
| 04  | `04_hosts.png`                      | Network        | Contents of /etc/hosts                                        |
| 05  | `05_netplan_yaml.png`               | Network        | Netplan file with both interfaces                             |
| 06  | `06_ip_a.png`                       | Network        | ip a showing enp0s3 (NAT) and enp0s8 (192.168.10.1)          |
| 07  | `07_ping_internet.png`              | Network        | Successful ping to 8.8.8.8                                    |
| 08  | `08_ipv6_desactivado.png`           | Network        | sysctl -p and ip a with no inet6 addresses                    |
| 09  | `09_resolved_disabled.png`          | Network        | ss -tulnp with nothing on port 53                             |
| 10  | `10_ntp_sync.png`                   | NTP            | ntpq -p and timedatectl with active sync                      |
| 11  | `11_domain_provision.png`           | Samba          | Full output of samba-tool domain provision                    |
| 12  | `12_samba_status.png`               | Samba          | systemctl status samba-ad-dc active (running)                 |
| 13  | `13_puertos_ad.png`                 | Samba          | ss -tulnp with ports 53, 88, 389, 445, 636, 3268              |
| 14  | `14_dns_check.png`                  | Kerberos/DNS   | host commands resolving SRV and A records                     |
| 15  | `15_kinit_klist.png`                | Kerberos/DNS   | klist with Administrator ticket and kdestroy                  |
| 16  | `16_smbclient.png`                  | Kerberos/DNS   | smbclient listing server shares                               |
| 17  | `17_cliente_ipv6_desactivado.png`   | Client         | ip a on client with no inet6 addresses                        |
| 18  | `18_realm_discover.png`             | Client         | realm discover DEATHSTAR.LOCAL output                         |
| 19  | `19_realm_join.png`                 | Client         | realm join completed without errors                           |
| 20  | `20_realm_list_id.png`              | Client         | realm list and id leia@DEATHSTAR.LOCAL                        |
| 21  | `21_usuarios_grupos.png`            | Users          | samba-tool user list and group listmembers jedis              |
| 22  | `22_smb_blueprints.png`             | Shares         | blueprints section in smb.conf                                |
| 23  | `23_family_web.png`                 | Shares         | Browser showing /sw/family at http://192.168.10.1/family      |
| 24  | `24_force_web.png`                  | Shares         | Browser showing /sw/force at http://192.168.10.1/force        |
| 25  | `25_shares_verificacion.png`        | Shares         | smbclient confirming per-user share access                    |
| 26  | `26_bespin_provision.png`           | Bespin         | Provisioning output for cloud01.city on bespin01              |
| 27  | `27_trap_lando.png`                 | Bespin         | lando accessing the trap share successfully                   |
| 28  | `28_trap_boba.png`                  | Bespin         | boba denied access to the trap share                          |
| 29  | `29_ssh_conexion.png`               | SSH            | SSH session established from host machine                     |
| 30  | `30_sl_signals.png`                 | SSH            | sl paused and resumed with Ctrl+Z/bg/fg or kill -19/-18       |
| 31  | `31_r2d2_script.png`                | Cron           | Contents of r2d2.sh shown with cat                            |
| 32  | `32_crontab.png`                    | Cron           | crontab -l showing task at 10:30                              |
| 33  | `33_luke_sky.png`                   | Cron           | luke.sky file created and r2d2 log                            |
| 34  | `34_trust_create.png`               | Trust          | Output of samba-tool domain trust create                      |
| 35  | `35_trust_verify.png`               | Trust          | trust list and validate showing WERR_OK                       |
| 36  | `36_trust_kinit.png`                | Trust          | Cross-domain Kerberos ticket from LAB01.LAN                   |
