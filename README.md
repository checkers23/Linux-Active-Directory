# 🌌 DeathStar S.A. — Linux Active Directory

> Full implementation of a Samba 4 Active Directory Domain Controller on Ubuntu Server, including domain client, shared folders, AWS secondary server, SSH process control, scheduled tasks, and domain trust relationship.

---

## 📋 Project Overview

**Company:** DeathStar S.A.  
**Author:** Miguel Pérez Ibáñez
**Repository:** https://github.com/checkers23/Linux-Active-Directory

---

## 🖥️ Infrastructure

| Server | Hostname | IP | Domain | Role |
|--------|----------|----|--------|------|
| Main DC | srv01.deathstar.local | 192.168.1.1 | deathstar.local | Domain Controller |
| Linux Client | uc01 | 192.168.1.3 | deathstar.local | Domain Member |
| Trust DC | ls07.lab07.lan | 192.168.1.2 | lab07.lan | Domain Controller |
| AWS DC | bespin07.cloud07.city | 10.0.1.10 | cloud07.city | Domain Controller (EC2) |

---

## 📁 Repository Structure

```
├── DOCUMENTATION.md       # Full step-by-step guide
├── README.md              # This file
└── images/                # Screenshots folder
    ├── images-001_lsblk_discos
    ├── images-002_hostname_y_etc_hosts
    ├── images-003-Netplan_configuration_srv1
    ├── images-004-resolv.conf_srv1
    ├── images-005-kerberos_reino
    ├── images-006-Domain_Provision_srv1
    ├── images-007-smb.conf_srv1
    ├── images-008_Kerberos_pruebas_srv1
    ├── images-009_creacion_usuarios_carpetas
    ├── images-010_smb.conf
    ├── images-011_Sl_Stop
    ├── images-012_Sl_Continue
    ├── images-013_script
    ├── images-013_script_r2d2
    ├── images-014_comprobaciones_permisos_usuarios
    ├── images-015_Configuracion_Red_Cliente
    ├── images-016_resolv.conf_client
    ├── images-017_Instalacion_realmssd_Client
    ├── images-018_Realm_Discover
    ├── images-019_Realm_Join
    ├── images-020_Prueba_Usuario_Cliente
    ├── images-021_Prueba_Usuarios_Cliente_v2
    ├── images-022_Creacion_VPC
    ├── images-023_Verificacion_VPC
    ├── images-024_Habilitar_IP_Automatica_VPC
    ├── images-025_Grupos_De_Seguridad_Y_Reglas
    ├── images-026_ssh_desde_windows_a_ec2
    ├── images-026_ssh_hostname_etc_hosts_bespin07
    ├── images-027_Aprovisionamiento_bespin01
    ├── images-028_Dns_Kinit_Bespin07
    ├── images-029_UnMask_Status_bespin01
    ├── images-030_Creacion_Usuarios_bespin01
    ├── images-031_Creacion_Carpetas_bespin01
    ├── images-031_Creacion_Carpetas_Y_Comprobaciones_bespin01
    ├── images-032_ParaElTrust_Añado_DnsForwarder_a_smb-conf_srv01
    ├── images-033_ParaElTrust_Añado_DnsForwarder_a_smb-conf_ls07(servidor2)
    ├── images-034_Trust_Create
    ├── images-035_Trust_Comprobaciones_srv01
    └── images-036_Trust_Comprobaciones_ls07(SERVER2)
```

---

## ✅ Tasks Completed

- [x] Ubuntu Server with 2 disks (/ and /home)
- [x] Samba 4 AD DC — domain deathstar.local
- [x] Domain users: leia, anakin, yoda — group: jedis
- [x] Shared folders: blueprints, family, force (with ACL permissions)
- [x] Linux client joined to domain
- [x] AWS EC2 server — domain cloud07.city
- [x] Shared folder trap (lando in, boba denied) + MPI subfolder
- [x] SSH from physical machine + sl process control (SIGSTOP/SIGCONT)
- [x] Cron task at 10:30 — r2d2.sh creates luke.sky
- [x] Bidirectional trust between deathstar.local and lab07.lan

---

## 🔑 Key Commands Reference

```bash
# Check domain
sudo samba-tool domain info 127.0.0.1

# List users
sudo samba-tool user list

# Kerberos ticket (always UPPERCASE realm)
kinit Administrator@DEATHSTAR.LOCAL
klist

# Check trust
sudo samba-tool domain trust list

# Test share access
smbclient //localhost/sharename -U username%password
```

---

## ⚠️ Common Issues and Fixes

| Problem | Fix |
|---------|-----|
| `samba-ad-dc.service does not exist` | `sudo apt install -y samba-ad-dc` then unmask |
| `KDC reply did not match expectations` | Check /etc/hosts — remove 127.0.1.1 line |
| `Client not found in Kerberos database` | Re-provision: always install packages BEFORE provisioning |
| Duplicate DNS IP (NAT 10.0.2.x) | `sudo samba-tool dns delete 127.0.0.1 domain.local hostname A 10.0.2.15` |
| SSH PEM bad permissions | `icacls key.pem /inheritance:r` then remove unwanted groups |
