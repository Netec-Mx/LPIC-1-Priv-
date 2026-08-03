# Implementar una estrategia de crecimiento de almacenamiento utilizando LVM, RAID y cuotas para garantizar disponibilidad, escalabilidad y control de recursos en un entorno corporativo

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 135 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio diseñarás e implementarás una infraestructura de almacenamiento empresarial completa en `srv-linux-02`. Combinarás RAID 5 por software (mdadm) para redundancia, LVM para flexibilidad y escalabilidad, y cuotas de disco para control de recursos por usuario. Esta arquitectura multicapa refleja escenarios reales donde la disponibilidad, el crecimiento dinámico y la gobernanza del almacenamiento son requisitos simultáneos.

## Objetivos de Aprendizaje

- [ ] Crear un arreglo RAID 5 con mdadm usando tres discos de 10 GB, verificando su estado y redundancia.
- [ ] Diseñar una topología LVM completa (PV → VG → LV) sobre RAID y disco independiente, con sistemas de archivos XFS y ext4.
- [ ] Extender un volumen lógico en línea sin desmontar el sistema de archivos, demostrando escalabilidad operativa.
- [ ] Crear un snapshot LVM de un volumen lógico activo para respaldo consistente.
- [ ] Implementar cuotas de disco por usuario con límites soft/hard en el sistema de archivos `/home`.

## Prerrequisitos

### Conocimientos Requeridos

- Familiaridad con la jerarquía FHS de Linux (lección 6.1), especialmente los directorios `/dev`, `/opt` y `/var`.
- Comprensión de dispositivos de bloque y archivos en `/dev` gestionados por udev.
- Experiencia básica con particionado GPT y sistemas de archivos (Lab 03).
- Conocimiento de procesamiento de texto en terminal (Lab 05).

### Acceso Requerido

- Sesión SSH o consola en `srv-linux-02` (192.168.100.20) como usuario `sysadmin`.
- Acceso a privilegios root mediante `sudo` (contraseña: `Linux@Admin2024!`).
- Cuatro discos virtuales adicionales conectados: `/dev/sdd`, `/dev/sde`, `/dev/sdf`, `/dev/sdg` (10 GB cada uno).

## Entorno de Laboratorio

### Hardware Virtual (srv-linux-02)

| Recurso | Especificación |
|---------|---------------|
| CPU | 4 vCPUs |
| RAM | 4 GB mínimo |
| Disco sistema | `/dev/sda` — 60 GB (Ubuntu 22.04.4 LTS) |
| Disco datos | `/dev/sdb` — 20 GB (configurado en Lab 03) |
| Disco respaldo | `/dev/sdc` — 20 GB (configurado en Lab 03) |
| Disco RAID-1 | `/dev/sdd` — 10 GB (nuevo) |
| Disco RAID-2 | `/dev/sde` — 10 GB (nuevo) |
| Disco RAID-3 | `/dev/sdf` — 10 GB (nuevo) |
| Disco LVM extra | `/dev/sdg` — 10 GB (nuevo) |

### Software Requerido

| Paquete | Versión | Propósito |
|---------|---------|-----------|
| lvm2 | 2.03.16 | Gestión de volúmenes lógicos |
| mdadm | 4.2 | RAID por software |
| quota | 4.06 | Cuotas de disco |
| xfsprogs | 5.13.0 | Sistema de archivos XFS |
| e2fsprogs | 1.46.5 | Sistema de archivos ext4 |

### Preparación Inicial del Entorno

```bash
# Conectar a srv-linux-02
ssh sysadmin@192.168.100.20

# Verificar que los discos adicionales están presentes
lsblk | grep -E "sd[d-g]"

# Instalar paquetes necesarios (si no están instalados)
sudo apt update
sudo apt install -y lvm2 mdadm xfsprogs quota quotatool

# Crear directorio de reportes
sudo mkdir -p /opt/sysreport
sudo chown sysadmin:sysadmin /opt/sysreport

# Crear directorios de montaje
sudo mkdir -p /opt/appdata /var/log/applogs
```

**Salida esperada de `lsblk | grep -E "sd[d-g]"`:**

```
sdd      8:48   0   10G  0 disk
sde      8:64   0   10G  0 disk
sdf      8:80   0   10G  0 disk
sdg      8:96   0   10G  0 disk
```

> **⚠️ Importante:** Si algún disco no aparece, apaga la VM, verifica la configuración de discos en VirtualBox y reinicia.

---

## Procedimiento Paso a Paso

### Paso 1: Crear el Arreglo RAID 5 con mdadm

**Objetivo:** Configurar un arreglo RAID 5 usando `/dev/sdd`, `/dev/sde` y `/dev/sdf` para obtener redundancia con tolerancia a la falla de un disco.

**Instrucciones:**

1. Verificar que los discos no tienen particiones previas:

```bash
sudo wipefs -a /dev/sdd /dev/sde /dev/sdf
```

2. Crear el arreglo RAID 5 con tres discos:

```bash
sudo mdadm --create /dev/md0 \
  --level=5 \
  --raid-devices=3 \
  /dev/sdd /dev/sde /dev/sdf
```

3. Cuando el sistema pregunte si desea continuar con la creación del arreglo, confirmar con `y`.

4. Verificar el estado de la construcción del RAID:

```bash
cat /proc/mdstat
```

5. Esperar a que la sincronización complete (puede tomar varios minutos). Monitorear con:

```bash
watch -n 2 cat /proc/mdstat
```

6. Una vez completada la sincronización, verificar los detalles del arreglo:

```bash
sudo mdadm --detail /dev/md0
```

7. Guardar la configuración de mdadm para persistencia tras reinicio:

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

**Salida esperada de `mdadm --detail /dev/md0`:**

```
/dev/md0:
           Version : 1.2
     Creation Time : [fecha actual]
        Raid Level : raid5
        Array Size : 20951040 (19.98 GiB 21.45 GB)
     Used Dev Size : 10475520 (9.99 GiB 10.73 GB)
      Raid Devices : 3
     Total Devices : 3
             State : clean
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

    Number   Major   Minor   RaidDevice State
       0       8       48        0      active sync   /dev/sdd
       1       8       64        1      active sync   /dev/sde
       2       8       80        2      active sync   /dev/sdf
```

**Verificación:**

```bash
# El arreglo debe estar activo y limpio
sudo mdadm --detail /dev/md0 | grep -E "State|Raid Level|Array Size"
```

El estado debe mostrar `clean` y el tamaño útil será aproximadamente 20 GB (2 discos × 10 GB, ya que un disco se usa para paridad).

---

### Paso 2: Crear Volúmenes Físicos (PV) para LVM

**Objetivo:** Inicializar `/dev/md0` (RAID 5) y `/dev/sdg` como volúmenes físicos LVM.

**Instrucciones:**

1. Crear el volumen físico sobre el arreglo RAID 5:

```bash
sudo pvcreate /dev/md0
```

2. Crear el volumen físico sobre el disco independiente:

```bash
sudo pvcreate /dev/sdg
```

3. Verificar los volúmenes físicos creados:

```bash
sudo pvdisplay
```

4. Listar de forma resumida:

```bash
sudo pvs
```

**Salida esperada de `pvs`:**

```
  PV         VG   Fmt  Attr PSize   PFree
  /dev/md0        lvm2 ---  <19.98g <19.98g
  /dev/sdg        lvm2 ---  <10.00g <10.00g
```

**Verificación:**

```bash
sudo pvs | grep -c "lvm2"
# Debe devolver al menos 2
```

---

### Paso 3: Crear el Grupo de Volúmenes (VG)

**Objetivo:** Agrupar ambos volúmenes físicos en un único grupo de volúmenes `vg_data` para administración centralizada.

**Instrucciones:**

1. Crear el grupo de volúmenes combinando ambos PVs:

```bash
sudo vgcreate vg_data /dev/md0 /dev/sdg
```

2. Verificar el grupo de volúmenes:

```bash
sudo vgdisplay vg_data
```

3. Confirmar el tamaño total disponible:

```bash
sudo vgs vg_data
```

**Salida esperada de `vgs vg_data`:**

```
  VG      #PV #LV #SN Attr   VSize   VFree
  vg_data   2   0   0 wz--n- <29.98g <29.98g
```

**Verificación:**

```bash
# El VG debe tener ~30 GB (20 GB del RAID + 10 GB de sdg)
sudo vgs vg_data --noheadings -o vg_size | awk '{print $1}'
```

El tamaño debe ser aproximadamente `29.98g`.

---

### Paso 4: Crear Volúmenes Lógicos (LV)

**Objetivo:** Crear dos volúmenes lógicos: `lv_app` (15 GB, XFS) para datos de aplicaciones y `lv_logs` (5 GB, ext4) para registros.

**Instrucciones:**

1. Crear el volumen lógico para aplicaciones:

```bash
sudo lvcreate -L 15G -n lv_app vg_data
```

2. Crear el volumen lógico para logs:

```bash
sudo lvcreate -L 5G -n lv_logs vg_data
```

3. Verificar los volúmenes lógicos creados:

```bash
sudo lvs vg_data
```

4. Ver información detallada:

```bash
sudo lvdisplay /dev/vg_data/lv_app
sudo lvdisplay /dev/vg_data/lv_logs
```

**Salida esperada de `lvs vg_data`:**

```
  LV      VG      Attr       LSize  Pool Origin Data%  Meta%
  lv_app  vg_data -wi-a----- 15.00g
  lv_logs vg_data -wi-a-----  5.00g
```

**Verificación:**

```bash
sudo lvs vg_data --noheadings -o lv_name,lv_size | column -t
```

---

### Paso 5: Formatear los Volúmenes Lógicos

**Objetivo:** Crear sistemas de archivos XFS en `lv_app` y ext4 en `lv_logs`, según los requisitos de cada carga de trabajo.

**Instrucciones:**

1. Formatear `lv_app` con XFS (óptimo para bases de datos y archivos grandes):

```bash
sudo mkfs.xfs /dev/vg_data/lv_app
```

2. Formatear `lv_logs` con ext4 (óptimo para archivos de log pequeños y numerosos):

```bash
sudo mkfs.ext4 /dev/vg_data/lv_logs
```

3. Verificar los sistemas de archivos:

```bash
sudo blkid /dev/vg_data/lv_app
sudo blkid /dev/vg_data/lv_logs
```

**Salida esperada:**

```
/dev/mapper/vg_data-lv_app: UUID="..." TYPE="xfs"
/dev/mapper/vg_data-lv_logs: UUID="..." TYPE="ext4"
```

**Verificación:**

```bash
sudo blkid /dev/vg_data/lv_app | grep -o 'TYPE="[^"]*"'
# Debe mostrar: TYPE="xfs"

sudo blkid /dev/vg_data/lv_logs | grep -o 'TYPE="[^"]*"'
# Debe mostrar: TYPE="ext4"
```

---

### Paso 6: Montar los Volúmenes y Configurar Persistencia

**Objetivo:** Montar los volúmenes lógicos en los directorios del FHS apropiados y configurar `/etc/fstab` para montaje automático al arranque.

**Instrucciones:**

1. Montar `lv_app` en `/opt/appdata` (según FHS, `/opt` aloja paquetes de software adicionales):

```bash
sudo mount /dev/vg_data/lv_app /opt/appdata
```

2. Montar `lv_logs` en `/var/log/applogs` (según FHS, `/var/log` contiene registros):

```bash
sudo mount /dev/vg_data/lv_logs /var/log/applogs
```

3. Verificar los montajes:

```bash
df -hT /opt/appdata /var/log/applogs
```

4. Obtener los UUIDs para la configuración persistente:

```bash
sudo blkid /dev/vg_data/lv_app
sudo blkid /dev/vg_data/lv_logs
```

5. Agregar entradas a `/etc/fstab` para persistencia:

```bash
# Agregar entradas usando rutas de dispositivo LVM (más legibles que UUID para LVM)
echo '/dev/vg_data/lv_app  /opt/appdata   xfs   defaults  0 2' | sudo tee -a /etc/fstab
echo '/dev/vg_data/lv_logs /var/log/applogs ext4 defaults  0 2' | sudo tee -a /etc/fstab
```

6. Validar la sintaxis de fstab:

```bash
sudo mount -a
echo $?
```

**Salida esperada de `df -hT`:**

```
Filesystem                  Type  Size  Used Avail Use% Mounted on
/dev/mapper/vg_data-lv_app  xfs    15G  137M   15G   1% /opt/appdata
/dev/mapper/vg_data-lv_logs ext4  4.9G   24K  4.6G   1% /var/log/applogs
```

**Verificación:**

```bash
# mount -a no debe producir errores (exit code 0)
sudo mount -a && echo "fstab OK" || echo "ERROR en fstab"

# Verificar montajes activos
mount | grep vg_data
```

---

### Paso 7: Extender lv_logs en Línea (de 5 GB a 8 GB)

**Objetivo:** Demostrar la escalabilidad de LVM extendiendo un volumen lógico sin desmontar el sistema de archivos, simulando un crecimiento de demanda operativa.

**Instrucciones:**

1. Verificar el espacio libre en el grupo de volúmenes:

```bash
sudo vgs vg_data --noheadings -o vg_free
```

2. Extender el volumen lógico a 8 GB:

```bash
sudo lvextend -L 8G /dev/vg_data/lv_logs
```

3. Redimensionar el sistema de archivos ext4 en línea:

```bash
sudo resize2fs /dev/vg_data/lv_logs
```

4. Verificar el nuevo tamaño:

```bash
df -h /var/log/applogs
sudo lvs /dev/vg_data/lv_logs
```

**Salida esperada:**

```
  LV      VG      Attr       LSize Pool Origin Data%  Meta%
  lv_logs vg_data -wi-ao---- 8.00g
```

```
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/vg_data-lv_logs 7.8G   24K  7.4G   1% /var/log/applogs
```

**Verificación:**

```bash
# El volumen debe mostrar 8 GB
sudo lvs /dev/vg_data/lv_logs --noheadings -o lv_size | xargs
# Debe devolver: 8.00g

# El sistema de archivos debe reflejar el nuevo tamaño
df -h /var/log/applogs | awk 'NR==2{print $2}'
# Debe mostrar aproximadamente 7.8G o 7.9G
```

> **📝 Nota:** La extensión de ext4 con `resize2fs` funciona en línea (sin desmontar). Para XFS se usa `xfs_growfs` como se verá en el siguiente paso si fuera necesario.

---

### Paso 8: Crear un Snapshot de lv_app para Respaldo

**Objetivo:** Crear un snapshot LVM del volumen `lv_app` para obtener una imagen consistente en un punto en el tiempo, útil para respaldos sin detener servicios.

**Instrucciones:**

1. Verificar espacio libre disponible para el snapshot:

```bash
sudo vgs vg_data --noheadings -o vg_free
```

2. Crear un snapshot de 2 GB de `lv_app`:

```bash
sudo lvcreate -L 2G -s -n snap_lv_app /dev/vg_data/lv_app
```

3. Verificar el snapshot creado:

```bash
sudo lvs vg_data
```

4. Montar el snapshot en modo solo lectura para verificación:

```bash
sudo mkdir -p /mnt/snapshot_app
sudo mount -o ro,nouuid /dev/vg_data/snap_lv_app /mnt/snapshot_app
```

> **Nota:** La opción `nouuid` es necesaria para XFS ya que no permite montar dos sistemas de archivos con el mismo UUID simultáneamente.

5. Verificar el contenido del snapshot:

```bash
ls -la /mnt/snapshot_app/
```

6. Desmontar el snapshot después de la verificación:

```bash
sudo umount /mnt/snapshot_app
```

7. Ver el porcentaje de uso del snapshot:

```bash
sudo lvs /dev/vg_data/snap_lv_app -o lv_name,lv_size,data_percent,origin
```

**Salida esperada de `lvs vg_data`:**

```
  LV           VG      Attr       LSize  Pool Origin Data%  Meta%
  lv_app       vg_data owi-aos--- 15.00g
  lv_logs      vg_data -wi-ao----  8.00g
  snap_lv_app  vg_data swi-a-s---  2.00g      lv_app 0.00
```

**Verificación:**

```bash
# El snapshot debe existir y mostrar su origen
sudo lvs /dev/vg_data/snap_lv_app --noheadings -o origin | xargs
# Debe devolver: lv_app
```

---

### Paso 9: Eliminar el Snapshot (Limpieza Post-Respaldo)

**Objetivo:** Eliminar el snapshot una vez completado el respaldo para liberar espacio en el VG.

**Instrucciones:**

1. Eliminar el snapshot:

```bash
sudo lvremove -f /dev/vg_data/snap_lv_app
```

2. Verificar que se eliminó:

```bash
sudo lvs vg_data
```

**Salida esperada:**

```
  LV      VG      Attr       LSize  Pool Origin Data%  Meta%
  lv_app  vg_data -wi-ao---- 15.00g
  lv_logs vg_data -wi-ao----  8.00g
```

**Verificación:**

```bash
sudo lvs vg_data | grep -c snap
# Debe devolver: 0
```

---

### Paso 10: Implementar Cuotas de Disco en /home

**Objetivo:** Configurar cuotas de disco por usuario en `/home` con límites soft de 500 MB y hard de 600 MB para controlar el consumo de almacenamiento.

**Instrucciones:**

1. Verificar el sistema de archivos de `/home`:

```bash
df -hT /home
```

2. Editar `/etc/fstab` para agregar soporte de cuotas en la partición de `/home`. Identificar la línea correspondiente y agregar las opciones `usrquota,grpquota`:

```bash
# Ver la entrada actual de /home
grep '/home' /etc/fstab

# Si /home está en la partición raíz, modificar la entrada de /
# Si /home tiene su propia partición, modificar esa línea
# Ejemplo: agregar usrquota,grpquota a las opciones de montaje
sudo sed -i '/\/home/s/defaults/defaults,usrquota,grpquota/' /etc/fstab
```

> **Nota:** Si `/home` no tiene una entrada separada en fstab (está dentro de `/`), modificar la entrada de `/` para agregar `usrquota,grpquota`.

3. Si `/home` está en la partición raíz, modificar la entrada raíz:

```bash
# Verificar si /home tiene entrada propia
grep -c '/home' /etc/fstab

# Si devuelve 0, /home está en la raíz. Modificar la raíz:
sudo sed -i '/\s\/\s/s/errors=remount-ro/errors=remount-ro,usrquota,grpquota/' /etc/fstab
```

4. Remontar la partición con las nuevas opciones:

```bash
sudo mount -o remount /home
# O si /home está en /:
sudo mount -o remount /
```

5. Verificar que las opciones de cuota están activas:

```bash
mount | grep quota
```

6. Crear los archivos de cuota:

```bash
sudo quotacheck -cugm /home
# Si /home está en /, usar:
# sudo quotacheck -cugm /
```

> **Nota:** Si `quotacheck` da error en sistemas con journaling, usar la opción `-f` para forzar:
> ```bash
> sudo quotacheck -cugmf /home
> ```

7. Activar el sistema de cuotas:

```bash
sudo quotaon /home
# O si está en /:
# sudo quotaon /
```

8. Crear un usuario de prueba para aplicar cuotas:

```bash
sudo useradd -m -s /bin/bash quota_test
sudo passwd quota_test
# Establecer contraseña: Test@Quota2024!
```

9. Establecer cuotas para el usuario `quota_test` (soft: 500 MB, hard: 600 MB):

```bash
sudo setquota -u quota_test 512000 614400 0 0 /home
# Parámetros: block-soft block-hard inode-soft inode-hard filesystem
# 512000 bloques de 1K = 500 MB (soft)
# 614400 bloques de 1K = 600 MB (hard)
```

> **Alternativa con edquota:**
> ```bash
> sudo edquota -u quota_test
> # Editar manualmente: soft=512000, hard=614400 (en bloques de 1K)
> ```

10. Verificar la cuota asignada:

```bash
sudo quota -u quota_test
sudo repquota -a
```

**Salida esperada de `repquota -a`:**

```
*** Report for user quotas on device /dev/sda2
Block grace time: 7days; Inode grace time: 7days
                        Block limits                File limits
User            used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------
root      --   xxxxx       0       0              x     0     0
sysadmin  --   xxxxx       0       0              x     0     0
quota_test--       0  512000  614400              0     0     0
```

**Verificación:**

```bash
# Verificar que la cuota está activa
sudo quotaon -p /home
# Debe indicar que las cuotas están activas

# Verificar límites del usuario
sudo quota -u quota_test | grep -E "blocks|quota_test"
```

---

### Paso 11: Probar los Límites de Cuota

**Objetivo:** Validar que el sistema de cuotas funciona correctamente intentando exceder los límites configurados.

**Instrucciones:**

1. Cambiar al usuario `quota_test`:

```bash
sudo su - quota_test
```

2. Intentar crear un archivo que exceda el límite soft (500 MB):

```bash
dd if=/dev/zero of=~/testfile_510MB bs=1M count=510
```

3. Verificar que se recibe una advertencia (el archivo se crea porque solo excede el soft limit):

```bash
ls -lh ~/testfile_510MB
quota
```

4. Intentar crear un archivo que exceda el límite hard (600 MB total):

```bash
dd if=/dev/zero of=~/testfile_extra bs=1M count=100
```

5. Este comando debe fallar con un error de cuota excedida:

**Salida esperada:**

```
dd: error writing '/home/quota_test/testfile_extra': Disk quota exceeded
```

6. Salir del usuario de prueba:

```bash
exit
```

7. Verificar el reporte de cuotas:

```bash
sudo repquota -a | grep quota_test
```

**Verificación:**

```bash
# El usuario debe estar marcado con '+' indicando que excedió soft limit
sudo repquota -a | grep quota_test
# Ejemplo: quota_test  +- 614400  512000  614400  7days    ...
```

---

### Paso 12: Generar Reporte de Almacenamiento

**Objetivo:** Documentar toda la configuración implementada en un reporte consolidado según los estándares del entorno.

**Instrucciones:**

1. Crear el script de generación de reporte:

```bash
cat << 'EOF' | sudo tee /opt/scripts/storage_report.sh
#!/bin/bash
# Script: storage_report.sh
# Propósito: Generar reporte de almacenamiento avanzado
# Fecha: $(date)

REPORT="/opt/sysreport/storage_advanced_report.txt"

echo "=============================================" > "$REPORT"
echo " REPORTE DE ALMACENAMIENTO AVANZADO" >> "$REPORT"
echo " Servidor: $(hostname)" >> "$REPORT"
echo " Fecha: $(date '+%Y-%m-%d %H:%M:%S')" >> "$REPORT"
echo "=============================================" >> "$REPORT"

echo "" >> "$REPORT"
echo "--- SECCIÓN 1: ESTADO DEL RAID ---" >> "$REPORT"
echo "" >> "$REPORT"
cat /proc/mdstat >> "$REPORT"
echo "" >> "$REPORT"
mdadm --detail /dev/md0 >> "$REPORT" 2>/dev/null

echo "" >> "$REPORT"
echo "--- SECCIÓN 2: VOLÚMENES FÍSICOS (PV) ---" >> "$REPORT"
echo "" >> "$REPORT"
pvs >> "$REPORT"

echo "" >> "$REPORT"
echo "--- SECCIÓN 3: GRUPOS DE VOLÚMENES (VG) ---" >> "$REPORT"
echo "" >> "$REPORT"
vgs >> "$REPORT"
echo "" >> "$REPORT"
vgdisplay vg_data >> "$REPORT"

echo "" >> "$REPORT"
echo "--- SECCIÓN 4: VOLÚMENES LÓGICOS (LV) ---" >> "$REPORT"
echo "" >> "$REPORT"
lvs vg_data >> "$REPORT"
echo "" >> "$REPORT"
lvdisplay /dev/vg_data/lv_app >> "$REPORT"
echo "" >> "$REPORT"
lvdisplay /dev/vg_data/lv_logs >> "$REPORT"

echo "" >> "$REPORT"
echo "--- SECCIÓN 5: SISTEMAS DE ARCHIVOS MONTADOS ---" >> "$REPORT"
echo "" >> "$REPORT"
df -hT | grep -E "vg_data|Filesystem" >> "$REPORT"

echo "" >> "$REPORT"
echo "--- SECCIÓN 6: CONFIGURACIÓN FSTAB ---" >> "$REPORT"
echo "" >> "$REPORT"
grep -E "vg_data|quota" /etc/fstab >> "$REPORT"

echo "" >> "$REPORT"
echo "--- SECCIÓN 7: CUOTAS DE DISCO ---" >> "$REPORT"
echo "" >> "$REPORT"
repquota -a >> "$REPORT" 2>/dev/null

echo "" >> "$REPORT"
echo "--- SECCIÓN 8: CONFIGURACIÓN MDADM ---" >> "$REPORT"
echo "" >> "$REPORT"
cat /etc/mdadm/mdadm.conf >> "$REPORT"

echo "" >> "$REPORT"
echo "=============================================" >> "$REPORT"
echo " FIN DEL REPORTE" >> "$REPORT"
echo "=============================================" >> "$REPORT"

echo "Reporte generado en: $REPORT"
EOF
```

2. Hacer ejecutable y ejecutar el script:

```bash
sudo chmod +x /opt/scripts/storage_report.sh
sudo /opt/scripts/storage_report.sh
```

3. Verificar el reporte generado:

```bash
cat /opt/sysreport/storage_advanced_report.txt
```

**Verificación:**

```bash
# El archivo debe existir y tener contenido significativo
wc -l /opt/sysreport/storage_advanced_report.txt
# Debe tener al menos 50 líneas

grep -c "SECCIÓN" /opt/sysreport/storage_advanced_report.txt
# Debe devolver: 8
```

---

## Validación y Pruebas Finales

Ejecutar las siguientes verificaciones para confirmar que toda la infraestructura está correctamente configurada:

```bash
echo "=== VALIDACIÓN COMPLETA DEL LABORATORIO ==="
echo ""

# 1. RAID 5 activo y limpio
echo "1. Estado RAID 5:"
sudo mdadm --detail /dev/md0 | grep "State" | head -1
echo ""

# 2. VG con espacio correcto
echo "2. Grupo de Volúmenes:"
sudo vgs vg_data --noheadings -o vg_name,vg_size,vg_free
echo ""

# 3. LVs con tamaños correctos
echo "3. Volúmenes Lógicos:"
sudo lvs vg_data --noheadings -o lv_name,lv_size
echo ""

# 4. Sistemas de archivos montados
echo "4. Montajes activos:"
df -hT /opt/appdata /var/log/applogs
echo ""

# 5. Persistencia en fstab
echo "5. Entradas fstab:"
grep vg_data /etc/fstab
echo ""

# 6. Cuotas activas
echo "6. Estado de cuotas:"
sudo quotaon -p /home 2>/dev/null || sudo quotaon -p / 2>/dev/null
echo ""

# 7. Reporte generado
echo "7. Reporte:"
ls -la /opt/sysreport/storage_advanced_report.txt
echo ""

echo "=== VALIDACIÓN COMPLETADA ==="
```

**Criterios de éxito:**

| Criterio | Resultado Esperado |
|----------|-------------------|
| RAID 5 estado | `clean` o `active` |
| VG tamaño total | ~29.98 GB |
| lv_app tamaño | 15.00 GB |
| lv_logs tamaño | 8.00 GB (extendido) |
| lv_app filesystem | XFS |
| lv_logs filesystem | ext4 |
| Montaje /opt/appdata | Activo |
| Montaje /var/log/applogs | Activo |
| Cuotas | Activas |
| Reporte | Generado con 8 secciones |

---

## Solución de Problemas

### Problema 1: El arreglo RAID no sincroniza o muestra estado "inactive"

**Síntomas:**
- `cat /proc/mdstat` muestra `md0 : inactive` o no muestra progreso de sincronización.
- `mdadm --detail /dev/md0` muestra `State : inactive`.

**Causa:**
Los discos pueden tener residuos de metadatos de configuraciones anteriores (superblocks RAID previos, firmas LVM o tablas de particiones GPT/MBR). mdadm no puede ensamblar un arreglo limpio sobre discos con metadatos conflictivos.

**Solución:**

```bash
# Detener el arreglo inactivo
sudo mdadm --stop /dev/md0

# Limpiar completamente los metadatos de cada disco
sudo mdadm --zero-superblock /dev/sdd
sudo mdadm --zero-superblock /dev/sde
sudo mdadm --zero-superblock /dev/sdf

# Eliminar cualquier firma residual
sudo wipefs -a /dev/sdd
sudo wipefs -a /dev/sde
sudo wipefs -a /dev/sdf

# Recrear el arreglo
sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdd /dev/sde /dev/sdf

# Verificar estado
cat /proc/mdstat
```

---

### Problema 2: `quotacheck` falla con "Cannot find filesystem to check" o "Quotafile not found"

**Síntomas:**
- Al ejecutar `sudo quotacheck -cugm /home` se obtiene el error: `quotacheck: Cannot find filesystem to check or filesystem not mounted with quota option`.
- O bien: `quotacheck: Scanning /dev/sdXX [/home] quotacheck: Old file not found`.

**Causa:**
El sistema de archivos no fue remontado con las opciones `usrquota,grpquota` después de modificar `/etc/fstab`, o la modificación en fstab fue incorrecta (error de sintaxis o la partición de `/home` no está separada de `/`).

**Solución:**

```bash
# Verificar las opciones de montaje actuales
mount | grep -E "/home|/ " | grep -v tmpfs

# Verificar que fstab tiene las opciones correctas
cat /etc/fstab | grep -E "/home|/ " | grep -v "^#"

# Si /home no tiene partición separada, las cuotas van en /
# Verificar cuál es el dispositivo de /
findmnt /home

# Remontar con opciones de cuota
sudo mount -o remount,usrquota,grpquota /home
# O si /home está en /:
sudo mount -o remount,usrquota,grpquota /

# Verificar que las opciones están activas
mount | grep quota

# Reintentar quotacheck con forzado
sudo quotacheck -cugmf /home
# O para /:
sudo quotacheck -cugmf /

# Activar cuotas
sudo quotaon -avug
```

---

## Limpieza

> **⚠️ NO ejecutar esta sección si planeas continuar con los laboratorios 07 y 09**, ya que `lv_app` en `/opt/appdata` será utilizado para la base de datos MySQL y las cuotas para gestión de usuarios.

Si necesitas revertir la configuración por cualquier motivo:

```bash
# 1. Desactivar cuotas
sudo quotaoff -a

# 2. Eliminar usuario de prueba
sudo userdel -r quota_test

# 3. Desmontar volúmenes
sudo umount /opt/appdata
sudo umount /var/log/applogs

# 4. Eliminar volúmenes lógicos
sudo lvremove -f /dev/vg_data/lv_app
sudo lvremove -f /dev/vg_data/lv_logs

# 5. Eliminar grupo de volúmenes
sudo vgremove vg_data

# 6. Eliminar volúmenes físicos
sudo pvremove /dev/md0
sudo pvremove /dev/sdg

# 7. Detener y eliminar RAID
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock /dev/sdd /dev/sde /dev/sdf

# 8. Eliminar entradas de fstab
sudo sed -i '/vg_data/d' /etc/fstab

# 9. Eliminar configuración mdadm agregada
sudo sed -i '/md0/d' /etc/mdadm/mdadm.conf
sudo update-initramfs -u

# 10. Eliminar opciones de cuota de fstab
sudo sed -i 's/,usrquota,grpquota//' /etc/fstab
sudo mount -o remount /
```

---

## Resumen

En este laboratorio has implementado una infraestructura de almacenamiento empresarial completa que demuestra:

| Componente | Logro |
|-----------|-------|
| **RAID 5** | Redundancia con tolerancia a fallo de 1 disco, ~20 GB útiles de 30 GB brutos |
| **LVM** | Abstracción de almacenamiento con PV → VG → LV, permitiendo gestión flexible |
| **Extensión en línea** | `lv_logs` extendido de 5 GB a 8 GB sin interrupción de servicio |
| **Snapshots** | Respaldo consistente de `lv_app` sin detener aplicaciones |
| **Cuotas** | Control de recursos por usuario con límites soft (500 MB) y hard (600 MB) |
| **Documentación** | Reporte automatizado en `/opt/sysreport/storage_advanced_report.txt` |

### Relación con el FHS

La ubicación de cada componente sigue el estándar FHS estudiado en la lección 6.1:
- `/opt/appdata` → datos de aplicaciones adicionales (FHS: `/opt` para software autocontenido)
- `/var/log/applogs` → registros variables del sistema (FHS: `/var/log` para logs)
- `/dev/md0`, `/dev/vg_data/*` → archivos de dispositivo gestionados por udev (FHS: `/dev`)
- `/etc/fstab`, `/etc/mdadm/mdadm.conf` → configuración estática del host (FHS: `/etc`)

### Recursos Adicionales

- `man mdadm` — Documentación completa de RAID por software
- `man lvm` — Visión general del subsistema LVM
- `man quota` — Sistema de cuotas de disco
- [Red Hat Storage Administration Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/managing_storage_devices/)
- [Ubuntu LVM Guide](https://ubuntu.com/server/docs/install/storage)
