# Análisis Completo de la Infraestructura Física y Lógica de un Servidor Linux

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 58 minutos |
| **Complejidad** | Fácil |
| **Nivel Bloom** | Aplicar |
| **Servidor objetivo** | srv-linux-01 (Ubuntu Server 22.04.4 LTS - el nombre del host puede ser diferente -) |
| **IP** | 192.168.100.10 (la IP puede ser diferente) |

## Descripción General

En este laboratorio realizarás un inventario exhaustivo del hardware, los módulos del kernel, los dispositivos y los recursos disponibles en el servidor `srv-linux-01`. Utilizarás herramientas de inspección del sistema como `lshw`, `lscpu`, `lspci`, `lsusb`, `lsblk`, `dmidecode`, `lsmod`, `modinfo` y los sistemas de archivos virtuales `/proc` y `/sys`. Al finalizar, generarás un reporte de diagnóstico inicial que servirá como línea base documentada para todos los laboratorios posteriores del curso.

## Objetivos de Aprendizaje

- [ ] Identificar y documentar los componentes de hardware del servidor utilizando herramientas de inspección del sistema (`lshw`, `lscpu`, `lspci`, `lsusb`, `lsblk`, `dmidecode`).
- [ ] Analizar la arquitectura del kernel Linux, sus módulos cargados y la interacción con los dispositivos de hardware mediante `lsmod`, `modinfo` y `/proc`.
- [ ] Examinar el proceso de inicialización del sistema revisando `dmesg` y `journalctl -b`.
- [ ] Generar un reporte de diagnóstico estructurado en `/opt/sysreport/` que documente el estado inicial del entorno.

## Prerrequisitos

### Conocimientos Previos

- Uso básico de la terminal Linux (navegación de directorios, ejecución de comandos, redirección de salida).
- Concepto de capas del sistema: hardware → kernel → espacio de usuario.
- Comprensión básica de los componentes de hardware: CPU, RAM, almacenamiento, buses.

### Acceso Requerido

- VM `srv-linux-01` encendida y accesible vía SSH o consola directa.
- Usuario `sysadmin` con acceso `sudo` configurado.
- Contraseña: `Linux@Admin2024!`

## Entorno del Laboratorio

### Hardware Virtual (srv-linux-01)

| Recurso | Especificación |
|---------|---------------|
| CPU | 4 vCPUs (x86_64) |
| RAM | 4 GB mínimo |
| Disco sistema | `/dev/sda` — 40 GB |
| Red Adaptador 1 | Red interna `syslab-network` (192.168.100.10/24) - Las IP´s configuradas pueden ser diferentes - |
| Red Adaptador 2 | NAT (acceso a internet) - Las IP´s configuradas pueden ser diferentes -|

### Software Requerido

| Herramienta | Versión | Paquete |
|-------------|---------|---------|
| lshw | 02.19.2 | `lshw` |
| lscpu | util-linux 2.37.2 | `util-linux` |
| lspci | pciutils 3.7.0 | `pciutils` |
| lsblk | util-linux 2.37.2 | `util-linux` |
| dmidecode | 3.3 | `dmidecode` |
| lsmod/modinfo | kmod 29 | `kmod` |
| journalctl | systemd 249.11 | `systemd` |
| htop | 3.2.1 | `htop` |
| tree | 2.0.2 | `tree` |

### Preparación Inicial del Entorno

Conéctate al servidor usando los datos que te de tu instructor e instala las herramientas necesarias:

```bash
ssh sysadmin@192.168.100.10 (los datos de conexión pueden ser diferentes)
o
Usa el cliente de RDP con los datos que te proporcione tu instructor.
```

```bash
sudo apt update && sudo apt install -y lshw pciutils usbutils dmidecode htop tree
```

Crea el directorio estándar para reportes:

```bash
sudo mkdir -p /opt/sysreport
sudo chown root:root /opt/sysreport
```

---

## Procedimiento Paso a Paso

### Paso 1: Inspección del Procesador (CPU)

**Objetivo:** Identificar la arquitectura, cantidad de núcleos, hilos, frecuencia y características de la CPU.

**Instrucciones:**

1. Ejecuta el comando `lscpu` para obtener información completa del procesador:

```bash
1.1 Cambiate al usuario `root` con el siguiente comando: sudo bash
1.2 Ejecuta los siguientes comandos que se te indican.

lscpu
```

2. Examina el archivo `/proc/cpuinfo` para obtener detalles por núcleo:

```bash
cat /proc/cpuinfo | head -30
```

3. Verifica las extensiones de virtualización disponibles:

```bash
grep -E '(vmx|svm)' /proc/cpuinfo | head -1
```

4. Guarda la información de CPU en un archivo temporal para el reporte:

```bash
lscpu > /opt/sysreport/cpu_info.txt
echo "---" >> /opt/sysreport/cpu_info.txt
echo "Flags de virtualización:" >> /opt/sysreport/cpu_info.txt
grep -cE '(vmx|svm)' /proc/cpuinfo >> /opt/sysreport/cpu_info.txt 2>/dev/null || echo "No detectadas" >> /opt/sysreport/cpu_info.txt
```

**Salida Esperada (ejemplo):**

```
Architecture:            x86_64
CPU op-mode(s):          32-bit, 64-bit
CPU(s):                  4
Thread(s) per core:      1
Core(s) per socket:      4
Socket(s):               1
Model name:              Intel(R) Core(TM) i7-... (o similar virtualizado)
L1d cache:               128 KiB
L1i cache:               128 KiB
L2 cache:                1 MiB
L3 cache:                ...
```

**Verificación:**

```bash
test -s /opt/sysreport/cpu_info.txt && echo "✓ Archivo CPU generado correctamente" || echo "✗ Error: archivo vacío"
```

---

### Paso 2: Inspección de la Memoria RAM

**Objetivo:** Determinar la cantidad de memoria instalada, su uso actual y la configuración de los módulos de memoria.

**Instrucciones:**

1. Consulta el resumen de memoria con `free`:

```bash
free -h
```

2. Obtén información detallada de los slots de memoria con `dmidecode`:

```bash
sudo dmidecode --type memory
```

3. Revisa `/proc/meminfo` para datos granulares del subsistema de memoria:

```bash
head -20 /proc/meminfo
```

4. Verifica si existe espacio de intercambio (swap) configurado:

```bash
swapon --show
```

5. Guarda la información de memoria:

```bash
{
  echo "=== RESUMEN DE MEMORIA ==="
  free -h
  echo ""
  echo "=== SWAP ==="
  swapon --show
  echo ""
  echo "=== DETALLES DMIDECODE ==="
  sudo dmidecode --type memory 2>/dev/null | grep -A5 "Memory Device" | head -40
} > /opt/sysreport/memory_info.txt
```

**Salida Esperada:**

```
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       512Mi       2.9Gi       1.0Mi       420Mi       3.1Gi
Swap:          2.0Gi          0B       2.0Gi
```

**Verificación:**

```bash
test -s /opt/sysreport/memory_info.txt && echo "✓ Archivo memoria generado" || echo "✗ Error"
```

---

### Paso 3: Inspección de Dispositivos de Almacenamiento

**Objetivo:** Identificar los discos, particiones, sistemas de archivos y puntos de montaje del servidor.

**Instrucciones:**

1. Lista los dispositivos de bloque con `lsblk`:

```bash
lsblk -f
```

2. Obtén información detallada de particiones y tamaños:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
```

3. Verifica el espacio en disco utilizado en los sistemas de archivos montados:

```bash
df -hT
```

4. Consulta la tabla de particiones del disco principal:

```bash
sudo fdisk -l /dev/sda
```

5. Guarda la información de almacenamiento:

```bash
{
  echo "=== DISPOSITIVOS DE BLOQUE ==="
  lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
  echo ""
  echo "=== USO DE DISCO ==="
  df -hT
  echo ""
  echo "=== TABLA DE PARTICIONES /dev/sda ==="
  sudo fdisk -l /dev/sda
} > /opt/sysreport/storage_info.txt
```

**Salida Esperada:**
# El reporte podria ser diferente al aquí mostrado 
```
NAME   SIZE TYPE FSTYPE MOUNTPOINT        MODEL
sda     40G disk                          VBOX HARDDISK
├─sda1   1G part vfat   /boot/efi
├─sda2   2G part ext4   /boot
└─sda3  37G part LVM2_m
  ├─ubuntu--vg-ubuntu--lv  ...  ext4   /
  ...
```

**Verificación:**

```bash
lsblk | grep -q "sda" && echo "✓ Disco principal detectado" || echo "✗ No se detectó /dev/sda"
```

---

### Paso 4: Inspección de Dispositivos PCI y USB

**Objetivo:** Identificar las tarjetas y controladores conectados a los buses PCI y USB del sistema.

**Instrucciones:**

1. Lista todos los dispositivos PCI:

```bash
lspci

o si estas en una VM de Azure

Listar dispositivos del bus virtual de Hyper-V (VMBus)
ls -l /sys/bus/vmbus/devices/
lsmod | grep hv_

```

2. Obtén información detallada de un dispositivo específico (controladora de red):

```bash
lspci -v | grep -A10 "Ethernet"
o en Azure
ls -l /sys/class/net/
```

3. Lista los dispositivos USB conectados (en Azure no hay dispositivos USB reales):

```bash
lsusb
# Este último comando en Azure no mostrará nada.
```

4. Examina la jerarquía de dispositivos USB con detalle:

```bash
lsusb -t
# Este último comando en Azure no mostrará nada.
```

5. Guarda la información de buses:

```bash
{
  echo "=== DISPOSITIVOS PCI ==="
  lspci
  echo ""
  echo "=== DISPOSITIVOS USB ==="
  lsusb
  echo ""
  echo "=== DETALLE TARJETA DE RED ==="
  lspci -v | grep -A15 -i "ethernet"
} > /opt/sysreport/bus_devices_info.txt
```

**Salida Esperada (ejemplo VirtualBox):**

```
00:00.0 Host bridge: Intel Corporation 440FX - 82441FX PMC [Natoma]
00:01.0 ISA bridge: Intel Corporation 82371SB PIIX3 ISA [Natoma/Triton II]
00:03.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller
00:08.0 SATA controller: Intel Corporation 82801HM/HEM (ICH8M/ICH8M-E) SATA Controller
```

**Verificación:**

```bash
lspci | grep -qi "ethernet" && echo "✓ Tarjeta de red detectada" || echo "✗ No se detectó tarjeta de red"
```

---

### Paso 5: Inventario Completo de Hardware con lshw

**Objetivo:** Generar un inventario completo y estructurado de todo el hardware del sistema.

**Instrucciones:**

1. Ejecuta `lshw` en formato resumido:

```bash
sudo lshw -short
```

2. Genera el inventario completo en formato HTML para referencia visual (opcional):

```bash
sudo lshw -html > /opt/sysreport/hardware_full.html
```

3. Genera el inventario en texto plano:

```bash
sudo lshw > /opt/sysreport/hardware_full.txt
```

4. Extrae solo la clase de red para documentar las interfaces:

```bash
sudo lshw -class network -short
```

**Salida Esperada (`lshw -short`, extracto):**

```
H/W path          Device      Class          Description
========================================================
                              system         VirtualBox
/0                            bus            VirtualBox
/0/0                          memory         128KiB BIOS
/0/1                          memory         4GiB System memory
/0/2                          processor      Intel(R) Core(TM)...
/0/100/3          enp0s3      network        82540EM Gigabit Ethernet Controller
```

**Verificación:**

```bash
test -s /opt/sysreport/hardware_full.txt && echo "✓ Inventario completo generado" || echo "✗ Error"
```

---

### Paso 6: Análisis de Módulos del Kernel

**Objetivo:** Examinar los módulos del kernel cargados, entender su propósito e identificar los controladores activos.

**Instrucciones:**

1. Lista todos los módulos del kernel actualmente cargados:

```bash
lsmod
```

2. Cuenta el número total de módulos cargados:

```bash
lsmod | tail -n +2 | wc -l
```

3. Identifica el módulo de la tarjeta de red virtual (generalmente `e1000`):

```bash
lsmod | grep -i e1000
o usa el siguiente comando si estas en Azure
ethtool -i eth0
```

4. Obtén información detallada del módulo de red:

```bash
modinfo e1000
o en Azure
ls -la /sys/module/hv_netvsc/parameters/
```

5. Examina las dependencias del módulo (si no estas en Azure):

```bash
modinfo e1000 | grep -E '(depends|description|filename|author)'
```

6. Revisa los módulos de almacenamiento cargados (si no estas en Azure):

```bash
lsmod | grep -iE '(ahci|ata|scsi|sd_mod)'
```

7. Guarda la información de módulos:

```bash
{
  echo "=== MÓDULOS CARGADOS ($(lsmod | tail -n +2 | wc -l) total) ==="
  lsmod | sort
  echo ""
  echo "=== DETALLE MÓDULO DE RED ==="
  modinfo e1000 2>/dev/null || modinfo virtio_net 2>/dev/null || echo "Módulo de red no identificado"
  echo ""
  echo "=== MÓDULOS DE ALMACENAMIENTO ==="
  lsmod | grep -iE '(ahci|ata|scsi|sd_mod|virtio_blk)'
} > /opt/sysreport/kernel_modules_info.txt
```

**Salida Esperada (`modinfo e1000`, extracto):**

```
filename:       /lib/modules/5.15.0-xxx-generic/kernel/drivers/net/ethernet/intel/e1000/e1000.ko
description:    Intel(R) PRO/1000 Network Driver
author:         Intel Corporation, <linux.nics@intel.com>
license:        GPL v2
```

**Verificación:**

```bash
lsmod | grep -q "e1000\|virtio_net" && echo "✓ Módulo de red cargado" || echo "⚠ Verificar módulo de red manualmente"
```

---

### Paso 7: Exploración de los Sistemas de Archivos Virtuales /proc y /sys

**Objetivo:** Comprender cómo el kernel expone información del hardware y del sistema a través de `/proc` y `/sys`.

**Instrucciones:**

1. Examina la información de la versión del kernel:

```bash
cat /proc/version
```

2. Consulta los parámetros de arranque del kernel:

```bash
cat /proc/cmdline
```

3. Revisa las interrupciones del sistema:

```bash
cat /proc/interrupts | head -20
```

4. Examina la estructura de `/sys/class` para ver las clases de dispositivos:

```bash
ls /sys/class/
```

5. Identifica las interfaces de red a través de `/sys`:

```bash
ls /sys/class/net/
```

6. Obtén la dirección MAC y el estado de la interfaz principal:

```bash
cat /sys/class/net/enp0s3/address
o
En Azure:
cat /sys/class/net/eth0/address

cat /sys/class/net/enp0s3/operstate
o
En Azure:
cat /sys/class/net/eth0/operstate
```

7. Explora la información del dispositivo de bloque:

```bash
cat /sys/block/sda/size
cat /sys/block/sda/device/model
```

8. Guarda la información de `/proc` y `/sys`:

```bash
{
  echo "=== VERSIÓN DEL KERNEL ==="
  cat /proc/version
  echo ""
  echo "=== LÍNEA DE COMANDOS DEL KERNEL ==="
  cat /proc/cmdline
  echo ""
  echo "=== INTERFACES DE RED (/sys) ==="
  for iface in $(ls /sys/class/net/); do
    echo "  $iface: MAC=$(cat /sys/class/net/$iface/address 2>/dev/null) Estado=$(cat /sys/class/net/$iface/operstate 2>/dev/null)"
  done
  echo ""
  echo "=== CLASES DE DISPOSITIVOS ==="
  ls /sys/class/ | column
} > /opt/sysreport/proc_sys_info.txt
```

**Salida Esperada:**

```
Linux version 5.15.0-xxx-generic (buildd@...) (gcc ...) #xxx-Ubuntu SMP ...
```

**Verificación:**

```bash
cat /proc/version | grep -q "Linux" && echo "✓ /proc accesible" || echo "✗ Error accediendo a /proc"
```

---

### Paso 8: Análisis del Proceso de Arranque

**Objetivo:** Examinar los mensajes de inicialización del sistema para entender la secuencia de arranque y detectar posibles errores.

**Instrucciones:**

1. Revisa los mensajes del kernel durante el arranque actual:

```bash
sudo dmesg | head -50
```

2. Filtra los mensajes de error o advertencia del arranque:

```bash
sudo dmesg --level=err,warn
```

3. Usa `journalctl` para ver los logs del arranque actual:

```bash
journalctl -b --no-pager | head -80
```

4. Identifica cuánto tiempo tomó el arranque:

```bash
systemd-analyze
```

5. Muestra las unidades que más tardaron en iniciar:

```bash
systemd-analyze blame | head -15
```

6. Revisa el orden de arranque en formato de cadena crítica:

```bash
systemd-analyze critical-chain | head -20
```

7. Guarda la información de arranque:

```bash
{
  echo "=== TIEMPO DE ARRANQUE ==="
  systemd-analyze
  echo ""
  echo "=== TOP 15 SERVICIOS MÁS LENTOS ==="
  systemd-analyze blame | head -15
  echo ""
  echo "=== ERRORES/ADVERTENCIAS EN DMESG ==="
  sudo dmesg --level=err,warn | tail -20
  echo ""
  echo "=== CADENA CRÍTICA ==="
  systemd-analyze critical-chain 2>/dev/null | head -20
} > /opt/sysreport/boot_analysis.txt
```

**Salida Esperada (`systemd-analyze`):**

```
Startup finished in 2.345s (kernel) + 8.123s (userspace) = 10.468s
graphical.target reached after 8.100s in userspace
```

**Verificación:**

```bash
test -s /opt/sysreport/boot_analysis.txt && echo "✓ Análisis de arranque guardado" || echo "✗ Error"
```

---

### Paso 9: Inspección de Recursos de Red

**Objetivo:** Documentar la configuración de red activa, interfaces y conectividad básica.

**Instrucciones:**

1. Muestra la configuración de todas las interfaces de red:

```bash
ip addr show
```

2. Consulta la tabla de rutas:

```bash
ip route show
```

3. Verifica la resolución DNS configurada:

```bash
cat /etc/resolv.conf
```

4. Muestra estadísticas de las interfaces:

```bash
ip -s link show
```

5. Verifica conectividad con el gateway:

```bash
ping -c 3 192.168.100.1 (usar la IP asignada en la interface principal)
```

6. Guarda la información de red:

```bash
{
  echo "=== INTERFACES DE RED ==="
  ip addr show
  echo ""
  echo "=== TABLA DE RUTAS ==="
  ip route show
  echo ""
  echo "=== DNS ==="
  cat /etc/resolv.conf
  echo ""
  echo "=== HOSTNAME ==="
  hostnamectl
} > /opt/sysreport/network_info.txt
```

**Salida Esperada (extracto):**

```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.100.10/24 brd 192.168.100.255 scope global enp0s3
```

**Verificación:**

```bash
ip addr show | grep -q "192.168.100.10" && echo "✓ IP configurada correctamente" || echo "✗ IP no encontrada"
```

---

### Paso 10: Generación del Reporte de Diagnóstico Consolidado (modificar los comandos necesarios para un ambiente virtual en Azure)

**Objetivo:** Compilar toda la información recopilada en un reporte único y estructurado que sirva como línea base del entorno.

**Instrucciones:**

1. Define la variable con la fecha actual para el nombre del archivo:

```bash
REPORT_DATE=$(date +%Y%m%d)
REPORT_FILE="/opt/sysreport/initial_report_${REPORT_DATE}.txt"
```

2. Genera el reporte consolidado:

```bash
{
  echo "╔══════════════════════════════════════════════════════════════╗"
  echo "║   REPORTE DE DIAGNÓSTICO INICIAL - srv-linux-01            ║"
  echo "║   Fecha: $(date '+%Y-%m-%d %H:%M:%S')                          ║"
  echo "║   Generado por: $(whoami)                                     ║"
  echo "╚══════════════════════════════════════════════════════════════╝"
  echo ""
  echo "================================================================"
  echo "1. INFORMACIÓN DEL SISTEMA"
  echo "================================================================"
  echo "Hostname: $(hostname)"
  echo "Kernel: $(uname -r)"
  echo "Arquitectura: $(uname -m)"
  echo "Sistema Operativo: $(cat /etc/os-release | grep PRETTY_NAME | cut -d= -f2 | tr -d '"')"
  echo "Uptime: $(uptime -p)"
  echo ""
  echo "================================================================"
  echo "2. PROCESADOR (CPU)"
  echo "================================================================"
  lscpu | grep -E '(Architecture|CPU\(s\)|Thread|Core|Socket|Model name|MHz|cache)'
  echo ""
  echo "================================================================"
  echo "3. MEMORIA RAM"
  echo "================================================================"
  free -h
  echo ""
  echo "================================================================"
  echo "4. ALMACENAMIENTO"
  echo "================================================================"
  lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
  echo ""
  echo "--- Uso de disco ---"
  df -hT | grep -v tmpfs
  echo ""
  echo "================================================================"
  echo "5. DISPOSITIVOS PCI"
  echo "================================================================"
  lspci
  echo ""
  echo "================================================================"
  echo "6. DISPOSITIVOS USB"
  echo "================================================================"
  lsusb
  echo ""
  echo "================================================================"
  echo "7. INTERFACES DE RED"
  echo "================================================================"
  ip -brief addr show
  echo ""
  echo "Gateway: $(ip route | grep default | awk '{print $3}')"
  echo ""
  echo "================================================================"
  echo "8. MÓDULOS DEL KERNEL"
  echo "================================================================"
  echo "Total módulos cargados: $(lsmod | tail -n +2 | wc -l)"
  echo ""
  echo "Módulos principales:"
  lsmod | grep -iE '(e1000|virtio|ahci|ata|ext4|dm_mod)' | sort
  echo ""
  echo "================================================================"
  echo "9. ANÁLISIS DE ARRANQUE"
  echo "================================================================"
  systemd-analyze 2>/dev/null
  echo ""
  echo "Top 10 servicios más lentos:"
  systemd-analyze blame 2>/dev/null | head -10
  echo ""
  echo "================================================================"
  echo "10. ERRORES DETECTADOS EN DMESG"
  echo "================================================================"
  ERRORS=$(sudo dmesg --level=err 2>/dev/null | wc -l)
  WARNINGS=$(sudo dmesg --level=warn 2>/dev/null | wc -l)
  echo "Errores: $ERRORS"
  echo "Advertencias: $WARNINGS"
  if [ "$ERRORS" -gt 0 ]; then
    echo ""
    echo "Últimos errores:"
    sudo dmesg --level=err | tail -5
  fi
  echo ""
  echo "================================================================"
  echo "FIN DEL REPORTE"
  echo "================================================================"
} > "$REPORT_FILE"
```

3. Verifica que el reporte se generó correctamente:

```bash
ls -lh "$REPORT_FILE"
```

4. Muestra las primeras y últimas líneas del reporte:

```bash
head -20 "$REPORT_FILE"
echo "..."
tail -5 "$REPORT_FILE"
```

5. Crea un resumen ejecutivo con los datos clave:

```bash
echo "=== RESUMEN EJECUTIVO ===" 
echo "Servidor: $(hostname)"
echo "IP: $(ip -4 addr show enp0s3 | grep inet | awk '{print $2}')"
echo "CPU: $(lscpu | grep 'Model name' | awk -F: '{print $2}' | xargs)"
echo "RAM Total: $(free -h | grep Mem | awk '{print $2}')"
echo "Disco Principal: $(lsblk -d -o NAME,SIZE | grep sda | awk '{print $2}')"
echo "Kernel: $(uname -r)"
echo "Módulos cargados: $(lsmod | tail -n +2 | wc -l)"
echo "Reporte guardado en: $REPORT_FILE"
```

**Salida Esperada:**

```
=== RESUMEN EJECUTIVO ===
Servidor: srv-linux-01
IP: 192.168.100.10/24
CPU: Intel(R) Core(TM) ... (virtualizado)
RAM Total: 3.8Gi
Disco Principal: 40G
Kernel: 5.15.0-xxx-generic
Módulos cargados: ~75
Reporte guardado en: /opt/sysreport/initial_report_20240XXX.txt
```

**Verificación:**

```bash
wc -l "$REPORT_FILE"
```

El reporte debería tener entre 80 y 200 líneas dependiendo de la configuración del sistema.

---

## Validación y Pruebas Finales

Ejecuta la siguiente secuencia de validación para confirmar que todos los entregables del laboratorio se completaron correctamente:

```bash
echo "╔══════════════════════════════════════════════════╗"
echo "║   VALIDACIÓN FINAL DEL LABORATORIO 01-00-01     ║"
echo "╚══════════════════════════════════════════════════╝"
echo ""

PASS=0
FAIL=0

# Test 1: Directorio de reportes existe
if [ -d /opt/sysreport ]; then
  echo "✓ PASS: Directorio /opt/sysreport existe"
  ((PASS++))
else
  echo "✗ FAIL: Directorio /opt/sysreport no existe"
  ((FAIL++))
fi

# Test 2: Reporte principal generado
REPORT=$(ls /opt/sysreport/initial_report_*.txt 2>/dev/null | head -1)
if [ -s "$REPORT" ]; then
  echo "✓ PASS: Reporte principal generado ($REPORT)"
  ((PASS++))
else
  echo "✗ FAIL: Reporte principal no encontrado"
  ((FAIL++))
fi

# Test 3: Archivo de CPU
if [ -s /opt/sysreport/cpu_info.txt ]; then
  echo "✓ PASS: Información de CPU documentada"
  ((PASS++))
else
  echo "✗ FAIL: Archivo cpu_info.txt no encontrado"
  ((FAIL++))
fi

# Test 4: Archivo de memoria
if [ -s /opt/sysreport/memory_info.txt ]; then
  echo "✓ PASS: Información de memoria documentada"
  ((PASS++))
else
  echo "✗ FAIL: Archivo memory_info.txt no encontrado"
  ((FAIL++))
fi

# Test 5: Archivo de almacenamiento
if [ -s /opt/sysreport/storage_info.txt ]; then
  echo "✓ PASS: Información de almacenamiento documentada"
  ((PASS++))
else
  echo "✗ FAIL: Archivo storage_info.txt no encontrado"
  ((FAIL++))
fi

# Test 6: Archivo de módulos del kernel
if [ -s /opt/sysreport/kernel_modules_info.txt ]; then
  echo "✓ PASS: Módulos del kernel documentados"
  ((PASS++))
else
  echo "✗ FAIL: Archivo kernel_modules_info.txt no encontrado"
  ((FAIL++))
fi

# Test 7: Archivo de arranque
if [ -s /opt/sysreport/boot_analysis.txt ]; then
  echo "✓ PASS: Análisis de arranque documentado"
  ((PASS++))
else
  echo "✗ FAIL: Archivo boot_analysis.txt no encontrado"
  ((FAIL++))
fi

# Test 8: Archivo de red
if [ -s /opt/sysreport/network_info.txt ]; then
  echo "✓ PASS: Información de red documentada"
  ((PASS++))
else
  echo "✗ FAIL: Archivo network_info.txt no encontrado"
  ((FAIL++))
fi

# Test 9: El reporte contiene secciones clave
if grep -q "PROCESADOR" "$REPORT" 2>/dev/null && grep -q "MEMORIA" "$REPORT" 2>/dev/null; then
  echo "✓ PASS: Reporte contiene secciones estructuradas"
  ((PASS++))
else
  echo "✗ FAIL: Reporte incompleto o mal estructurado"
  ((FAIL++))
fi

# Test 10: Herramientas instaladas
if command -v lshw &>/dev/null && command -v htop &>/dev/null && command -v tree &>/dev/null; then
  echo "✓ PASS: Herramientas de diagnóstico instaladas"
  ((PASS++))
else
  echo "✗ FAIL: Faltan herramientas de diagnóstico"
  ((FAIL++))
fi

echo ""
echo "════════════════════════════════════════"
echo "  Resultados: $PASS aprobados / $FAIL fallidos de 10 pruebas"
echo "════════════════════════════════════════"

if [ $FAIL -eq 0 ]; then
  echo "  🎉 ¡LABORATORIO COMPLETADO EXITOSAMENTE!"
else
  echo "  ⚠  Revisa los puntos fallidos antes de continuar."
fi
```

---

## Resolución de Problemas

### Problema 1: `lshw` no produce salida o muestra "WARNING: you should run this program as super user"

**Síntomas:** Al ejecutar `lshw -short`, la salida está incompleta o aparece un mensaje de advertencia indicando que se necesitan privilegios elevados. Algunos dispositivos no se muestran.

**Causa:** La herramienta `lshw` requiere permisos de root para acceder a información completa del hardware a través de `/sys`, `/proc` y llamadas directas al firmware (DMI/SMBIOS). Sin `sudo`, solo puede reportar información parcial.

**Solución:**

```bash
# Ejecutar siempre con sudo para obtener información completa
sudo lshw -short

# Si el paquete no está instalado
sudo apt install -y lshw

# Verificar que funciona correctamente
sudo lshw -class system -short
```

---

### Problema 2: `dmesg` muestra "dmesg: read kernel buffer failed: Operation not permitted"

**Síntomas:** Al ejecutar `dmesg` como usuario normal, se recibe un error de permisos y no se puede acceder al buffer de mensajes del kernel.

**Causa:** A partir de ciertas versiones del kernel Linux, el parámetro `kernel.dmesg_restrict` está habilitado por defecto (valor `1`), lo que restringe el acceso al buffer de mensajes del kernel solo a usuarios con capacidad `CAP_SYSLOG` o root.

**Solución:**

```bash
# Opción 1: Usar sudo (recomendado para laboratorios)
sudo dmesg

# Opción 2: Verificar el valor actual de la restricción
sudo sysctl kernel.dmesg_restrict

# Opción 3 (temporal, solo para el lab): Permitir acceso sin sudo
sudo sysctl -w kernel.dmesg_restrict=0

# Alternativa: Usar journalctl que respeta los permisos de grupo
journalctl -k -b --no-pager
```

> **Nota:** En entornos de producción, mantener `dmesg_restrict=1` es una buena práctica de seguridad. Usar `sudo` es la forma correcta de acceder a esta información.

---

## Limpieza

Este laboratorio genera archivos de referencia que serán utilizados en laboratorios posteriores. **No eliminar** los archivos en `/opt/sysreport/`.

Si necesitas reiniciar el laboratorio desde cero:

```bash
# Solo si es necesario repetir el lab completo
rm -f /opt/sysreport/cpu_info.txt
rm -f /opt/sysreport/memory_info.txt
rm -f /opt/sysreport/storage_info.txt
rm -f /opt/sysreport/bus_devices_info.txt
rm -f /opt/sysreport/hardware_full.txt
rm -f /opt/sysreport/hardware_full.html
rm -f /opt/sysreport/kernel_modules_info.txt
rm -f /opt/sysreport/proc_sys_info.txt
rm -f /opt/sysreport/boot_analysis.txt
rm -f /opt/sysreport/network_info.txt
rm -f /opt/sysreport/initial_report_*.txt
```

---

## Resumen

En este laboratorio has completado las siguientes actividades:

| Actividad | Herramientas Utilizadas | Resultado |
|-----------|------------------------|-----------|
| Inspección de CPU | `lscpu`, `/proc/cpuinfo` | Arquitectura, núcleos y capacidades documentadas |
| Inspección de memoria | `free`, `dmidecode`, `/proc/meminfo` | Capacidad total, uso y configuración registrados |
| Inspección de almacenamiento | `lsblk`, `df`, `fdisk` | Discos, particiones y sistemas de archivos mapeados |
| Inspección de buses | `lspci`, `lsusb` | Dispositivos PCI y USB identificados |
| Inventario completo | `lshw` | Hardware total documentado en texto y HTML |
| Módulos del kernel | `lsmod`, `modinfo` | Controladores activos y sus dependencias analizados |
| Sistemas virtuales | `/proc`, `/sys` | Información dinámica del kernel explorada |
| Análisis de arranque | `dmesg`, `journalctl`, `systemd-analyze` | Secuencia de inicio y tiempos registrados |
| Configuración de red | `ip`, `/etc/resolv.conf`, `hostnamectl` | Interfaces, rutas y DNS documentados |
| Reporte consolidado | Script Bash | Línea base generada en `/opt/sysreport/` |

### Conceptos Clave Reforzados

- La arquitectura Linux opera en capas (hardware → kernel → espacio de usuario) y cada capa expone información a través de interfaces específicas.
- Los sistemas de archivos virtuales `/proc` y `/sys` son la fuente primaria de información en tiempo real sobre el hardware y el kernel.
- Los módulos del kernel (`lsmod`) son los controladores que permiten la comunicación entre el kernel y los dispositivos de hardware.
- Las herramientas de diagnóstico requieren privilegios elevados (`sudo`) para acceder a información completa del sistema.

### Recursos Adicionales

- `man lshw`, `man lscpu`, `man lspci`, `man dmidecode` — Páginas del manual de cada herramienta.
- [Documentación del kernel Linux — /proc filesystem](https://www.kernel.org/doc/html/latest/filesystems/proc.html)
- [Arch Wiki — Sysfs](https://wiki.archlinux.org/title/Sysfs)
- [Ubuntu Server Guide — System Information](https://ubuntu.com/server/docs)

---
