# 🌌 DeathStar S.A. — Linux Active Directory

> Full implementation of a Samba 4 Active Directory infrastructure on Ubuntu Server 22.04. This project covers domain controller setup, Linux client integration, shared folder permissions with ACLs, a secondary domain on AWS EC2, process control via POSIX signals, scheduled tasks with cron, and a bidirectional domain trust relationship.

📖 **[Read the full documentation](DOCUMENTATION.md)**

---

## 🖥️ Infrastructure

| Server | Hostname | IP | Domain | Role |
|--------|----------|----|--------|------|
| Main DC | srv01.deathstar.local | 192.168.1.1 | deathstar.local | Samba 4 AD Domain Controller |
| Linux Client | uc01 | 192.168.1.3 | deathstar.local | Domain Member (Ubuntu Desktop) |
| Trust DC | ls07.lab07.lan | 192.168.1.2 | lab07.lan | Samba 4 AD Domain Controller |
| AWS DC | bespin07.cloud07.city | 10.0.1.10 | cloud07.city | Domain Controller (EC2 instance) |

---

## ✅ What Was Implemented

**Domain Controller (srv01)**
- Ubuntu Server 22.04 with two disks — `/` on a 20GB disk, `/home` on a separate 10GB disk
- Samba 4 AD DC provisioned with internal DNS and Kerberos
- Domain users `leia`, `anakin`, `yoda` and security group `jedis`

**Shared Folders with ACL Permissions**
- `blueprints` at `/deathstar/blueprints` — private, accessible only by `leia`
- `family` at `/sw/family` — public, `anakin` explicitly denied
- `force` at `/sw/force` — public, read-only, `leia` denied, `anakin`/`yoda`/`jedis` allowed

**Linux Client (uc01)**
- Ubuntu 24.04 Desktop joined to `deathstar.local` using `realmd` and `sssd`
- Domain users authenticated via Kerberos and verified with graphical login

**AWS Server (bespin07)**
- EC2 instance on AWS Academy with custom VPC, security group and Elastic IP
- Samba 4 AD DC provisioned for domain `cloud07.city`
- Share `trap` at `/city/trap` — `lando` allowed, `boba` denied, `MPI` subfolder created

**SSH and Process Control**
- SSH access from physical machine to srv01 via VirtualBox port forwarding
- `sl` process paused with `SIGSTOP` (kill -19) and resumed with `SIGCONT` (kill -18)

**Scheduled Task**
- Script `r2d2.sh` creates `luke.sky` file and logs execution timestamp
- Scheduled daily at 10:30 via crontab

**Domain Trust**
- Bidirectional external trust between `deathstar.local` and `lab07.lan`
- DNS forwarding configured on both DCs
- Cross-domain Kerberos authentication verified with `kinit Administrator@LAB07.LAN`

---

## ⚠️ Key Lessons Learned

| Problem Encountered | Solution |
|---------------------|----------|
| `samba-ad-dc.service does not exist` | Install `samba-ad-dc` package **before** provisioning, never after |
| `KDC reply did not match expectations` | Remove `127.0.1.1` from `/etc/hosts` and fix duplicate DNS records |
| `Client not found in Kerberos database` | Always use UPPERCASE realm — `Administrator@DEATHSTAR.LOCAL` not `deathstar.local` |
| Duplicate NAT IP in DNS (`10.0.2.x`) | `sudo samba-tool dns delete 127.0.0.1 domain hostname A 10.0.2.x` |
| SSH PEM bad permissions on Windows | `icacls key.pem /inheritance:r` then remove `BUILTIN\Usuarios` and `NT AUTHORITY\Usuarios autentificados` |

---

## 📁 Repository Structure

```
├── DOCUMENTATION.md    ← Full narrative guide with screenshots
├── README.md           ← This file
└── Images/             ← All screenshots referenced in the documentation
```

---

*MPI — DeathStar S.A. Systems Administration Project*
