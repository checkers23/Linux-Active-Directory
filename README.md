# 💀 DeathStar S.A. — Servidor Ubuntu con Samba 4 AD

[![Samba](https://img.shields.io/badge/Samba-4.x-blue)](https://www.samba.org/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20LTS-orange)](https://ubuntu.com/)
[![Estado](https://img.shields.io/badge/Estado-Completado-green)]()

> Documentación completa del despliegue de un servidor Ubuntu como Controlador de Dominio Active Directory con Samba 4, cliente Linux unido al dominio, carpetas compartidas, servidor secundario en la nube, sincronización horaria con NTP y relación de confianza entre dominios.

---

## 🏗️ Infraestructura

### Servidor Principal — `srv01.deathstar.local`

| Campo        | Valor                    |
|--------------|--------------------------|
| Dominio      | `DEATHSTAR.LOCAL`        |
| Hostname     | `srv01.deathstar.local`  |
| IP puente    | `172.30.20.1/24`         |
| IP interna   | `192.168.10.1/24`        |
| SO           | Ubuntu Server 22.04 LTS  |
| Disco 1      | 20 GB (raíz `/`)         |
| Disco 2      | 10 GB (montado en `/home`)|

### Cliente Linux — `cli01`

| Campo      | Valor                   |
|------------|-------------------------|
| Hostname   | `cli01`                 |
| IP puente  | `172.30.20.10/24`       |
| IP interna | `192.168.10.10/24`      |
| SO         | Ubuntu Desktop 22.04    |
| Estado     | ✅ Unido al dominio     |

### Servidor Secundario — `bespin01.cloud01.city`

| Campo      | Valor                  |
|------------|------------------------|
| Dominio    | `CLOUD01.CITY`         |
| Hostname   | `bespin01.cloud01.city`|
| IP puente  | `172.30.20.2/24`       |
| IP interna | `192.168.10.2/24`      |
| SO         | Ubuntu Server 22.04    |

### Servidor Trust — `lab01.lab01.lan`

| Campo      | Valor                  |
|------------|------------------------|
| Dominio    | `LAB01.LAN`            |
| Hostname   | `lab01.lab01.lan`      |
| IP puente  | `172.30.20.3/24`       |
| IP interna | `192.168.10.3/24`      |
| SO         | Ubuntu Server 22.04    |

---

## 📚 Documentación

| Documento | Descripción |
|-----------|-------------|
| [📖 DOCUMENTACION_COMPLETA.md](docs/DOCUMENTACION_COMPLETA.md) | Guía paso a paso completa con todos los apartados |
| [🔧 TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Solución a problemas comunes |
| [⚡ REFERENCIA_RAPIDA.md](docs/REFERENCIA_RAPIDA.md) | Comandos más usados |

---

## 📁 Estructura del repositorio

```
deathstar-samba/
│
├── README.md                        ← Estás aquí
│
├── docs/
│   ├── DOCUMENTACION_COMPLETA.md   ← Guía principal paso a paso
│   ├── TROUBLESHOOTING.md          ← Solución de problemas
│   └── REFERENCIA_RAPIDA.md        ← Comandos rápidos
│
├── scripts/
│   ├── r2d2.sh                     ← Script de tarea programada
│   └── crear-usuarios.sh           ← Creación de usuarios en bloque
│
├── examples/
│   ├── smb.conf                    ← Configuración Samba ejemplo
│   └── krb5.conf                   ← Configuración Kerberos ejemplo
│
└── images/                         ← Capturas de pantalla
    └── (ver documentación)
```

---

## ✅ Checklist del proyecto

- [x] Servidor con 2 discos configurados y `/home` en disco secundario
- [x] Dominio `DEATHSTAR.LOCAL` provisionado con Samba 4
- [x] NTP configurado para sincronización horaria
- [x] Cliente Linux unido al dominio
- [x] Carpetas compartidas con permisos correctos
- [x] Servidor secundario `cloud01.city` en AWS/nube
- [x] Acceso SSH + proceso `sl` gestionado con `bg/fg`
- [x] Tarea programada con `cron` y script `r2d2.sh`
- [x] Relación de confianza entre `DEATHSTAR.LOCAL` y `LAB01.LAN`

---

## 🔑 Credenciales del dominio (laboratorio)

| Usuario | Contraseña | Rol |
|---------|-----------|-----|
| `administrator` | `P@ssw0rd2024!` | Administrador del dominio |
| `leia` | `P@ssw0rd2024!` | Usuario de dominio |
| `anakin` | `P@ssw0rd2024!` | Usuario de dominio |
| `yoda` | `P@ssw0rd2024!` | Usuario de dominio |
| `lando` | `P@ssw0rd2024!` | Usuario servidor nube |

> ⚠️ Contraseñas de laboratorio únicamente. Nunca usar en producción.
