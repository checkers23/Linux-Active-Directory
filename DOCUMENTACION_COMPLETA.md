# 📖 Documentación Completa — DeathStar S.A.

> Guía paso a paso para configurar el servidor Ubuntu como Controlador de Dominio Active Directory (Samba 4), unir un cliente Linux, configurar carpetas compartidas, servidor secundario en la nube, SSH, cron y trust entre dominios.

---

## Índice

1. [Preparación del servidor y discos](#1-preparación-del-servidor-y-discos)
2. [Configuración de red y hostname](#2-configuración-de-red-y-hostname)
3. [Sincronización horaria con NTP](#3-sincronización-horaria-con-ntp)
4. [Instalación y provisión de Samba AD DC](#4-instalación-y-provisión-de-samba-ad-dc)
5. [Verificación del dominio y Kerberos](#5-verificación-del-dominio-y-kerberos)
6. [Unir cliente Linux al dominio](#6-unir-cliente-linux-al-dominio)
7. [Usuarios y grupos de dominio](#7-usuarios-y-grupos-de-dominio)
8. [Carpetas compartidas con permisos](#8-carpetas-compartidas-con-permisos)
9. [Servidor secundario en la nube (cloud01.city)](#9-servidor-secundario-en-la-nube-cloud01city)
10. [Acceso SSH y gestión de procesos](#10-acceso-ssh-y-gestión-de-procesos)
11. [Tarea programada con cron](#11-tarea-programada-con-cron)
12. [Relación de confianza entre dominios (Trust)](#12-relación-de-confianza-entre-dominios-trust)

---

## 1. Preparación del servidor y discos

Durante la instalación de Ubuntu Server 22.04 se configuraron dos discos directamente en el particionado del instalador:

- **Disco 1** (`/dev/sda`): 20 GB → `/`
- **Disco 2** (`/dev/sdb`): 10 GB → `/home`

El instalador ya escribe las entradas correctas en `/etc/fstab` automáticamente, no es necesario ningún paso adicional. Solo verificamos que todo está correcto:

```bash
lsblk
df -h
cat /etc/fstab
```

> 📸 **Captura:** `images/01_lsblk_discos.png` — Salida de lsblk mostrando /dev/sda montado en / y /dev/sdb en /home.
> 📸 **Captura:** `images/02_df_home.png` — Salida de df -h confirmando el montaje de /home en el disco secundario.

---

## 2. Configuración de red y hostname

### 2.1 Establecer hostname

```bash
sudo hostnamectl set-hostname srv01.deathstar.local
hostname -f
# Debe devolver: srv01.deathstar.local
```

### 2.2 Editar `/etc/hosts`

```bash
sudo nano /etc/hosts
```

Contenido:

```
127.0.0.1       localhost
192.168.10.1    srv01.deathstar.local srv01
```

> ⚠️ Eliminar cualquier línea que asocie `127.0.1.1` con el hostname anterior. Solo debe quedar la línea con la IP interna del servidor.

### 2.3 Configurar interfaces de red con Netplan

El servidor tiene **dos interfaces de red** configuradas en VirtualBox:

- **enp0s3** → Adaptador puente (Bridge) con IP estática `172.30.20.1/24` → para salir a internet e instalar paquetes
- **enp0s8** → Red interna (`intnet`) con IP estática `192.168.10.1/24` → para el tráfico del dominio AD

> En VirtualBox: Configuración de la VM → Red:
> - Adaptador 1: **Adaptador puente** (seleccionar tu tarjeta de red física)
> - Adaptador 2: **Red interna** (nombre: `intnet`)

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
        - 172.30.20.1/24
      routes:
        - to: default
          via: 172.30.20.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
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

Explicación:
- `enp0s3`: IP estática en la red de tu casa (`172.30.20.x`), con el gateway de tu router. Para tener internet e instalar paquetes.
- `enp0s8`: IP estática `192.168.10.1` en la red interna del dominio. El DNS apunta a `127.0.0.1` porque el propio servidor gestionará el DNS del dominio.

> ⚠️ El gateway (`172.30.20.254`) debe ser la IP de tu router. Comprueba con `ip a` antes de editar cuál es la IP que tiene tu red y ajusta el rango `172.30.20.x` en consecuencia.

```bash
sudo netplan apply
ip a
ip route
```

Verificar conectividad:

```bash
# Salida a internet (por enp0s3)
ping -c 4 8.8.8.8

# Red interna (por enp0s8)
ping -c 4 192.168.10.1
```

> 📸 **Captura:** `images/05_netplan_ip.png` — Salida de ip a mostrando enp0s3 con 172.30.20.1 y enp0s8 con 192.168.10.1.

### 2.4 Deshabilitar IPv6

IPv6 puede causar conflictos con Samba y Kerberos cuando el sistema intenta resolver nombres por IPv6 antes que por IPv4. En un entorno de laboratorio con todo configurado en IPv4 es recomendable desactivarlo.

```bash
sudo nano /etc/sysctl.conf
```

Añadir al final del fichero:

```
# Deshabilitar IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

Aplicar los cambios sin reiniciar:

```bash
sudo sysctl -p
```

Verificar que IPv6 está desactivado:

```bash
ip a | grep inet6
```

Si no devuelve nada, IPv6 está correctamente desactivado.

> 📸 **Captura:** `images/05b_ipv6_desactivado.png` — Salida de ip a sin direcciones inet6.

### 2.5 Deshabilitar systemd-resolved

⚠️ **CRÍTICO:** Samba necesita el puerto 53 libre para gestionar su propio DNS interno. Ubuntu 22.04 arranca `systemd-resolved` por defecto, que ocupa ese puerto. Si no lo desactivamos, Samba no podrá iniciar su servidor DNS y el dominio no funcionará.

```bash
# Parar y deshabilitar systemd-resolved
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

# Eliminar el enlace simbólico de resolv.conf
sudo unlink /etc/resolv.conf

# Crear resolv.conf manual
sudo nano /etc/resolv.conf
```

Contenido:

```
nameserver 127.0.0.1
nameserver 8.8.8.8
search deathstar.local
```

Verificar que el puerto 53 está libre:

```bash
sudo ss -tulnp | grep :53
```

Si no devuelve nada, el puerto está libre y Samba podrá usarlo.

> 📸 **Captura:** `images/05c_resolved_disabled.png` — Salida de ss -tulnp sin nada en el puerto 53.

---

## 3. Sincronización horaria con NTP

Kerberos (usado por Active Directory) falla si los relojes difieren más de 5 minutos. Es obligatorio configurar NTP. Usamos `ntpd` clásico, **no chrony**.

### 3.1 Quitar chrony si está instalado

Ubuntu 22.04 instala chrony por defecto. Hay que desactivarlo antes:

```bash
sudo systemctl stop chronyd
sudo systemctl disable chronyd
sudo apt remove -y chrony
```

### 3.2 Instalar NTP clásico

```bash
sudo apt install -y ntp
```

### 3.3 Configurar `/etc/ntp.conf` en el servidor (srv01)

```bash
sudo nano /etc/ntp.conf
```

Añadir o asegurar estas líneas:

```
# Servidores NTP públicos
pool ntp.ubuntu.com iburst
pool pool.ntp.org iburst

# Permitir que los clientes de la red sincronicen contra este servidor
restrict default nomodify nopeer noquery limited kod
restrict 127.0.0.1
restrict ::1
restrict 192.168.10.0 mask 255.255.255.0 nomodify nopeer
```

```bash
sudo systemctl restart ntp
sudo systemctl enable ntp
```

### 3.4 Configurar AppArmor para NTP signing de Samba

Samba genera automáticamente el directorio `/var/lib/samba/ntp_signd/` para firmar paquetes NTP, necesario para que los clientes Windows sincronicen correctamente con el DC. Hay que dar permisos a ntpd sobre ese directorio:

```bash
sudo nano /etc/apparmor.d/usr.sbin.ntpd
```

Añadir dentro del bloque existente:

```
/var/lib/samba/ntp_signd r,
/var/lib/samba/ntp_signd/** rw,
```

Aplicar los cambios:

```bash
sudo systemctl restart apparmor
sudo systemctl restart ntp
```

### 3.5 Verificar sincronización

```bash
ntpq -p
```

Salida esperada (el `*` indica la fuente activa):

```
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*ntp.ubuntu.com  .GPS.            1 u   45   64  377   15.231    0.412   0.108
```

```bash
timedatectl
```

Debe mostrar:

```
System clock synchronized: yes
NTP service: active
```

> 📸 **Captura:** `images/06_ntp_sync.png` — Salida de ntpq -p y timedatectl mostrando sincronización activa.

---

## 4. Instalación y provisión de Samba AD DC

### 4.1 Instalar paquetes necesarios

```bash
sudo apt install -y samba krb5-user samba-dsdb-modules samba-vfs-modules winbind
```

> Cuando pregunte por el realm de Kerberos, escribir `DEATHSTAR.LOCAL` (en mayúsculas).

### 4.2 Parar servicios existentes y limpiar configuración

```bash
sudo systemctl stop smbd nmbd winbind 2>/dev/null || true
sudo systemctl disable smbd nmbd winbind 2>/dev/null || true

# Hacer copia de seguridad de la configuración existente
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig 2>/dev/null || true
sudo mv /etc/krb5.conf /etc/krb5.conf.bak 2>/dev/null || true
```

### 4.3 Provisionar el dominio

```bash
sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=DEATHSTAR.LOCAL \
  --domain=DEATHSTAR \
  --adminpass='P@ssw0rd2024!' \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL
```

Opciones explicadas:
- `--use-rfc2307`: habilita atributos POSIX/UNIX para compatibilidad con Linux
- `--realm`: el realm Kerberos en MAYÚSCULAS
- `--domain`: el nombre NetBIOS del dominio
- `--adminpass`: contraseña del administrador (debe ser compleja)
- `--server-role=dc`: este servidor será Controlador de Dominio
- `--dns-backend=SAMBA_INTERNAL`: usa el DNS interno de Samba

> 📸 **Captura:** `images/07_domain_provision.png` — Salida completa del comando samba-tool domain provision.

### 4.4 Copiar configuración de Kerberos generada por Samba

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

### 4.5 Iniciar y habilitar el servicio

```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
sudo systemctl status samba-ad-dc
```

> 📸 **Captura:** `images/08_samba_status.png` — Salida de systemctl status samba-ad-dc mostrando active (running).

### 4.6 Verificar que todos los puertos AD están escuchando

```bash
sudo ss -tulnp | grep -E ':(53|88|389|445|636|3268)'
```

Resultado esperado — deben aparecer todos estos puertos:

```
53   → DNS
88   → Kerberos
389  → LDAP
445  → SMB
636  → LDAPS
3268 → Global Catalog
```

> 📸 **Captura:** `images/08b_puertos_ad.png` — Salida de ss mostrando todos los puertos AD activos.

---

## 5. Verificación del dominio y Kerberos

### 5.1 Comprobar nivel del dominio

```bash
sudo samba-tool domain level show
```

### 5.2 Verificar DNS interno

```bash
host -t SRV _kerberos._tcp.deathstar.local 127.0.0.1
host -t SRV _ldap._tcp.deathstar.local 127.0.0.1
host -t A srv01.deathstar.local 127.0.0.1
```

> 📸 **Captura:** `images/09_dns_check.png` — Salida de los comandos host mostrando resolución correcta de registros SRV y A.

### 5.3 Obtener ticket Kerberos del administrador

```bash
kinit administrator@DEATHSTAR.LOCAL
# Introducir contraseña: P@ssw0rd2024!

klist

# Limpiar el ticket cuando ya no se necesite
kdestroy
```

Salida esperada:

```
Credentials cache: FILE:/tmp/krb5cc_0
        Principal: administrator@DEATHSTAR.LOCAL

  Issued                Expires               Principal
Jun  1 10:00:00 2024  Jun  1 20:00:00 2024  krbtgt/DEATHSTAR.LOCAL@DEATHSTAR.LOCAL
```

> 📸 **Captura:** `images/10_kinit_klist.png` — Salida de klist mostrando el ticket Kerberos del administrador.

### 5.4 Probar acceso SMB

```bash
smbclient -L localhost -U administrator%P@ssw0rd2024!
```

> 📸 **Captura:** `images/11_smbclient.png` — Listado de shares del servidor mediante smbclient.

---

## 6. Unir cliente Linux al dominio

> Estos pasos se realizan en la máquina cliente `cli01` (Ubuntu Desktop 22.04).

### 6.1 Configurar red del cliente

El cliente también tiene **dos interfaces** en VirtualBox:
- **enp0s3** → Adaptador puente, IP estática `172.30.20.10/24` → para internet
- **enp0s8** → Red interna (`intnet`), IP estática `192.168.10.10/24` → red del dominio

> ⚠️ El nombre `intnet` debe ser exactamente el mismo que en srv01 para que se vean entre ellos.

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
          via: 172.30.20.254
      nameservers:
        addresses:
          - 8.8.8.8
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.10.10/24
      nameservers:
        addresses:
          - 192.168.10.1       # DNS = nuestro DC
          - 8.8.8.8
        search:
          - deathstar.local
```

```bash
sudo netplan apply
ip a
```

Verificar conectividad con el servidor:

```bash
ping -c 4 192.168.10.1
```

### 6.2 Deshabilitar IPv6 en el cliente

Igual que en el servidor, desactivamos IPv6 para evitar conflictos:

```bash
sudo nano /etc/sysctl.conf
```

Añadir al final:

```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

```bash
sudo sysctl -p
ip a | grep inet6
```

> 📸 **Captura:** `images/11b_cliente_ipv6_desactivado.png` — Salida de ip a en el cliente sin direcciones inet6.

### 6.4 Verificar resolución DNS desde el cliente

```bash
host -t A srv01.deathstar.local
host -t SRV _kerberos._tcp.deathstar.local
```

### 6.5 Instalar paquetes en el cliente

```bash
sudo apt install -y realmd sssd sssd-tools krb5-user samba-common packagekit adcli
```

> Cuando pregunte por el realm de Kerberos, escribir `DEATHSTAR.LOCAL`.

### 6.6 Sincronizar hora con el DC

Igual que en el servidor, quitamos chrony y usamos ntpd clásico:

```bash
sudo systemctl stop chronyd
sudo systemctl disable chronyd
sudo apt remove -y chrony

sudo apt install -y ntp
sudo nano /etc/ntp.conf
```

Añadir (el cliente sincroniza directamente contra el DC, no contra internet):

```
server 192.168.10.1 iburst
```

```bash
sudo systemctl restart ntp
ntpq -p
```

### 6.7 Descubrir y unirse al dominio

```bash
realm discover DEATHSTAR.LOCAL
```

> 📸 **Captura:** `images/12_realm_discover.png` — Salida de realm discover mostrando información del dominio DEATHSTAR.LOCAL.

```bash
sudo realm join --user=Administrator DEATHSTAR.LOCAL
# Introducir contraseña: P@ssw0rd2024!
```

> 📸 **Captura:** `images/13_realm_join.png` — Salida de realm join (sin errores indica éxito).

### 6.8 Verificar unión al dominio

```bash
realm list
id leia@DEATHSTAR.LOCAL
```

> 📸 **Captura:** `images/14_realm_list_id.png` — Salida de realm list y del comando id mostrando el usuario de dominio.

### 6.9 Permitir inicio de sesión sin especificar el dominio (opcional)

```bash
sudo nano /etc/sssd/sssd.conf
```

Buscar y modificar:

```ini
use_fully_qualified_names = False
```

```bash
sudo systemctl restart sssd
```

---

## 7. Usuarios y grupos de dominio

> Ejecutar en el servidor `srv01`.

### 7.1 Crear usuarios de dominio

```bash
sudo samba-tool user add leia P@ssw0rd2024!
sudo samba-tool user add anakin P@ssw0rd2024!
sudo samba-tool user add yoda P@ssw0rd2024!
```

### 7.2 Crear grupo `jedis`

```bash
sudo samba-tool group add jedis
```

### 7.3 Añadir usuarios al grupo jedis

```bash
sudo samba-tool group addmembers jedis anakin
sudo samba-tool group addmembers jedis yoda
```

### 7.4 Verificar usuarios y grupos

```bash
sudo samba-tool user list
sudo samba-tool group listmembers jedis
```

> 📸 **Captura:** `images/15_usuarios_grupos.png` — Salida de samba-tool user list y group listmembers jedis.

---

## 8. Carpetas compartidas con permisos

### 8.1 Crear estructura de directorios

```bash
sudo mkdir -p /deathstar/blueprints
sudo mkdir -p /sw/family
sudo mkdir -p /sw/force
```

### 8.2 Share `blueprints` — Solo accede `leia`, no pública

**Configuración de permisos del sistema:**

```bash
# Crear grupo para blueprints
sudo groupadd blueprints_grp

# Propietario y permisos
sudo chown root:blueprints_grp /deathstar/blueprints
sudo chmod 770 /deathstar/blueprints
```

**Añadir leia al grupo local** (si se usa autenticación local) o via samba-tool:

```bash
# Mapear el grupo de dominio a permisos de carpeta
sudo samba-tool group add blueprints_access
sudo samba-tool group addmembers blueprints_access leia

# Establecer ACL de Windows sobre la carpeta
sudo samba-tool ntacl set "O:DAG:DUD:PAI(A;OICI;0x001f01ff;;;DA)(A;OICI;0x001f01ff;;;SY)(A;OICI;0x001200a9;;;S-1-5-21-XXXXXXXX-XXXXXXXX-XXXXXXXX-YYYY)" /deathstar/blueprints
```

> Para un laboratorio simplificado, usamos `smb.conf` con `valid users`:

**Añadir a `/etc/samba/smb.conf`:**

```bash
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

> 📸 **Captura:** `images/16_smb_blueprints.png` — Sección blueprints del archivo smb.conf.

### 8.3 Share `family` — `anakin` no puede acceder, pública y visible en navegador

```bash
sudo chown root:users /sw/family
sudo chmod 775 /sw/family
```

**Instalar Apache para acceso web:**

```bash
sudo apt install -y apache2

# Crear enlace simbólico o configurar alias
sudo ln -s /sw/family /var/www/html/family
```

**Añadir a `smb.conf`:**

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

**Habilitar DirectoryIndex en Apache para ver los archivos:**

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

Añadir dentro del VirtualHost:

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

> 📸 **Captura:** `images/17_family_web.png` — Navegador mostrando el contenido de /sw/family en http://192.168.10.1/family.

### 8.4 Share `force` — Solo `anakin`, `yoda` y grupo `jedis`, no `leia`, pública, web, solo lectura

```bash
sudo chown root:users /sw/force
sudo chmod 755 /sw/force
```

**Añadir a `smb.conf`:**

```ini
[force]
   path = /sw/force
   browseable = yes
   read only = yes
   guest ok = yes
   valid users = anakin@DEATHSTAR.LOCAL, yoda@DEATHSTAR.LOCAL, @jedis@DEATHSTAR.LOCAL
   invalid users = leia@DEATHSTAR.LOCAL
```

**Configurar acceso web (Apache):**

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

Añadir:

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

> 📸 **Captura:** `images/18_force_web.png` — Navegador mostrando el contenido de /sw/force en http://192.168.10.1/force.

### 8.5 Verificar sintaxis y reiniciar Samba

Antes de reiniciar, verificar que smb.conf no tiene errores de sintaxis:

```bash
testparm
```

Si no hay errores, reiniciar:

```bash
sudo systemctl restart samba-ad-dc

# Verificar shares
smbclient -L localhost -U leia%P@ssw0rd2024!
smbclient -L localhost -U anakin%P@ssw0rd2024!
```

> 📸 **Captura:** `images/19_shares_verificacion.png` — Salida de smbclient mostrando los shares accesibles para cada usuario.

---

## 9. Servidor secundario en la nube (cloud01.city)

> Estos pasos se realizan en el segundo servidor (instancia AWS o VM separada), llamado `bespin01`, con IP `10.0.0.1`.

### 9.1 Configurar hostname y hosts

```bash
sudo hostnamectl set-hostname bespin01.cloud01.city

sudo nano /etc/hosts
```

```
127.0.0.1       localhost
192.168.10.2    bespin01.cloud01.city bespin01
```

### 9.2 Configurar IP estática

bespin01 también tiene **dos interfaces** en VirtualBox:
- **enp0s3** → Adaptador puente, IP estática `172.30.20.2/24` → para internet
- **enp0s8** → Red interna (`intnet`), IP estática `192.168.10.2/24` → red interna

> El nombre de red interna `intnet` debe ser el mismo en todas las VMs para que se vean entre ellas.

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
        - 172.30.20.2/24
      routes:
        - to: default
          via: 172.30.20.254
      nameservers:
        addresses:
          - 8.8.8.8
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
ip a

# Verificar conectividad con srv01
ping -c 4 192.168.10.1
```

### 9.3 Deshabilitar systemd-resolved en bespin01

Igual que en srv01, necesitamos liberar el puerto 53 para que Samba pueda gestionarlo:

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo unlink /etc/resolv.conf

sudo nano /etc/resolv.conf
```

Contenido:

```
nameserver 127.0.0.1
nameserver 8.8.8.8
search cloud01.city
```

```bash
sudo ss -tulnp | grep :53
# Debe estar vacío
```

### 9.4 Instalar NTP y sincronizar

```bash
sudo systemctl stop chronyd
sudo systemctl disable chronyd
sudo apt remove -y chrony
sudo apt install -y ntp
```

Editar `/etc/ntp.conf`:

```
pool ntp.ubuntu.com iburst
pool pool.ntp.org iburst
```

```bash
sudo systemctl restart ntp && sudo systemctl enable ntp
ntpq -p
```

### 9.6 Instalar y provisionar Samba como DC del dominio `cloud01.city`

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

> 📸 **Captura:** `images/20_bespin_provision.png` — Salida del provisioning del dominio cloud01.city en bespin01.

### 9.7 Crear usuarios `lando` y `boba` y directorio con iniciales

```bash
sudo samba-tool user add lando P@ssw0rd2024!
sudo samba-tool user add boba P@ssw0rd2024!

# Crear directorio con iniciales (ejemplo: ABC)
sudo mkdir -p /city/trap/abc
```

### 9.8 Configurar share `trap` — Solo accede `lando`, no `boba`

```bash
sudo mkdir -p /city/trap
sudo chown root:users /city/trap
sudo chmod 770 /city/trap
```

**Añadir a `/etc/samba/smb.conf`:**

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
sudo systemctl restart samba-ad-dc
smbclient //localhost/trap -U lando%P@ssw0rd2024!
```

> 📸 **Captura:** `images/21_trap_lando.png` — Acceso exitoso de lando al share trap con smbclient.
> 📸 **Captura:** `images/22_trap_boba.png` — Acceso denegado de boba al share trap.

---

## 10. Acceso SSH y gestión de procesos

### 10.1 Habilitar SSH en el servidor

```bash
# Verificar que SSH está activo
sudo systemctl status ssh

# Si no está instalado:
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

### 10.2 Conectarse desde la máquina real al servidor

Desde tu máquina local (terminal):

```bash
ssh usuario@192.168.10.1
```

> 📸 **Captura:** `images/23_ssh_conexion.png` — Terminal mostrando la sesión SSH establecida con el servidor.

### 10.3 Instalar y lanzar `sl` (tren de vapor)

```bash
# Instalar sl si no está disponible
sudo apt install -y sl

# Lanzar sl en primer plano
sl
```

### 10.4 Pausar y reanudar el proceso

Hay dos formas de hacerlo, ambas válidas:

**Método 1 — Ctrl+Z y bg/fg (desde la misma terminal):**

Con `sl` corriendo, pulsar `Ctrl+Z` para pausarlo:

```bash
jobs
# [1]+  Stopped    sl

bg %1    # reanudar en segundo plano
fg %1    # traer al primer plano
```

**Método 2 — Señales POSIX con kill (desde otra terminal):**

Este método demuestra el control de procesos a bajo nivel mediante señales POSIX:

```bash
# Desde una segunda terminal, obtener el PID
pgrep -x sl

# SIGSTOP (señal 19) — congela el proceso
kill -19 $(pgrep -x sl)

# SIGCONT (señal 18) — reanuda el proceso desde donde se paró
kill -18 $(pgrep -x sl)
```

La diferencia: `Ctrl+Z` envía SIGTSTP (que el proceso puede ignorar), mientras que `kill -19` envía SIGSTOP que el kernel aplica directamente y el proceso no puede ignorar.

> 📸 **Captura:** `images/24_sl_stop_bg_fg.png` — Terminal mostrando sl lanzado, pausado y reanudado (con cualquiera de los dos métodos).

---

## 11. Tarea programada con cron

### 11.1 Crear el script `r2d2.sh`

```bash
sudo nano /usr/local/bin/r2d2.sh
```

Contenido del script:

```bash
#!/bin/bash
# r2d2.sh - Crea el fichero luke.sky

touch /home/administrador/luke.sky
echo "Fichero luke.sky creado el $(date)" >> /var/log/r2d2.log
```

```bash
sudo chmod +x /usr/local/bin/r2d2.sh
```

> 📸 **Captura:** `images/25_r2d2_script.png` — Contenido del script r2d2.sh mostrado con cat.

### 11.2 Crear la tarea programada en crontab

```bash
crontab -e
```

Añadir la siguiente línea al final:

```
30 10 * * * /usr/local/bin/r2d2.sh
```

Esto ejecutará el script cada día a las **10:30**.

```bash
# Verificar que la tarea está registrada
crontab -l
```

> 📸 **Captura:** `images/26_crontab.png` — Salida de crontab -l mostrando la tarea programada a las 10:30.

### 11.3 Verificar funcionamiento (prueba manual)

```bash
# Ejecutar el script manualmente para comprobar
/usr/local/bin/r2d2.sh

# Verificar que se ha creado el fichero
ls -la /home/administrador/luke.sky
cat /var/log/r2d2.log
```

> 📸 **Captura:** `images/27_luke_sky.png` — Salida de ls mostrando el fichero luke.sky creado y el log de r2d2.

---

## 12. Relación de confianza entre dominios (Trust)

Se establece una relación de confianza entre `DEATHSTAR.LOCAL` (srv01) y `LAB01.LAN` (cuarta VM creada para simular el servidor de prácticas).

### 12.1 Configurar la cuarta VM — DC de `lab01.lan`

La cuarta VM (`lab01`) tiene también dos interfaces en VirtualBox:
- **enp0s3** → Adaptador puente, IP `172.30.20.3/24`
- **enp0s8** → Red interna (`intnet`), IP `192.168.10.3/24`

Seguir los mismos pasos que con srv01 (hostname, hosts, netplan, deshabilitar IPv6 y systemd-resolved, NTP, provisionar Samba) con estos valores distintos:

```bash
sudo hostnamectl set-hostname lab01.lab01.lan
```

`/etc/hosts`:
```
127.0.0.1     localhost
192.168.10.3  lab01.lab01.lan lab01
```

Netplan (`enp0s8`): IP `192.168.10.3/24`, DNS `127.0.0.1`, search `lab01.lan`

Provisioning:
```bash
sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=LAB01.LAN \
  --domain=LAB01 \
  --adminpass='P@ssw0rd2024!' \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL
```

### 12.2 Verificar conectividad entre los dos DCs

Desde srv01, comprobar que llega a lab01:

```bash
ping -c 4 192.168.10.3
host -t SRV _kerberos._tcp.lab01.lan 192.168.10.3
```

Desde lab01, comprobar que llega a srv01:

```bash
ping -c 4 192.168.10.1
host -t SRV _kerberos._tcp.deathstar.local 192.168.10.1
```

### 12.3 Configurar DNS forwarding entre dominios

**En srv01** (`/etc/samba/smb.conf`), sección `[global]`:

```bash
sudo nano /etc/samba/smb.conf
```

```ini
dns forwarder = 192.168.10.3
```

**En lab01** (`/etc/samba/smb.conf`), sección `[global]`:

```ini
dns forwarder = 192.168.10.1
```

Reiniciar Samba en ambos:

```bash
sudo systemctl restart samba-ad-dc
```

Verificar resolución cruzada:

```bash
# Desde srv01
host -t SRV _ldap._tcp.lab01.lan

# Desde lab01
host -t SRV _ldap._tcp.deathstar.local
```

### 12.4 Crear la relación de confianza desde `DEATHSTAR.LOCAL`

```bash
sudo samba-tool domain trust create LAB01.LAN \
  --type=forest \
  --direction=both \
  -U administrator%P@ssw0rd2024! \
  --remote-admin="administrator%P@ssw0rd2024!"
```

> 📸 **Captura:** `images/28_trust_create.png` — Salida del comando samba-tool domain trust create mostrando el trust creado.

### 12.5 Verificar la relación de confianza

```bash
sudo samba-tool domain trust list
sudo samba-tool domain trust validate LAB01.LAN -U administrator%P@ssw0rd2024!
```

> ⚠️ Un error de netlogon durante la validación es normal en Samba 4. Lo importante es ver: `TRUST[WERR_OK]`
```

> 📸 **Captura:** `images/29_trust_verify.png` — Salida de samba-tool domain trust list y validate mostrando el trust activo.

### 12.5 Probar autenticación cruzada

Desde el servidor `srv01` (DEATHSTAR.LOCAL), obtener un ticket de un usuario del otro dominio:

```bash
kinit usuario@LAB01.LAN
klist
```

> 📸 **Captura:** `images/30_trust_kinit.png` — Ticket Kerberos de un usuario de LAB01.LAN obtenido desde DEATHSTAR.LOCAL.

---

## Resumen de capturas necesarias

| Nº  | Archivo                        | Qué debe mostrar                                      |
|-----|-------------------------------|-------------------------------------------------------|
| 01  | `01_lsblk_discos.png`         | `lsblk` con /dev/sda en / y /dev/sdb en /home        |
| 02  | `02_df_home.png`              | `df -h` confirmando montaje de /home                  |
| 05  | `05_netplan_ip.png`           | `ip a` con IP 192.168.10.1 configurada                |
| 05b | `05b_ipv6_desactivado.png`    | `ip a` sin direcciones inet6                          |
| 05c | `05c_resolved_disabled.png`   | `ss -tulnp` sin nada en el puerto 53                  |
| 06  | `06_ntp_sync.png`             | `ntpq -p` y `timedatectl` con sync activa             |
| 07  | `07_domain_provision.png`     | Salida completa de `samba-tool domain provision`      |
| 08  | `08_samba_status.png`         | `systemctl status samba-ad-dc` activo                 |
| 08b | `08b_puertos_ad.png`          | `ss -tulnp` mostrando puertos 53,88,389,445,636,3268  |
| 09  | `09_dns_check.png`            | Resolución DNS de registros SRV y A                   |
| 10  | `10_kinit_klist.png`          | `klist` con ticket de administrator                   |
| 11  | `11_smbclient.png`            | Listado de shares con smbclient                       |
| 12  | `12_realm_discover.png`       | `realm discover DEATHSTAR.LOCAL`                      |
| 13  | `13_realm_join.png`           | `realm join` sin errores                              |
| 14  | `14_realm_list_id.png`        | `realm list` e `id leia@DEATHSTAR.LOCAL`              |
| 15  | `15_usuarios_grupos.png`      | `samba-tool user list` y `group listmembers jedis`    |
| 16  | `16_smb_blueprints.png`       | Sección blueprints de smb.conf                        |
| 17  | `17_family_web.png`           | Navegador con /sw/family visible en Apache            |
| 18  | `18_force_web.png`            | Navegador con /sw/force visible en Apache             |
| 19  | `19_shares_verificacion.png`  | smbclient con shares accesibles por usuario           |
| 20  | `20_bespin_provision.png`     | Provisioning de cloud01.city                          |
| 21  | `21_trap_lando.png`           | Acceso de lando al share trap                         |
| 22  | `22_trap_boba.png`            | Acceso denegado de boba al share trap                 |
| 23  | `23_ssh_conexion.png`         | Sesión SSH establecida                                |
| 24  | `24_sl_stop_bg_fg.png`        | sl pausado con Ctrl+Z y reanudado con bg/fg           |
| 25  | `25_r2d2_script.png`          | Contenido del script r2d2.sh                          |
| 26  | `26_crontab.png`              | `crontab -l` con tarea a las 10:30                    |
| 27  | `27_luke_sky.png`             | Fichero luke.sky creado                               |
| 28  | `28_trust_create.png`         | Salida de `samba-tool domain trust create`            |
| 29  | `29_trust_verify.png`         | `samba-tool domain trust list` activo                 |
| 30  | `30_trust_kinit.png`          | Ticket Kerberos cruzado entre dominios                |
