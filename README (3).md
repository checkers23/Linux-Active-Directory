# 💀 DeathStar S.A. — Ubuntu Server with Samba 4 AD

[![Samba](https://img.shields.io/badge/Samba-4.x-blue)](https://www.samba.org/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20LTS-orange)](https://ubuntu.com/)
[![Status](https://img.shields.io/badge/Status-Completed-green)]()

> Complete documentation for deploying an Ubuntu Server as an Active Directory Domain Controller using Samba 4. Covers domain provisioning, Linux client join, shared folders with permissions, a secondary cloud server, SSH process control, scheduled tasks, and a cross-domain trust relationship.

---

## 🏗️ Infrastructure

All VMs are configured in VirtualBox with **two network adapters**:

- **Adapter 1 → `enp0s3`** — NAT — internet access for package installation
- **Adapter 2 → `enp0s8`** — Internal Network (`intnet`) — domain/AD traffic, static IP

> ⚠️ Replace `XX` with your classroom number throughout the entire project.

| Machine | Hostname | `enp0s3` NAT IP | `enp0s8` Internal IP | Role |
|---------|----------|-----------------|----------------------|------|
| srvWXX | `srvWXX.SWXX.local` | `172.30.20.10` | `192.168.1.1/24` | Main DC |
| BespinXX | `BespinXX.cloudXX.city` | `172.30.20.20` | `192.168.1.2/24` | Secondary DC (AWS) |
| ucXX | `ucXX` | `172.30.20.30` | `192.168.1.3/24` | Linux Client |
| labXX | `labXX.labXX.lan` | `172.30.20.40` | `192.168.1.4/24` | DC for trust |

### srvWXX — Main Server Details

| Field | Value |
|-------|-------|
| Domain | `SWXX.LOCAL` |
| Disk 1 | 20 GB → `/` |
| Disk 2 | 10 GB → `/home` |
| OS | Ubuntu Server 22.04 LTS |

---

## 📚 Documentation

All steps are documented in a single file:

**[📖 docs/DOCUMENTATION.md](docs/DOCUMENTATION.md)**

---

## 📁 Repository Structure

```
deathstar-samba/
├── README.md
├── docs/
│   └── DOCUMENTATION.md     ← Full step-by-step guide
└── images/                  ← Screenshots (taken during the lab)
```

---

## ✅ Project Checklist

- [x] Server with 2 disks — `/home` on the secondary disk
- [x] IPv6 disabled, systemd-resolved disabled
- [x] Classic NTP configured (no chrony)
- [x] Domain `SWXX.LOCAL` provisioned with Samba 4
- [x] Linux client `ucXX` joined to the domain
- [x] Shared folders: `blueprints`, `family`, `force` with correct permissions
- [x] Apache web access for `family` and `force` shares
- [x] Secondary server `BespinXX` with domain `cloudXX.city` and share `trap`
- [x] SSH access + `sl` process paused/resumed with POSIX signals
- [x] Scheduled cron task running `r2d2.sh` at 10:30
- [x] Bidirectional trust between `SWXX.LOCAL` and `LABXX.LAN`

---

## 🔑 Lab Credentials

| User | Password | Domain | Role |
|------|----------|--------|------|
| `administrator` | `P@ssw0rd2024!` | SWXX.LOCAL | Main DC Admin |
| `leia` | `P@ssw0rd2024!` | SWXX.LOCAL | User (blueprints only) |
| `anakin` | `P@ssw0rd2024!` | SWXX.LOCAL | User (force, not family) |
| `yoda` | `P@ssw0rd2024!` | SWXX.LOCAL | User (force) |
| `lando` | `P@ssw0rd2024!` | CLOUDXX.CITY | User (trap access) |
| `boba` | `P@ssw0rd2024!` | CLOUDXX.CITY | User (no trap access) |
| `administrator` | `P@ssw0rd2024!` | LABXX.LAN | Trust DC Admin |

> ⚠️ Lab credentials only — never use in production.
