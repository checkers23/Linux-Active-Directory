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
| srv01 | `srv01.deathstar.local` | `172.30.20.10` | `192.168.1.1/24` | Main DC |
| bespin01 | `bespin01.cloud01.city` | `172.30.20.20` | `192.168.1.2/24` | Secondary DC (AWS) |
| uc01 | `uc01` | `172.30.20.30` | `192.168.1.3/24` | Linux Client |
| lab01 | `lab01.lab01.lan` | `172.30.20.40` | `192.168.1.4/24` | DC for trust |

### srv01 — Main Server Details

| Field | Value |
|-------|-------|
| Domain | `DEATHSTAR.LOCAL` |
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
- [x] Domain `DEATHSTAR.LOCAL` provisioned with Samba 4
- [x] Linux client `uc01` joined to the domain
- [x] Shared folders: `blueprints`, `family`, `force` with correct permissions
- [x] Apache web access for `family` and `force` shares
- [x] Secondary server `bespin01` with domain `cloud01.city` and share `trap`
- [x] SSH access + `sl` process paused/resumed with POSIX signals
- [x] Scheduled cron task running `r2d2.sh` at 10:30
- [x] Bidirectional trust between `DEATHSTAR.LOCAL` and `LAB01.LAN`

---

## 🔑 Lab Credentials

| User | Password | Domain | Role |
|------|----------|--------|------|
| `administrator` | `P@ssw0rd2024!` | DEATHSTAR.LOCAL | Main DC Admin |
| `leia` | `P@ssw0rd2024!` | DEATHSTAR.LOCAL | User (blueprints only) |
| `anakin` | `P@ssw0rd2024!` | DEATHSTAR.LOCAL | User (force, not family) |
| `yoda` | `P@ssw0rd2024!` | DEATHSTAR.LOCAL | User (force) |
| `lando` | `P@ssw0rd2024!` | CLOUD01.CITY | User (trap access) |
| `boba` | `P@ssw0rd2024!` | CLOUD01.CITY | User (no trap access) |
| `administrator` | `P@ssw0rd2024!` | LAB01.LAN | Trust DC Admin |

> ⚠️ Lab credentials only — never use in production.
