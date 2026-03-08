# Raspberry Pi Home Server (OMV + Docker + Homepage + Plex + rclone)

Guía paso a paso para montar un **servidor casero completo en Raspberry Pi** usando:

* OpenMediaVault (NAS)
* Docker
* Homepage (dashboard)
* Plex Media Server
* FileBrowser
* rclone (montaje SFTP remoto)
* Glances (monitorización)

El objetivo es tener un servidor donde puedas:

* gestionar discos
* reproducir películas/series con Plex
* descargar archivos desde un servidor remoto SFTP
* navegar archivos desde el navegador
* controlar todos los servicios desde un dashboard

---

# Arquitectura

Raspberry Pi

```
OpenMediaVault
│
├── Docker
│   ├── Homepage
│   ├── Plex
│   ├── FileBrowser
│   └── Glances
│
└── rclone
    └── montaje SFTP remoto
```

Dashboard principal:

```
http://raspberry
```

---

# 1. Instalar OpenMediaVault

Instalar Raspberry Pi OS Lite y después ejecutar:

```bash
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash
```

Después reiniciar.

Acceder a la interfaz:

```
http://raspberry
```

Credenciales:

```
usuario: admin
password: openmediavault
```
Cámbiala inmediatamente

---
# 2. Monta los discos en OMV
- Storage → Disks
  
    Verifica que aparece tu HDD

- Storage → File Systems
  
    Selecciona el sistema existente → Mount

- Storage → Shared Folders
  
    Crea las carpetas para docker:
  
      - /srv/docker/backup
      - /srv/docker/compose
      - /srv/docker/data
  
    Crea tu sistema de carpetas para **Pelis** y **Series**.

- Services → SMB
  
    Comparte las carpetas por Samba si quieres acceder desde otro dispositivo.

---

# 3. Instalar Docker en OMV

En OMV instalar los plugins:
 - **OMV-Extras** (openmediavault-omvextrasorg)
 - **Docker** (openmediavault-compose)

Con docker instalado vamos a Services → Compose → Config y configuramos las carpetas que creamos antes.

Services → Compose → Files y creamos Portainer:

```yaml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /srv/docker/data/portainer:/data
    restart: unless-stopped
```

Portainer quedará disponible en:

```
http://raspberry:9000
```

---

# 4. Cambiar el puerto de OMV

Por defecto OpenMediaVault usa el puerto **80** para su interfaz web.

Como vamos a usar **Homepage** como dashboard principal en ese puerto, primero debemos cambiar el puerto de OMV.

Entrar en la interfaz de OMV:

```
http://raspberry
```

Ir a:

System → Workbench → Web Administration

Cambiar port: `80 → 8080`

Aplicar cambios.

Ahora OMV estará disponible en:

```
http://raspberry:8080
```

# 5. Instalar Homepage (en Portainer)

Homepage será el **dashboard principal del servidor**.

Stack Docker:

```yaml
version: "3"

services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped

    ports:
      - "80:3000"

    volumes:
      - /srv/docker/data/homepage:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro

      # HDDs
      - /srv/dev-disk-by-uuid-XXXX:/disk1
      - /srv/dev-disk-by-uuid-YYYY:/disk2
      ...
      - /srv/dev-disk-by-uuid-ZZZZ:/diskN

    environment:
      - HOMEPAGE_ALLOWED_HOSTS=*

    group_add:
      - "977" # Cambia por el tuyo
```

Para obtener el grupo de docker (977) usar:
```
getent group docker
```

Después de desplegar el contenedor:

```
http://raspberry
```

---

# 6. Configuración de Homepage

Archivos principales:

```
/srv/docker/data/homepage/
  - services.yaml
  - widgets.yaml
  - settings.yaml
```

---

## services.yaml

Ejemplo básico:

```yaml
- Sistema:
    - OpenMediaVault:
        icon: openmediavault.png
        href: http://raspberry:8080
        description: Administración NAS

    - Portainer:
        icon: portainer.png
        href: http://raspberry:9000

    - FileBrowser:
        icon: filebrowser.png
        href: http://raspberry:8085

- Media:
    - Plex:
        icon: plex.png
        href: http://raspberry:32400
        widget:
          type: plex
          url: http://192.168.1.155:32400  # Importante usar la IP
          key: TU_TOKEN_PLEX
          user: TU_USER_PLEX
```

---

## widgets.yaml

Monitorización usando Glances.

```yaml
# --- Glances (monitor de sistema) ---
- glances:
    url: http://192.168.1.155:61208  # IP local de la Raspberry
    version: 4
    label: System
    expanded: true # show the expanded view
    cpu: true
    cputemp: true
    mem: true
    uptime: true

# --- Almacenamiento / HDDs ---
- resources:
    label: Storages
    expanded: true
    disk:
      - /
      - /disk1
      - /disk2
```

Importante usar la ruta de los HDDs del contenedor.

## settings.yaml

Para cambiar la UI.

```yaml
---
# For configuration options and examples, please see:
# https://gethomepage.dev/configs/settings/

providers:
  openweathermap: openweathermapapikey
  weatherapi: weatherapiapikey

title: Raspberry Pi Server

theme: dark
color: slate

layout:
  Sistema:
    style: row
  Media:
    style: row
```
---

# 7. Instalar Glances (monitorización)

Docker stack:

```yaml
version: "3"

services:
  glances:
    image: nicolargo/glances:latest
    container_name: glances
    restart: unless-stopped

    pid: host
    network_mode: host

    ports:
      - "61208:61208"

    environment:
      - GLANCES_OPT=-w

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

Interfaz:

```
http://raspberry:61208
```

---

# 8. Instalar Plex

Docker stack básico:

```yaml
version: "3"

services:
  plex:
    image: linuxserver/plex:latest
    container_name: plex
    restart: unless-stopped
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
      - VERSION=docker
    volumes:
      # Config Plex
      - /srv/docker/data/plex:/config

      # HDD 1
      - /srv/dev-disk-by-uuid-XXXX/Pelis:/media/movies/hdd1
      - /srv/dev-disk-by-uuid-XXXX/Series:/media/series/hdd1
      
      # HDD 2
      - /srv/dev-disk-by-uuid-YYYY/pelis2/:/media/movies/hdd2
      - /srv/dev-disk-by-uuid-YYYY/series2/:/media/series/hdd2
```

Acceso:

```
http://raspberry:32400/web
```

---

# 9. Instalar FileBrowser

Stack Docker:

```yaml
version: "3"

services:
  filebrowser:
    image: filebrowser/filebrowser
    container_name: filebrowser
    restart: unless-stopped

    ports:
      - "8085:80"

    volumes:
      # HDD 1
      - /srv/dev-disk-by-uuid-XXXX/Pelis:/srv/hdd1/pelis
      - /srv/dev-disk-by-uuid-XXXX/Series:/srv/hdd1/series
      
      # HDD 2
      - /srv/dev-disk-by-uuid-YYYY/pelis2/:/srv/hdd2/pelis
      - /srv/dev-disk-by-uuid-YYYY/series2/:/srv/hdd2/series

      # SFTP
      - /mnt/sftp:/srv/sftp

      # Base de datos y config de FileBrowser
      - /srv/docker/data/filebrowser/database:/database
      - /srv/docker/data/filebrowser/config:/config
```

Acceso:

```
http://raspberry:8085
```

---

# 10. Montar servidor SFTP remoto con rclone

Instalar rclone:

```bash
sudo apt install rclone
```

Configurar conexión:

```bash
rclone config
```

Crear conexión SFTP.

Ejemplo:

```
name: sftpserver
type: sftp
host: IP_SERVIDOR
user: USER
port: 2022
```

---

# 11. Montar el servidor

Crear punto de montaje:

```bash
sudo mkdir -p /mnt/sftp
```

Montar:

```bash
rclone mount sftpserver:/media /mnt/sftp \
--allow-other \
--vfs-cache-mode writes
```

---

# 12. Montaje automático con systemd

Crear servicio:

```bash
sudo nano /etc/systemd/system/rclone-sftp.service
```

Contenido:

```ini
[Unit]
Description=Rclone SFTP Mount
After=network-online.target

[Service]
Type=simple
User=ggilperez
ExecStart=/usr/bin/rclone mount sftpserver:/media /mnt/sftp --allow-other --vfs-cache-mode writes
ExecStop=/bin/fusermount -u /mnt/sftp
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Activar:

```bash
sudo systemctl daemon-reload
sudo systemctl enable rclone-sftp
sudo systemctl start rclone-sftp
```

---

# 13. Solución de problemas comunes

## Homepage error: Host validation failed

Añadir variable:

```
HOMEPAGE_ALLOWED_HOSTS=*
```

---

## Error Plex widget (JSON)

Usar la **IP en lugar del hostname**.

---

## Error duplicated mapping key

En YAML los discos deben ser una lista:

```yaml
disk:
  - /
  - /disk1
  - /disk2
```

---

## Error missingdocker

Montar el socket Docker:

```
/var/run/docker.sock
```

y añadir el grupo docker.

---

## HDD no aparece correctamente (disco formateado en Windows)

Si conectas un disco duro que estaba formateado en Windows (NTFS o exFAT), puede ocurrir que:

- OMV lo detecte pero no lo monte correctamente
- Docker no vea el disco
- Homepage no pueda mostrar su uso en widgets

Esto ocurre porque el disco no está montado de forma persistente en Linux.

### 1. Identificar el disco

Listar discos:

```bash
lsblk
```
Ejemplo:
```
sda
└─sda1
```
También se puede comprobar el UUID:
```
sudo blkid
```

### 2. Crear punto de montaje
```
sudo mkdir -p /mnt/disk1
```

### 3. Montar manualmente
Ejemplo para NTFS:
```
sudo mount -t ntfs-3g /dev/sda1 /mnt/disk1
```
Ejemplo para exFAT:
```
sudo mount -t exfat /dev/sda1 /mnt/disk1
```

### 4. Montaje automático al arrancar
Editar:
```
sudo nano /etc/fstab
```
Añadir una línea con el UUID del disco:
```
UUID=XXXXXXXX /mnt/disk1 ntfs-3g defaults,nofail 0 0
```
Para exFAT:
```
UUID=XXXXXXXX /mnt/disk1 exfat defaults,nofail 0 0
```

Ejemplo real:
```bash
proc            /proc           proc    defaults          0       0
PARTUUID=5de99f39-01  /boot/firmware  vfat    defaults          0       2
PARTUUID=5de99f39-02            /       ext4    noatime,nodiratime,defaults     0 1
UUID=CA187C14187C01AD  /srv/dev-disk-by-uuid-CA187C14187C01AD  ntfs-3g  defaults,big_writes,nofail,x-systemd.device-timeout=10  0  0
UUID=6a1529f2-6162-41cb-8b91-cf45542c71d4 /srv/dev-disk-by-uuid-6a1529f2-6162-41cb-8b91-cf45542c71d4 ext4 defaults,nofail,user_xattr,usrquota,grpquota,acl 0 0
# >>> [openmediavault]
/dev/disk/by-uuid/CA187C14187C01AD              /srv/dev-disk-by-uuid-CA187C14187C01AD  ntfs    defaults,nofail,big_writes      0 2
/dev/disk/by-uuid/e98ac4e5-f471-4de3-b907-b6454d38a705          /srv/dev-disk-by-uuid-e98ac4e5-f471-4de3-b907-b6454d38a705      ext4    defaults,nofail,user_xattr,usrquota,grpquota,acl        0 2
# <<< [openmediavault]
```
Guardar y probar:
```
sudo mount -a
```

### 5. Verificar
```
df -h
```
El disco debería aparecer montado.
---
### Nota

En sistemas NAS basados en Linux (como OpenMediaVault) es recomendable usar sistemas de archivos nativos como:

- ext4
- btrfs

pero NTFS y exFAT funcionan correctamente si se montan manualmente.

---

Cuando el disco está montado manualmente, **Docker puede no tener permisos**. En ese caso hay que hacer:

```bash
sudo chown -R 1000:1000 /mnt/disk1
```
Esto evita problemas con Plex, FileBrowser o rclone.












# Resultado final

Dashboard principal:

```
http://raspberry
```

Servicios:

```
Homepage
OpenMediaVault
Portainer
Plex
FileBrowser
Docker
Glances
SFTP mount (rclone)
```

---

# Posibles mejoras

* Reverse proxy (Nginx Proxy Manager)
* Uptime Kuma para monitorización
* Sonarr / Radarr para descargas automáticas
* Autenticación externa

---

# Licencia

MIT
