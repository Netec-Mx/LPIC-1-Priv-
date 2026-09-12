# Implementar un servidor Linux desde cero — Diseño de almacenamiento y sistemas de archivos empresariales

## 1. Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 105 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## 2. Descripción General

En este laboratorio implementarás el servidor **srv-linux-02** desde cero, partiendo de la creación de la máquina virtual en Azure hasta obtener un sistema Ubuntu Server 22.04.4 LTS o superior completamente operativo con un esquema de particionado GPT empresarial. Diseñarás manualmente las particiones utilizando `gdisk`, crearás sistemas de archivos optimizados para cada carga de trabajo (ext4 para sistema, xfs para logs y aplicaciones, swap para memoria virtual), configurarás el montaje persistente mediante `/etc/fstab` con UUIDs, y practicarás la verificación y reparación de sistemas de archivos. El resultado será la base de almacenamiento sobre la que se construirán los laboratorios posteriores de LVM y administración avanzada.

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Planificar e implementar un esquema de particionado GPT completo para un servidor empresarial con siete particiones diferenciadas por función.
- [ ] Crear y configurar sistemas de archivos ext4, xfs y swap optimizados para diferentes patrones de carga de trabajo.
- [ ] Configurar el montaje persistente de todos los sistemas de archivos mediante `/etc/fstab` utilizando UUIDs y opciones de montaje apropiadas.
- [ ] Verificar la integridad de sistemas de archivos y reparar errores simulados utilizando `fsck`, `e2fsck` y `xfs_repair`.

## 4. Prerrequisitos

### Conocimientos previos
- Familiaridad con la interfaz de línea de comandos de Linux (Lab 01 completado).
- Comprensión de los conceptos de dispositivos de bloque, particiones y sistemas de archivos.
- Conocimiento básico de la diferencia entre MBR y GPT (Lección 3.2).
- Entendimiento del proceso de planeación de instalaciones Linux (Lección 3.1).

### Acceso y recursos necesarios
- Acceso a HyperV Server Administration instalado y funcional en el equipo anfitrión.
- ISO de Ubuntu Server 22.04.4 LTS o superior descargada (`ubuntu-22.04.4-live-server-amd64.iso` o similar).
- Al menos 90 GB de espacio libre en disco del anfitrión para los discos virtuales.
- Acceso administrativo al equipo anfitrión para crear y configurar máquinas virtuales.
- Red interna `syslab-network` ya creada en Azure (del Lab 01).

## 5. Entorno del Laboratorio

### Especificaciones de la máquina virtual srv-linux-02

| Componente | Configuración |
|------------|---------------|
| **Nombre VM** | srv-linux-02 |
| **Sistema operativo** | Ubuntu Server 22.04.4 LTS o superior (64-bit) |
| **RAM** | 8192 MB |
| **CPUs** | 2 mínimo |
| **Disco 1 (/dev/sda)** | 60 GB — VDI, dinámico — Sistema operativo |
| **Disco 2 (/dev/sdb)** | 20 GB — VDI, dinámico — Datos |
| **Disco 3 (/dev/sdc)** | 20 GB — VDI, dinámico — Respaldo |
| **Adaptador de red 1** | Red interna: `syslab-network` |
| **Firmware** | EFI habilitado |
| **Controlador de disco** | SATA (AHCI) |

### Esquema de particionado objetivo para /dev/sda (60 GB)

| Partición | Punto de montaje | Tamaño | Sistema de archivos | Justificación |
|-----------|-----------------|--------|--------------------:|---------------|
| /dev/sda1 | /boot/efi | 512 MB | FAT32 (vfat) | Partición EFI requerida por UEFI |
| /dev/sda2 | /boot | 1 GB | ext4 | Kernels e initramfs fuera de LVM |
| /dev/sda3 | / | 20 GB | ext4 | Sistema raíz estable |
| /dev/sda4 | /var | 15 GB | xfs | Logs con escritura secuencial intensiva |
| /dev/sda5 | /home | 10 GB | ext4 | Datos de usuarios separados |
| /dev/sda6 | /opt | 10 GB | xfs | Aplicaciones empresariales |
| /dev/sda7 | swap | 4 GB | swap | Memoria virtual |

### En Azure crear la VM y los recursos necesarios segun se especifica.

```bash
Con el administrador de Hiper-V de Windows Server, verificar que esten creado los recursos.

# Una VM con 8GB RAM, 3 discos duros: Uno de 60Gb (disco del SO) y dos discos adicionales de 20Gb.
# En Azure con Windows Server y Hyper-V Manager el ambiente ya esta preparado y listo para usarse.

```

> **Nota:** Ajusta `/ruta/a/ubuntu-22.04.4-live-server-amd64.iso` a la ubicación real de tu ISO.

## 6. Procedimiento Paso a Paso

---

### Paso 1: Instalar Ubuntu Server con particionado mínimo temporal

**Objetivo:** Realizar la instalación base de Ubuntu Server 22.04.4 LTS o superior utilizando el instalador interactivo con un esquema de particionado mínimo que luego reconfiguraremos manualmente.

**Instrucciones:**

1. Inicia la VM **`srv-linux-02` desde HyperV-Manager**:
   ```bash
   Elegir la máquina indicada con le boton del mouse derecho y dar "connect" y después "start" para iniciarla.
   ```

2. En el menú GRUB del instalador, selecciona **"Try or Install Ubuntu Server"**.

3. Selecciona el idioma **English** (el sistema se administrará en inglés para consistencia con los labs).

   Elegir despues [contiue without updating]
   Identify Keyboard [Spanish LatinAmerica] [Done]
   Base installation (Ubuntu Server) [Done]

5. En la pantalla de configuración de red, configura la interfaz `eth0` manualmente:
   - Subnet: `192.168.100.0/24`
   - Address (manual/privada): `192.168.100.20/24`
   - Gateway: `192.168.100.1`
   - Name servers: `8.8.8.8` [Done]
   - Proxy Address: [Enter] [Done]
   - Mirror Address: [Borrar la URL] [Done]
   - Configure a custom storage layout (X) Custom storage layout
   - Select in Available Devices [Local Disk 60 GB]
   - Bajar donde dice "free space" y luego [Add GPT Partition]
   - Selecciona `/dev/sda` → "Add GPT Partition Table"
   - Crea partición 1: Tamaño 512M, formato fat32, montaje `/boot/efi`
   - Crea partición 2: Tamaño 1G, formato ext4, montaje `/boot`
   - Crea partición 3: Tamaño restante (~58.5G), formato ext4, montaje `/` [Done]
   - Para seguir la instalación elegir [Continue]

   > **Importante:** Este esquema temporal nos permite instalar el sistema base. Luego reparticionaremos manualmente para lograr el esquema empresarial objetivo.

8. Configura el perfil del servidor:
   - Your name: `System Administrator`
   - Your server's name: `srv-linux-02`
   - Pick a username: `sysadmin`
   - Password: `Raiz1234`
   - (X) Skip Ubuntu-PRO [Continue]
     
9. En la pantalla de SSH, marca **"Install OpenSSH server"**.

10. No selecciones ningún snap adicional. Procede con la instalación.

11. Cuando la instalación finalice, selecciona **"Reboot Now"**. Retira la ISO virtual cuando se solicite (o presiona Enter si ya se desconectó automáticamente).

**Salida esperada:**

Tras el reinicio, deberías ver el prompt de login:
```
srv-linux-02 login: _
```

**Verificación:**

```bash
# Inicia sesión con sysadmin
# Verifica el nombre de host y la versión del sistema
hostnamectl
```

Salida esperada:
```
 Static hostname: srv-linux-02
       Icon name: computer-vm
         Chassis: vm
      Machine ID: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
         Boot ID: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  Operating System: Ubuntu 22.04.4 LTS
            Kernel: Linux 5.15.0-xxx-generic
      Architecture: x86-64
```

---

### Paso 2: Preparar el disco /dev/sda con el esquema GPT empresarial

**Objetivo:** Reconfigurar el disco principal con el esquema de siete particiones GPT diseñado para operación empresarial, utilizando un enfoque de reinstalación limpia con las herramientas del entorno live.

> **Nota estratégica:** Dado que el instalador de Ubuntu no permite crear más de unas pocas particiones con la granularidad que necesitamos, realizaremos el particionado manual arrancando desde la ISO en modo live. Alternativamente, si ya completaste el Paso 1 y tienes el sistema instalado, puedes hacer una reinstalación aprovechando el conocimiento del esquema. En este paso documentamos el enfoque con la ISO live.

**Instrucciones:**

1. Apaga la VM:
   ```bash
   sudo shutdown -h now
   ```

2. Reconecta la ISO al drive virtual desde VirtualBox (o vía línea de comandos):
   ```bash
   VBoxManage storageattach "srv-linux-02" --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium "/ruta/a/ubuntu-22.04.4-live-server-amd64.iso"
   ```

3. Cambia el orden de arranque para iniciar desde DVD:
   ```bash
   VBoxManage modifyvm "srv-linux-02" --boot1 dvd --boot2 disk
   ```

4. Inicia la VM:
   ```bash
   VBoxManage startvm "srv-linux-02" --type gui
   ```

5. En el menú GRUB del instalador, selecciona **"Try or Install Ubuntu Server"**. Cuando aparezca la primera pantalla del instalador (selección de idioma), presiona **Ctrl+Alt+F2** (o en VirtualBox: Host+F2) para acceder a una consola shell.

   > Si no puedes acceder a un shell con Ctrl+Alt+F2, selecciona el idioma y avanza hasta la pantalla de "Storage configuration". Desde ahí puedes seleccionar "Shell" si el instalador lo ofrece, o bien usa el enfoque alternativo del Paso 2b.

6. Una vez en el shell, verifica los discos disponibles:
   ```bash
   lsblk
   ```

   Salida esperada:
   ```
   NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
   sda      8:0    0    60G  0 disk
   sdb      8:16   0    20G  0 disk
   sdc      8:32   0    20G  0 disk
   sr0     11:0    1   1.8G  0 rom
   ```

7. Elimina cualquier tabla de particiones existente y crea una nueva tabla GPT con `gdisk`:
   ```bash
   sudo gdisk /dev/sda
   ```

8. Dentro de `gdisk`, ejecuta los siguientes comandos interactivos:

   ```
   Command (? for help): o
   This option deletes all partitions and creates a new protective MBR.
   Proceed? (Y/N): Y
   ```

9. Crea la **Partición 1** — EFI System Partition (512 MB):
   ```
   Command (? for help): n
   Partition number (1-128, default 1): 1
   First sector (34-125829086, default = 2048) or {+-}size{KMGTP}: [Enter]
   Last sector (2048-125829086) or {+-}size{KMGTP}: +512M
   Current type is 8300 (Linux filesystem)
   Hex code or GUID (L to show codes, Enter = 8300): EF00
   ```

10. Crea la **Partición 2** — /boot (1 GB, ext4):
    ```
    Command (? for help): n
    Partition number (2-128, default 2): 2
    First sector (default): [Enter]
    Last sector or {+-}size: +1G
    Hex code or GUID: 8300
    ```

11. Crea la **Partición 3** — / raíz (20 GB, ext4):
    ```
    Command (? for help): n
    Partition number (3-128, default 3): 3
    First sector (default): [Enter]
    Last sector or {+-}size: +20G
    Hex code or GUID: 8300
    ```

12. Crea la **Partición 4** — /var (15 GB, xfs):
    ```
    Command (? for help): n
    Partition number (4-128, default 4): 4
    First sector (default): [Enter]
    Last sector or {+-}size: +15G
    Hex code or GUID: 8300
    ```

13. Crea la **Partición 5** — /home (10 GB, ext4):
    ```
    Command (? for help): n
    Partition number (5-128, default 5): 5
    First sector (default): [Enter]
    Last sector or {+-}size: +10G
    Hex code or GUID: 8300
    ```

14. Crea la **Partición 6** — /opt (10 GB, xfs):
    ```
    Command (? for help): n
    Partition number (6-128, default 6): 6
    First sector (default): [Enter]
    Last sector or {+-}size: +10G
    Hex code or GUID: 8300
    ```

15. Crea la **Partición 7** — swap (4 GB):
    ```
    Command (? for help): n
    Partition number (7-128, default 7): 7
    First sector (default): [Enter]
    Last sector or {+-}size: +4G
    Hex code or GUID: 8200
    ```

16. Verifica la tabla de particiones:
    ```
    Command (? for help): p
    ```

    **Salida esperada:**
    ```
    Number  Start (sector)    End (sector)  Size       Code  Name
       1            2048         1050623   512.0 MiB   EF00  EFI system partition
       2         1050624         3147775   1024.0 MiB  8300  Linux filesystem
       3         3147776        45090815   20.0 GiB    8300  Linux filesystem
       4        45090816        76546047   15.0 GiB    8300  Linux filesystem
       5        76546048        97517567   10.0 GiB    8300  Linux filesystem
       6        97517568       118489087   10.0 GiB    8300  Linux filesystem
       7       118489088       126877695   4.0 GiB     8200  Linux swap
    ```

17. Escribe la tabla al disco y sal:
    ```
    Command (? for help): w
    Do you want to proceed? (Y/N): Y
    ```

**Verificación:**

```bash
sudo lsblk /dev/sda
```

Salida esperada:
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   60G  0 disk
├─sda1   8:1    0  512M  0 part
├─sda2   8:2    0    1G  0 part
├─sda3   8:3    0   20G  0 part
├─sda4   8:4    0   15G  0 part
├─sda5   8:5    0   10G  0 part
├─sda6   8:6    0   10G  0 part
└─sda7   8:7    0    4G  0 part
```

---

### Paso 3: Crear los sistemas de archivos

**Objetivo:** Formatear cada partición con el sistema de archivos apropiado según su función, aplicando opciones de optimización.

**Instrucciones:**

1. Crea el sistema de archivos FAT32 para la partición EFI:
   ```bash
   sudo mkfs.vfat -F 32 -n EFI /dev/sda1
   ```

2. Crea el sistema de archivos ext4 para `/boot`:
   ```bash
   sudo mkfs.ext4 -L boot /dev/sda2
   ```

3. Crea el sistema de archivos ext4 para `/` (raíz) con reserva del 5% para root:
   ```bash
   sudo mkfs.ext4 -L root /dev/sda3
   ```

4. Crea el sistema de archivos XFS para `/var` (optimizado para logs):
   ```bash
   sudo mkfs.xfs -L var /dev/sda4
   ```

   > **Nota:** XFS es ideal para `/var` porque maneja eficientemente escrituras secuenciales grandes (logs) y soporta cuotas a nivel de directorio.

5. Crea el sistema de archivos ext4 para `/home`:
   ```bash
   sudo mkfs.ext4 -L home /dev/sda5
   ```

6. Crea el sistema de archivos XFS para `/opt`:
   ```bash
   sudo mkfs.xfs -L opt /dev/sda6
   ```

7. Crea el espacio swap:
   ```bash
   sudo mkswap -L swap /dev/sda7
   ```

**Salida esperada (ejemplo para mkfs.ext4 en /dev/sda3):**
```
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 5242880 4k blocks and 1310720 inodes
Filesystem UUID: a1b2c3d4-e5f6-7890-abcd-ef1234567890
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

**Verificación:**

```bash
sudo blkid /dev/sda*
```

Salida esperada (los UUIDs variarán):
```
/dev/sda1: LABEL_FATBOOT="EFI" LABEL="EFI" UUID="ABCD-1234" TYPE="vfat" PARTLABEL="EFI system partition" PARTUUID="..."
/dev/sda2: LABEL="boot" UUID="11111111-2222-3333-4444-555555555555" TYPE="ext4" PARTUUID="..."
/dev/sda3: LABEL="root" UUID="aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee" TYPE="ext4" PARTUUID="..."
/dev/sda4: LABEL="var" UUID="ffffffff-1111-2222-3333-444444444444" TYPE="xfs" PARTUUID="..."
/dev/sda5: LABEL="home" UUID="55555555-6666-7777-8888-999999999999" TYPE="ext4" PARTUUID="..."
/dev/sda6: LABEL="opt" UUID="aaaaaaaa-1111-2222-3333-bbbbbbbbbbbb" TYPE="xfs" PARTUUID="..."
/dev/sda7: LABEL="swap" UUID="cccccccc-dddd-eeee-ffff-000000000000" TYPE="swap" PARTUUID="..."
```

---

### Paso 4: Instalar Ubuntu Server en el esquema de particiones creado

**Objetivo:** Completar la instalación de Ubuntu Server 22.04.4 LTS o superior utilizando el esquema de particiones GPT empresarial recién creado.

**Instrucciones:**

1. Regresa al instalador de Ubuntu (Ctrl+Alt+F1 o reinicia la VM con la ISO):
   ```bash
   # Si estás en el shell, puedes reiniciar:
   sudo reboot
   ```

2. En el instalador, avanza hasta la pantalla **"Storage configuration"** y selecciona **"Custom storage layout"**.

3. Asigna los puntos de montaje a las particiones existentes (el instalador debería mostrar las particiones ya creadas en /dev/sda):

   | Partición | Formato | Montaje |
   |-----------|---------|---------|
   | /dev/sda1 | fat32 (no reformatear) | /boot/efi |
   | /dev/sda2 | ext4 (no reformatear) | /boot |
   | /dev/sda3 | ext4 (no reformatear) | / |
   | /dev/sda4 | xfs (no reformatear) | /var |
   | /dev/sda5 | ext4 (no reformatear) | /home |
   | /dev/sda6 | xfs (no reformatear) | /opt |
   | /dev/sda7 | swap | swap |

   > **Importante:** Si el instalador insiste en reformatear, permite el formateo ya que usará los mismos tipos de sistema de archivos que configuramos. Lo crítico es que los puntos de montaje sean correctos.

4. Configura el perfil del servidor:
   - Server name: `srv-linux-02`
   - Username: `sysadmin`
   - Password: `Raiz1234`

5. Habilita **OpenSSH server** en la pantalla de servicios.

6. Completa la instalación y reinicia.

7. Tras el reinicio, retira la ISO del drive virtual:
   ```bash
   VBoxManage storageattach "srv-linux-02" --storagectl "IDE" --port 0 --device 0 --type dvddrive --medium emptydrive
   ```

8. Cambia el orden de arranque:
   ```bash
   VBoxManage modifyvm "srv-linux-02" --boot1 disk --boot2 none
   ```

**Verificación:**

Inicia sesión como `sysadmin` y ejecuta:

```bash
lsblk -f /dev/sda
```

Salida esperada:
```
NAME   FSTYPE FSVER LABEL UUID                                 MOUNTPOINTS
sda
├─sda1 vfat   FAT32 EFI   ABCD-1234                            /boot/efi
├─sda2 ext4   1.0   boot  11111111-2222-3333-4444-555555555555  /boot
├─sda3 ext4   1.0   root  aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee  /
├─sda4 xfs          var   ffffffff-1111-2222-3333-444444444444  /var
├─sda5 ext4   1.0   home  55555555-6666-7777-8888-999999999999  /home
├─sda6 xfs          opt   aaaaaaaa-1111-2222-3333-bbbbbbbbbbbb  /opt
└─sda7 swap   1     swap  cccccccc-dddd-eeee-ffff-000000000000  [SWAP]
```

```bash
df -hT
```

Salida esperada (aproximada):
```
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/sda3      ext4       20G  3.2G   16G  17% /
/dev/sda2      ext4      974M  130M  777M  15% /boot
/dev/sda1      vfat      511M  5.3M  506M   2% /boot/efi
/dev/sda4      xfs        15G   95M   15G   1% /var
/dev/sda5      ext4      9.8G   24K  9.3G   1% /home
/dev/sda6      xfs        10G  104M  9.9G   2% /opt
```

```bash
free -h
```

Debe mostrar ~4 GB de swap:
```
               total        used        free
Swap:          4.0Gi          0B       4.0Gi
```

---

### Paso 5: Configurar /etc/fstab con UUIDs y opciones de montaje optimizadas

**Objetivo:** Verificar y optimizar la configuración de montaje persistente en `/etc/fstab`, asegurando que se usen UUIDs y opciones de montaje apropiadas para cada sistema de archivos.

**Instrucciones:**

1. Obtén los UUIDs actuales de todas las particiones:
   ```bash
   sudo blkid /dev/sda1 /dev/sda2 /dev/sda3 /dev/sda4 /dev/sda5 /dev/sda6 /dev/sda7
   ```

2. Crea un respaldo del fstab actual:
   ```bash
   sudo cp /etc/fstab /etc/fstab.bak
   ```

3. Edita `/etc/fstab` para asegurar que todas las entradas usen UUIDs y opciones optimizadas:
   ```bash
   sudo nano /etc/fstab
   ```

4. El archivo debe quedar con el siguiente formato (reemplaza los UUIDs con los reales de tu sistema):
   ```
   # /etc/fstab - srv-linux-02 - Esquema empresarial GPT
   # <file system>                            <mount point>  <type>  <options>                    <dump> <pass>
   UUID=ABCD-1234                             /boot/efi      vfat    umask=0077                   0      1
   UUID=11111111-2222-3333-4444-555555555555  /boot          ext4    defaults                     0      2
   UUID=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee  /              ext4    defaults,errors=remount-ro   0      1
   UUID=ffffffff-1111-2222-3333-444444444444  /var           xfs     defaults,noatime             0      2
   UUID=55555555-6666-7777-8888-999999999999  /home          ext4    defaults,nosuid,nodev        0      2
   UUID=aaaaaaaa-1111-2222-3333-bbbbbbbbbbbb  /opt           xfs     defaults,noatime             0      2
   UUID=cccccccc-dddd-eeee-ffff-000000000000  none           swap    sw                           0      0
   ```

   **Explicación de las opciones:**
   - `errors=remount-ro` en `/`: Si se detectan errores, remonta como solo lectura para proteger datos.
   - `noatime` en `/var` y `/opt`: No actualiza el tiempo de acceso en cada lectura, mejorando rendimiento de E/S.
   - `nosuid,nodev` en `/home`: Previene la ejecución de binarios con SUID y la creación de dispositivos especiales por usuarios regulares.
   - `umask=0077` en EFI: Restringe permisos de la partición EFI solo a root.

5. Verifica la sintaxis del fstab sin reiniciar:
   ```bash
   sudo findmnt --verify
   ```

   Si no hay errores, el comando no produce salida (o muestra solo advertencias informativas).

6. Prueba que todas las entradas se montan correctamente:
   ```bash
   sudo umount /home
   sudo mount -a
   ```

   > Si `umount /home` falla porque está en uso, puedes verificar con `mount -a` directamente (solo intentará montar lo que no esté montado).

**Verificación:**

```bash
# Verificar que todas las particiones están montadas con las opciones correctas
mount | grep -E "sda[2-7]"
```

Salida esperada:
```
/dev/sda3 on / type ext4 (rw,relatime,errors=remount-ro)
/dev/sda2 on /boot type ext4 (rw,relatime)
/dev/sda4 on /var type xfs (rw,noatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/sda5 on /home type ext4 (rw,nosuid,nodev,relatime)
/dev/sda6 on /opt type xfs (rw,noatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
```

```bash
# Verificar swap activo
swapon --show
```

Salida esperada:
```
NAME      TYPE      SIZE USED PRIO
/dev/sda7 partition   4G   0B   -2
```

---

### Paso 6: Verificar la integridad de los sistemas de archivos

**Objetivo:** Utilizar las herramientas `tune2fs`, `dumpe2fs` y `xfs_info` para inspeccionar los metadatos y la salud de los sistemas de archivos creados.

**Instrucciones:**

1. Inspecciona los parámetros del sistema de archivos raíz (ext4):
   ```bash
   sudo tune2fs -l /dev/sda3 | grep -E "(Filesystem state|Block count|Reserved block count|Filesystem created|Mount count|Check interval)"
   ```

   **Salida esperada:**
   ```
   Filesystem state:         clean
   Block count:              5242880
   Reserved block count:     262144
   Filesystem created:       [fecha de creación]
   Mount count:              1
   Check interval:           0 (<none>)
   ```

2. Verifica la información del sistema de archivos XFS en `/var`:
   ```bash
   sudo xfs_info /var
   ```

   **Salida esperada:**
   ```
   meta-data=/dev/sda4              isize=512    agcount=4, agsize=983040 blks
            =                       sectsz=512   attr=2, projid32bit=1
            =                       crc=1        finobt=1, sparse=1, rmapbt=0
            =                       reflink=1    bigtime=1 inobtcount=1
   data     =                       bsize=4096   blocks=3932160, imaxpct=25
            =                       sunit=0      swidth=0 blks
   naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
   log      =internal log           bsize=4096   blocks=16384, version=2
            =                       sectsz=512   sunit=0 blks, lazy-count=1
   realtime =none                   extsz=4096   blocks=0, rtextents=0
   ```

3. Verifica el espacio reservado para root en la partición `/home`:
   ```bash
   sudo tune2fs -l /dev/sda5 | grep "Reserved block count"
   ```

4. Reduce el espacio reservado en `/home` al 1% (los usuarios no necesitan el 5% predeterminado):
   ```bash
   sudo tune2fs -m 1 /dev/sda5
   ```

   **Salida esperada:**
   ```
   Setting reserved blocks percentage to 1% (26214 blocks)
   ```

5. Verifica el cambio:
   ```bash
   sudo tune2fs -l /dev/sda5 | grep "Reserved block count"
   ```

   Salida esperada:
   ```
   Reserved block count:     26214
   ```

**Verificación:**

```bash
# Estado general de todos los sistemas de archivos montados
df -hT --output=source,fstype,size,used,avail,pcent,target | grep sda
```

---

### Paso 7: Simular y reparar errores en sistemas de archivos

**Objetivo:** Practicar la detección y reparación de errores en sistemas de archivos ext4 y xfs utilizando `fsck`, `e2fsck` y `xfs_repair` en modo offline.

**Instrucciones:**

#### 7a: Reparación de sistema de archivos ext4 (/home)

1. Desmonta la partición `/home` (asegúrate de que ningún proceso la esté usando):
   ```bash
   # Verifica si hay procesos usando /home
   sudo lsof +D /home 2>/dev/null
   
   # Desmonta
   sudo umount /home
   ```

2. Ejecuta una verificación de solo lectura con `fsck`:
   ```bash
   sudo fsck -n /dev/sda5
   ```

   **Salida esperada (sistema limpio):**
   ```
   fsck from util-linux 2.37.2
   e2fsck 1.46.5 (30-Dec-2021)
   home: clean, 11/655360 files, 66753/2621440 blocks
   ```

3. Simula una corrupción menor escribiendo datos aleatorios en una zona del disco (¡SOLO en entorno de laboratorio!):
   ```bash
   # Escribir 4KB de datos aleatorios en un offset que no destruya el superbloque principal
   # El superbloque principal está en el bloque 0-1, evitamos esa zona
   sudo dd if=/dev/urandom of=/dev/sda5 bs=4096 count=1 seek=50000 conv=notrunc
   ```

4. Ejecuta `e2fsck` para detectar el error:
   ```bash
   sudo e2fsck -n /dev/sda5
   ```

   **Salida esperada (puede variar):**
   ```
   e2fsck 1.46.5 (30-Dec-2021)
   home: recovering journal
   home contains a filesystem with errors, check forced.
   ...
   home: *** FILE SYSTEM WAS MODIFIED ***
   home: 11/655360 files, 66753/2621440 blocks
   ```

5. Repara el sistema de archivos automáticamente:
   ```bash
   sudo e2fsck -y /dev/sda5
   ```

   **Salida esperada:**
   ```
   e2fsck 1.46.5 (30-Dec-2021)
   Pass 1: Checking inodes, blocks, and sizes
   Pass 2: Checking directory structure
   Pass 3: Checking directory connectivity
   Pass 4: Checking reference counts
   Pass 5: Checking group summary information
   home: 11/655360 files, 66753/2621440 blocks
   ```

6. Verifica que el sistema de archivos esté limpio:
   ```bash
   sudo e2fsck -n /dev/sda5
   ```

   **Salida esperada:**
   ```
   home: clean, 11/655360 files, 66753/2621440 blocks
   ```

7. Vuelve a montar `/home`:
   ```bash
   sudo mount /home
   ```

#### 7b: Verificación de sistema de archivos XFS (/opt)

1. Desmonta `/opt`:
   ```bash
   sudo umount /opt
   ```

2. Ejecuta `xfs_repair` en modo verificación (dry-run):
   ```bash
   sudo xfs_repair -n /dev/sda6
   ```

   **Salida esperada (sistema limpio):**
   ```
   Phase 1 - find and verify superblock...
   Phase 2 - using internal log
           - zero log...
           - scan filesystem freespace and inode maps...
           - found root inode chunk
   Phase 3 - for each AG...
           - scan and clear agi unlinked lists...
           - process known inodes and perform inode discovery...
           - agno = 0
           - agno = 1
           - agno = 2
           - agno = 3
   Phase 4 - check for duplicate blocks...
           - setting up duplicate extent list...
           - check for inodes claiming duplicate blocks...
           - agno = 0
           - agno = 1
           - agno = 2
           - agno = 3
   Phase 5 - rebuild AG headers and trees...
           - reset superblock...
   Phase 6 - check inode connectivity...
           - resetting contents of realtime bitmap and summary inodes
           - traversing filesystem ...
           - traversal finished ...
           - moving disconnected inodes to lost+found ...
   Phase 7 - verify and correct link counts...
   No modify flag set, skipping filesystem flush and target.
   ```

3. Ejecuta la reparación real (aunque no haya errores, es buena práctica verificar):
   ```bash
   sudo xfs_repair /dev/sda6
   ```

4. Vuelve a montar `/opt`:
   ```bash
   sudo mount /opt
   ```

   > **Importante:** `xfs_repair` NUNCA debe ejecutarse en un sistema de archivos montado. A diferencia de `fsck` que puede verificar en modo lectura sobre un FS montado (con riesgos), `xfs_repair` requiere desmontaje obligatorio.

**Verificación:**

```bash
# Confirmar que todos los sistemas de archivos están montados correctamente
mount | grep sda
echo "---"
# Verificar que no hay errores en dmesg
sudo dmesg | grep -i "error\|corrupt\|fail" | grep -i "sda" | tail -5
```

---

### Paso 8: Configurar la red y finalizar la preparación del servidor

**Objetivo:** Configurar la dirección IP estática definitiva de srv-linux-02 y verificar la conectividad dentro de la red del laboratorio.

**Instrucciones:**

1. Identifica la interfaz de red:
   ```bash
   ip link show
   ```

   La interfaz principal será `enp0s3` (o similar).

2. Configura la IP estática usando Netplan:
   ```bash
   sudo nano /etc/netplan/00-installer-config.yaml
   ```

3. Escribe la siguiente configuración:
   ```yaml
   network:
     version: 2
     ethernets:
       enp0s3:
         addresses:
           - 192.168.100.20/24
         routes:
           - to: default
             via: 192.168.100.1
         nameservers:
           addresses:
             - 8.8.8.8
             - 8.8.4.4
   ```

4. Aplica la configuración:
   ```bash
   sudo netplan apply
   ```

5. Verifica la IP asignada:
   ```bash
   ip addr show enp0s3
   ```

   **Salida esperada:**
   ```
   2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
       link/ether 08:00:27:xx:xx:xx brd ff:ff:ff:ff:ff:ff
       inet 192.168.100.20/24 brd 192.168.100.255 scope global enp0s3
          valid_lft forever preferred_lft forever
   ```

6. Si srv-linux-01 está activo, verifica la conectividad:
   ```bash
   ping -c 3 192.168.100.10
   ```

7. Crea los directorios de trabajo estándar del curso:
   ```bash
   sudo mkdir -p /opt/sysreport /opt/scripts /opt/backups /opt/appdata
   sudo chown -R sysadmin:sysadmin /opt/sysreport /opt/scripts /opt/backups /opt/appdata
   ```

8. Habilita la contraseña de root (requerida por algunos labs posteriores):
   ```bash
   echo "root:R00t@Linux2024!" | sudo chpasswd
   ```

**Verificación:**

```bash
# Resumen completo del servidor
echo "=== HOSTNAME ==="
hostname
echo ""
echo "=== IP ADDRESS ==="
ip -4 addr show enp0s3 | grep inet
echo ""
echo "=== DISK LAYOUT ==="
lsblk -f /dev/sda
echo ""
echo "=== FILESYSTEM USAGE ==="
df -hT | grep -v tmpfs
echo ""
echo "=== SWAP ==="
swapon --show
echo ""
echo "=== WORK DIRECTORIES ==="
ls -la /opt/
```

---

## 7. Validación y Pruebas Finales (OPCIONAL).

Ejecuta el siguiente script de validación para confirmar que todos los objetivos del laboratorio se han cumplido:

```bash
#!/bin/bash
# Script de validación - Lab 03-00-01
# Ejecutar como root: sudo bash validate_lab03.sh

PASS=0
FAIL=0

check() {
    if eval "$2" > /dev/null 2>&1; then
        echo "[PASS] $1"
        ((PASS++))
    else
        echo "[FAIL] $1"
        ((FAIL++))
    fi
}

echo "============================================"
echo " VALIDACIÓN LAB 03-00-01 - srv-linux-02"
echo "============================================"
echo ""

# 1. Verificar tabla GPT
check "Tabla de particiones GPT en /dev/sda" \
    "sudo gdisk -l /dev/sda 2>/dev/null | grep -q 'GPT'"

# 2. Verificar 7 particiones
check "7 particiones en /dev/sda" \
    "[ $(lsblk -ln /dev/sda | grep -c part) -eq 7 ]"

# 3. Partición EFI (FAT32)
check "Partición EFI con FAT32" \
    "blkid /dev/sda1 | grep -q 'TYPE=\"vfat\"'"

# 4. /boot con ext4
check "/boot montado con ext4" \
    "mount | grep '/boot ' | grep -q 'ext4'"

# 5. / con ext4
check "/ montado con ext4" \
    "mount | grep 'on / ' | grep -q 'ext4'"

# 6. /var con xfs
check "/var montado con xfs" \
    "mount | grep '/var ' | grep -q 'xfs'"

# 7. /home con ext4
check "/home montado con ext4" \
    "mount | grep '/home ' | grep -q 'ext4'"

# 8. /opt con xfs
check "/opt montado con xfs" \
    "mount | grep '/opt ' | grep -q 'xfs'"

# 9. Swap activo
check "Swap activo (~4G)" \
    "swapon --show | grep -q '/dev/sda7'"

# 10. fstab usa UUIDs
check "/etc/fstab usa UUIDs (no /dev/sdX)" \
    "grep -c '^UUID=' /etc/fstab | grep -qE '^[5-9]|^[1-9][0-9]'"

# 11. Opciones de montaje en /var (noatime)
check "/var montado con noatime" \
    "mount | grep '/var ' | grep -q 'noatime'"

# 12. Opciones de seguridad en /home (nosuid)
check "/home montado con nosuid" \
    "mount | grep '/home ' | grep -q 'nosuid'"

# 13. Hostname correcto
check "Hostname es srv-linux-02" \
    "[ \"$(hostname)\" = 'srv-linux-02' ]"

# 14. IP correcta
check "IP 192.168.100.20 configurada" \
    "ip addr show | grep -q '192.168.100.20'"

# 15. Directorios de trabajo creados
check "Directorios /opt/{sysreport,scripts,backups,appdata} existen" \
    "[ -d /opt/sysreport ] && [ -d /opt/scripts ] && [ -d /opt/backups ] && [ -d /opt/appdata ]"

echo ""
echo "============================================"
echo " RESULTADOS: $PASS aprobados, $FAIL fallidos"
echo "============================================"

if [ $FAIL -eq 0 ]; then
    echo " ✓ LABORATORIO COMPLETADO EXITOSAMENTE"
else
    echo " ✗ Revisa los items marcados como [FAIL]"
fi
```

Guarda y ejecuta:
```bash
sudo nano /opt/scripts/validate_lab03.sh
sudo bash /opt/scripts/validate_lab03.sh
```

**Resultado esperado:** 15/15 pruebas aprobadas.

---

## 8. Solución de Problemas

### Problema 1: El sistema no arranca después de reparticionar — "No bootable device"

**Síntomas:** Tras completar la instalación y reiniciar, la VM muestra el mensaje "No bootable device" o entra al EFI Shell en lugar de cargar GRUB.

**Causa:** La partición EFI (`/dev/sda1`) no fue correctamente registrada en el firmware UEFI, o el bootloader GRUB no se instaló en la ubicación esperada (`/boot/efi/EFI/ubuntu/`).

**Solución:**

1. Arranca desde la ISO de Ubuntu en modo live (shell).
2. Monta las particiones del sistema:
   ```bash
   sudo mount /dev/sda3 /mnt
   sudo mount /dev/sda2 /mnt/boot
   sudo mount /dev/sda1 /mnt/boot/efi
   sudo mount /dev/sda4 /mnt/var
   sudo mount --bind /dev /mnt/dev
   sudo mount --bind /proc /mnt/proc
   sudo mount --bind /sys /mnt/sys
   sudo mount --bind /sys/firmware/efi/efivars /mnt/sys/firmware/efi/efivars
   ```
3. Entra al chroot:
   ```bash
   sudo chroot /mnt
   ```
4. Reinstala GRUB:
   ```bash
   grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
   update-grub
   ```
5. Sal del chroot y reinicia:
   ```bash
   exit
   sudo umount -R /mnt
   sudo reboot
   ```

---

### Problema 2: La partición /var no se monta al arranque — "mount: wrong fs type"

**Síntomas:** Al arrancar, el sistema muestra un error similar a:
```
mount: /var: wrong fs type, bad option, bad superblock on /dev/sda4
```
El sistema entra en modo emergencia (emergency mode).

**Causa:** El UUID en `/etc/fstab` no coincide con el UUID real de la partición (ocurre si se reformateó la partición después de configurar fstab), o el paquete `xfsprogs` no está instalado y el kernel no puede montar XFS.

**Solución:**

1. En el modo emergencia, inicia sesión como root.
2. Verifica el UUID real:
   ```bash
   blkid /dev/sda4
   ```
3. Compara con el UUID en fstab:
   ```bash
   grep var /etc/fstab
   ```
4. Si los UUIDs no coinciden, corrige el fstab:
   ```bash
   # Obtener el UUID correcto
   NEW_UUID=$(blkid -s UUID -o value /dev/sda4)
   # Editar fstab
   nano /etc/fstab
   # Reemplazar el UUID antiguo de /var con $NEW_UUID
   ```
5. Si el problema es la falta de xfsprogs:
   ```bash
   # Montar / como lectura-escritura
   mount -o remount,rw /
   # Instalar xfsprogs (si hay red disponible)
   apt install xfsprogs
   ```
6. Reinicia:
   ```bash
   reboot
   ```

---

## 9. Limpieza

Este laboratorio **no requiere limpieza** ya que srv-linux-02 será utilizada en laboratorios posteriores (Lab 04 y Lab 06). La VM debe permanecer configurada y funcional.

Si necesitas liberar recursos temporalmente:
```bash
# Apagar la VM de forma ordenada (ejecutar dentro de srv-linux-02)
sudo shutdown -h now

# O desde el anfitrión:
VBoxManage controlvm "srv-linux-02" acpipowerbutton
```

Para restaurar el laboratorio desde cero (solo si es necesario):
```bash
# Eliminar la VM completamente (¡DESTRUCTIVO!)
VBoxManage unregistervm "srv-linux-02" --delete
```

---

## 10. Resumen

### Logros alcanzados

En este laboratorio has implementado un servidor Linux empresarial completo, aplicando los principios de planeación de instalaciones estudiados en la Lección 3.1:

| Objetivo | Herramientas utilizadas | Resultado |
|----------|------------------------|-----------|
| Particionado GPT empresarial | `gdisk`, `lsblk`, `blkid` | 7 particiones con roles específicos |
| Sistemas de archivos optimizados | `mkfs.ext4`, `mkfs.xfs`, `mkswap` | ext4 para estabilidad, xfs para rendimiento de E/S |
| Montaje persistente | `/etc/fstab`, `mount`, `findmnt` | UUIDs + opciones de seguridad y rendimiento |
| Verificación y reparación | `e2fsck`, `xfs_repair`, `tune2fs` | Capacidad de diagnosticar y corregir corrupción |

### Conceptos clave reforzados

- **Separación de particiones:** Aislar `/var`, `/home`, `/opt` y `/tmp` previene que un directorio lleno afecte la estabilidad del sistema completo.
- **Selección de sistema de archivos por carga:** XFS para escrituras secuenciales intensivas (logs), ext4 para cargas mixtas y estabilidad general.
- **UUIDs vs. nombres de dispositivo:** Los UUIDs garantizan montaje correcto independientemente del orden de detección de discos.
- **Opciones de montaje como capa de seguridad:** `nosuid`, `nodev`, `noexec` y `noatime` son controles de bajo costo con alto impacto.

### Próximos pasos

La VM srv-linux-02 con su esquema de almacenamiento será la base para:
- **Lab 04:** Configuración de discos adicionales `/dev/sdb` y `/dev/sdc` para datos y respaldos.
- **Lab 06:** Implementación de LVM sobre los discos `/dev/sdd`, `/dev/sde`, `/dev/sdf` y `/dev/sdg` para almacenamiento flexible y escalable.

### Recursos adicionales

- `man gdisk` — Documentación completa de GPT fdisk
- `man fstab` — Formato y opciones de /etc/fstab
- `man e2fsck` — Verificación de sistemas de archivos ext2/3/4
- `man xfs_repair` — Reparación de sistemas de archivos XFS
- [Ubuntu Server Guide - Storage](https://ubuntu.com/server/docs/device-mapper-multipathing-introduction)
- [Red Hat - XFS Administration Guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/index)
