# 💀 DeathStar S.A. — Servidor Ubuntu con Samba 4 AD

[![Samba](https://img.shields.io/badge/Samba-4.x-blue)](https://www.samba.org/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20LTS-orange)](https://ubuntu.com/)
[![Estado](https://img.shields.io/badge/Estado-Completado-green)]()

> Documentación completa del despliegue de un servidor Ubuntu como Controlador de Dominio Active Directory con Samba 4, cliente Linux unido al dominio, carpetas compartidas, servidor secundario y relación de confianza entre dominios.

---

## 🏗️ Infraestructura

Todas las VMs tienen dos adaptadores de red en VirtualBox:
- **Adaptador 1 (enp0s3)**: NAT — para salir a internet e instalar paquetes
- **Adaptador 2 (enp0s8)**: Red interna `intnet` — para el tráfico del dominio

| Máquina | Hostname | IP Red Interna (enp0s8) | Rol |
|---------|----------|------------------------|-----|
| srv01 | `srv01.deathstar.local` | `192.168.10.1/24` | DC principal |
| cli01 | `cli01` | `192.168.10.10/24` | Cliente Linux |
| bespin01 | `bespin01.cloud01.city` | `192.168.10.2/24` | DC secundario |
| lab01 | `lab01.lab01.lan` | `192.168.10.3/24` | DC para trust |

### Detalles srv01

| Campo        | Valor                    |
|--------------|--------------------------|
| Dominio      | `DEATHSTAR.LOCAL`        |
| Disco 1      | 20 GB — `/`              |
| Disco 2      | 10 GB — `/home`          |
| SO           | Ubuntu Server 22.04 LTS  |

---

## 📚 Documentación

| Documento | Descripción |
|-----------|-------------|
| [📖 DOCUMENTACION_COMPLETA.md](docs/DOCUMENTACION_COMPLETA.md) | Guía paso a paso con todos los apartados del proyecto |
| [🔧 TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Solución a problemas comunes |
| [⚡ REFERENCIA_RAPIDA.md](docs/REFERENCIA_RAPIDA.md) | Comandos más usados |

---

## 📁 Estructura del repositorio

```
deathstar-samba/
│
├── README.md
│
├── docs/
│   ├── DOCUMENTACION_COMPLETA.md
│   ├── TROUBLESHOOTING.md
│   └── REFERENCIA_RAPIDA.md
│
├── scripts/
│   ├── r2d2.sh                  ← Script tarea programada cron
│   └── crear-usuarios.sh        ← Creación de usuarios en bloque
│
├── examples/
│   ├── smb.conf                 ← Configuración Samba de ejemplo
│   └── krb5.conf                ← Configuración Kerberos de ejemplo
│
└── images/                      ← Capturas de pantalla (01 a 36)
```

---

## ✅ Checklist del proyecto

- [x] Servidor con 2 discos — `/home` en disco secundario
- [x] IPv6 desactivado y systemd-resolved deshabilitado
- [x] NTP clásico configurado (sin chrony)
- [x] Dominio `DEATHSTAR.LOCAL` provisionado con Samba 4
- [x] Cliente Linux unido al dominio
- [x] Carpetas compartidas con permisos por usuario/grupo
- [x] Apache para acceso web a shares family y force
- [x] Servidor secundario `cloud01.city` (bespin01)
- [x] Acceso SSH + proceso `sl` con señales POSIX
- [x] Tarea programada con cron y script `r2d2.sh`
- [x] Trust bidireccional entre `DEATHSTAR.LOCAL` y `LAB01.LAN`

---

## 🔑 Credenciales (laboratorio)

| Usuario | Contraseña | Dominio | Rol |
|---------|-----------|---------|-----|
| `administrator` | `P@ssw0rd2024!` | DEATHSTAR.LOCAL | Admin DC principal |
| `leia` | `P@ssw0rd2024!` | DEATHSTAR.LOCAL | Usuario |
| `anakin` | `P@ssw0rd2024!` | DEATHSTAR.LOCAL | Usuario |
| `yoda` | `P@ssw0rd2024!` | DEATHSTAR.LOCAL | Usuario |
| `lando` | `P@ssw0rd2024!` | CLOUD01.CITY | Usuario |
| `boba` | `P@ssw0rd2024!` | CLOUD01.CITY | Usuario (sin acceso a trap) |
| `administrator` | `P@ssw0rd2024!` | LAB01.LAN | Admin DC trust |

> ⚠️ Contraseñas de laboratorio únicamente. Nunca usar en producción.
