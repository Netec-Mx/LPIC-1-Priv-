---LAB_START---
LAB_ID: 07-00-01
---MARKDOWN---
# Desarrollo de Scripts Bash para Automatización Administrativa

| Campo | Detalle |
|-------|---------|
| **Duración** | 117 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio desarrollarás una suite completa de 5 scripts Bash modulares que automatizan tareas administrativas críticas en un entorno Linux de producción. Los scripts integran conocimientos de laboratorios anteriores: inventario de hardware/software, monitoreo de servicios con systemctl, reportes de almacenamiento (LVM/RAID), análisis de logs con herramientas de procesamiento de texto, y generación de reportes desde MySQL 8.0.36. Todos los scripts implementan manejo robusto de errores mediante `trap`, `set -e`, `set -u`, logging centralizado y validación de parámetros con `getopts`.

## Objetivos de Aprendizaje

- [ ] Desarrollar scripts Bash modulares con funciones, manejo de parámetros (`getopts`) y validación de errores para automatizar tareas administrativas
- [ ] Implementar scripts de monitoreo del sistema que generen alertas basadas en umbrales de CPU, memoria y disco
- [ ] Automatizar la generación de reportes del sistema integrando consultas SQL a MySQL 8.0.36
- [ ] Aplicar técnicas de depuración (`bash -x`, `set -e`, `trap`) para garantizar la robustez de los scripts
- [ ] Crear un framework de logging centralizado reutilizable en `/var/log/admin_scripts.log`

## Prerrequisitos

### Conocimientos Requeridos

- Comprensión del entorno shell, variables locales y de entorno, y archivos de inicialización (Lección 7.1)
- Manejo básico de comandos Linux (ls, cat, grep, awk, sed)
- Familiaridad con systemctl para gestión de servicios
- Conceptos básicos de LVM y RAID
- Consultas SQL básicas (SELECT, INSERT)

### Acceso Requerido

- Acceso como `sysadmin` a `srv-linux-01` (192.168.100.10)
- Acceso como `sysadmin` a `srv-linux-02` (192.168.100.20) para MySQL
- Conectividad de red entre ambos servidores verificada
- Privilegios `sudo` en ambas máquinas

## Entorno del Laboratorio

### Infraestructura

| Servidor | SO | IP | Rol |
|----------|----|----|-----|
| srv-linux-01 | Ubuntu 22.04.4 LTS | 192.168.100.10 | Ejecución de scripts |
| srv-linux-02 | Ubuntu 22.04.4 LTS | 192.168.100.20 | Servidor MySQL 8.0.36 |

### Software Requerido en srv-linux-01

| Paquete | Versión |
|---------|---------|
| Bash | 5.1.16 |
| curl | 7.81.0 |
| jq | 1.6 |
| mysql-client | 8.0.36 |
| gawk | 5.1.0 |
| sed | 4.8 |
| grep | 3.7 |

### Preparación Inicial del Entorno

Ejecutar en **srv-linux-01** como `sysadmin`:

```bash
# Crear estructura de directorios de trabajo
sudo mkdir -p /opt/scripts
sudo mkdir -p /opt/sysreport
sudo mkdir -p /opt/backups
sudo chown -R sysadmin:sysadmin /opt/scripts /opt/sysreport /opt/backups

# Crear archivo de log centralizado
sudo touch /var/log/admin_scripts.log
sudo chown sysadmin:sysadmin /var/log/admin_scripts.log
sudo chmod 664 /var/log/admin_scripts.log

# Verificar herramientas disponibles
bash --version | head -1
mysql --version
curl --version | head -1
jq --version
```

**Salida esperada:**
```
GNU bash, version 5.1.16(1)-release (x86_64-pc-linux-gnu)
mysql  Ver 8.0.36 for Linux on x86_64 (MySQL Community Server - GPL)
curl 7.81.0 (x86_64-pc-linux-gnu)
jq-1.6
```

Ejecutar en **srv-linux-02** para preparar la base de datos:

```bash
# Conectar a MySQL y crear la base de datos de inventario
sudo mysql -u root -p'R00t@Linux2024!' << 'EOF'
CREATE DATABASE IF NOT EXISTS srv_inventory;
USE srv_inventory;

CREATE TABLE IF NOT EXISTS servers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    hostname VARCHAR(50) NOT NULL,
    ip VARCHAR(15) NOT NULL,
    os VARCHAR(50) NOT NULL,
    status ENUM('active','inactive','maintenance') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insertar datos de ejemplo
INSERT INTO servers (hostname, ip, os, status) VALUES
('srv-linux-01', '192.168.100.10', 'Ubuntu 22.04.4 LTS', 'active'),
('srv-linux-02', '192.168.100.20', 'Ubuntu 22.04.4 LTS', 'active'),
('srv-linux-03', '192.168.100.30', 'Rocky Linux 9.3', 'active'),
('srv-web-01', '192.168.100.40', 'Ubuntu 22.04.4 LTS', 'maintenance'),
('srv-db-backup', '192.168.100.50', 'Rocky Linux 9.3', 'inactive');

-- Crear usuario para scripts si no existe
CREATE USER IF NOT EXISTS 'dbuser_scripts'@'192.168.100.10' IDENTIFIED BY 'DB@Scripts2024!';
GRANT SELECT, INSERT, UPDATE ON srv_inventory.* TO 'dbuser_scripts'@'192.168.100.10';
FLUSH PRIVILEGES;
EOF
```

Verificar conectividad desde **srv-linux-01**:

```bash
mysql -h 192.168.100.20 -u dbuser_scripts -p'DB@Scripts2024!' -e "SELECT COUNT(*) FROM srv_inventory.servers;"
```

**Salida esperada:**
```
+----------+
| COUNT(*) |
+----------+
|        5 |
+----------+
```

---

## Paso 1: Crear el Framework de Logging y Funciones Comunes

### Objetivo
Desarrollar una biblioteca de funciones compartidas (`common_lib.sh`) que todos los scripts utilizarán para logging, manejo de errores y validaciones.

### Instrucciones

1. Crear el archivo de biblioteca común:

```bash
cat > /opt/scripts/common_lib.sh << 'EOF'
#!/bin/bash
#===============================================================================
# Archivo: common_lib.sh
# Descripción: Biblioteca de funciones comunes para scripts de automatización
# Autor: sysadmin
# Versión: 1.0
#===============================================================================

# Variables globales de la biblioteca
readonly LOG_FILE="/var/log/admin_scripts.log"
readonly REPORT_DIR="/opt/sysreport"
readonly BACKUP_DIR="/opt/backups"
readonly SCRIPT_DIR="/opt/scripts"

# Colores para salida en terminal
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly BLUE='\033[0;34m'
readonly NC='\033[0m' # Sin color

#-------------------------------------------------------------------------------
# Función: log_message
# Descripción: Registra mensajes en el archivo de log y opcionalmente en stdout
# Parámetros: $1 = nivel (INFO|WARN|ERROR|DEBUG), $2 = mensaje
#-------------------------------------------------------------------------------
log_message() {
    local level="${1:-INFO}"
    local message="${2:-Sin mensaje}"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local script_name
    script_name=$(basename "${0}")
    
    # Escribir al archivo de log
    echo "[${timestamp}] [${level}] [${script_name}] ${message}" >> "${LOG_FILE}"
    
    # También enviar al syslog usando logger
    logger -t "admin_scripts" -p "local0.info" "[${level}] [${script_name}] ${message}"
    
    # Mostrar en terminal con colores según nivel
    case "${level}" in
        INFO)  echo -e "${GREEN}[INFO]${NC} ${message}" ;;
        WARN)  echo -e "${YELLOW}[WARN]${NC} ${message}" >&2 ;;
        ERROR) echo -e "${RED}[ERROR]${NC} ${message}" >&2 ;;
        DEBUG) [[ "${DEBUG:-0}" == "1" ]] && echo -e "${BLUE}[DEBUG]${NC} ${message}" ;;
    esac
}

#-------------------------------------------------------------------------------
# Función: check_root
# Descripción: Verifica si el script se ejecuta con privilegios de root
#-------------------------------------------------------------------------------
check_root() {
    if [[ "${EUID}" -ne 0 ]]; then
        log_message "ERROR" "Este script requiere privilegios de root. Use sudo."
        exit 1
    fi
}

#-------------------------------------------------------------------------------
# Función: check_dependency
# Descripción: Verifica que un comando esté disponible en el sistema
# Parámetros: $1 = nombre del comando
#-------------------------------------------------------------------------------
check_dependency() {
    local cmd="${1}"
    if ! command -v "${cmd}" &>/dev/null; then
        log_message "ERROR" "Dependencia no encontrada: ${cmd}"
        return 1
    fi
    log_message "DEBUG" "Dependencia verificada: ${cmd}"
    return 0
}

#-------------------------------------------------------------------------------
# Función: check_dependencies
# Descripción: Verifica múltiples dependencias
# Parámetros: Lista de comandos separados por espacio
#-------------------------------------------------------------------------------
check_dependencies() {
    local missing=0
    for cmd in "$@"; do
        if ! check_dependency "${cmd}"; then
            missing=$((missing + 1))
        fi
    done
    if [[ ${missing} -gt 0 ]]; then
        log_message "ERROR" "${missing} dependencia(s) faltante(s). Abortando."
        exit 1
    fi
}

#-------------------------------------------------------------------------------
# Función: cleanup_on_exit
# Descripción: Función de limpieza ejecutada al finalizar el script
#-------------------------------------------------------------------------------
cleanup_on_exit() {
    local exit_code=$?
    if [[ ${exit_code} -ne 0 ]]; then
        log_message "WARN" "Script finalizado con código de error: ${exit_code}"
    else
        log_message "INFO" "Script finalizado exitosamente."
    fi
}

#-------------------------------------------------------------------------------
# Función: generate_report_header
# Descripción: Genera encabezado estándar para reportes
# Parámetros: $1 = título del reporte
#-------------------------------------------------------------------------------
generate_report_header() {
    local title="${1:-Reporte del Sistema}"
    local separator
    separator=$(printf '=%.0s' {1..70})
    
    cat << HEADER
${separator}
  ${title}
  Servidor: $(hostname)
  Fecha: $(date '+%Y-%m-%d %H:%M:%S')
  Usuario: $(whoami)
${separator}

HEADER
}

#-------------------------------------------------------------------------------
# Función: send_alert
# Descripción: Envía alerta al log y opcionalmente a un archivo de alertas
# Parámetros: $1 = severidad (critical|warning|info), $2 = mensaje
#-------------------------------------------------------------------------------
send_alert() {
    local severity="${1:-info}"
    local message="${2:-Alerta sin descripción}"
    local alert_file="${REPORT_DIR}/alerts_$(date '+%Y%m%d').log"
    
    echo "[$(date '+%H:%M:%S')] [${severity^^}] ${message}" >> "${alert_file}"
    log_message "WARN" "ALERTA [${severity^^}]: ${message}"
}

EOF
chmod +x /opt/scripts/common_lib.sh
```

2. Verificar la biblioteca:

```bash
# Probar que se puede cargar sin errores
bash -n /opt/scripts/common_lib.sh && echo "Sintaxis correcta"

# Probar funciones básicas
source /opt/scripts/common_lib.sh
log_message "INFO" "Prueba de biblioteca common_lib.sh"
check_dependency "bash"
echo "LOG_FILE=${LOG_FILE}"
```

### Salida Esperada

```
Sintaxis correcta
[INFO] Prueba de biblioteca common_lib.sh
LOG_FILE=/var/log/admin_scripts.log
```

### Verificación

```bash
# Confirmar que el mensaje se registró en el log
tail -1 /var/log/admin_scripts.log
```

Debe mostrar una línea con formato: `[FECHA] [INFO] [bash] Prueba de biblioteca common_lib.sh`

---

## Paso 2: Script 1 — system_inventory.sh (Inventario del Sistema)

### Objetivo
Crear un script que genere un reporte completo del hardware y software del servidor, utilizando variables de entorno y comandos del sistema.

### Instrucciones

1. Crear el script de inventario:

```bash
cat > /opt/scripts/system_inventory.sh << 'SCRIPT'
#!/bin/bash
#===============================================================================
# Script: system_inventory.sh
# Descripción: Genera reporte de inventario de hardware y software del sistema
# Uso: ./system_inventory.sh [-o archivo_salida] [-v] [-h]
# Versión: 1.0
#===============================================================================

set -euo pipefail

# Cargar biblioteca común
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/common_lib.sh"

# Trap para limpieza al salir
trap cleanup_on_exit EXIT

# Variables por defecto
OUTPUT_FILE="${REPORT_DIR}/inventory_$(hostname)_$(date '+%Y%m%d_%H%M%S').txt"
VERBOSE=0

#-------------------------------------------------------------------------------
# Función: show_usage
#-------------------------------------------------------------------------------
show_usage() {
    cat << USAGE
Uso: $(basename "$0") [OPCIONES]

Opciones:
  -o FILE    Archivo de salida (default: ${REPORT_DIR}/inventory_<host>_<fecha>.txt)
  -v         Modo verbose (muestra información detallada)
  -h         Muestra esta ayuda

Ejemplo:
  $(basename "$0") -o /tmp/mi_inventario.txt -v
USAGE
}

#-------------------------------------------------------------------------------
# Función: get_cpu_info
#-------------------------------------------------------------------------------
get_cpu_info() {
    echo "--- INFORMACIÓN DE CPU ---"
    echo "Modelo: $(grep -m1 'model name' /proc/cpuinfo | cut -d: -f2 | xargs)"
    echo "Núcleos físicos: $(grep -c '^processor' /proc/cpuinfo)"
    echo "Arquitectura: $(uname -m)"
    echo "Frecuencia: $(grep -m1 'cpu MHz' /proc/cpuinfo | cut -d: -f2 | xargs) MHz"
    echo ""
}

#-------------------------------------------------------------------------------
# Función: get_memory_info
#-------------------------------------------------------------------------------
get_memory_info() {
    echo "--- INFORMACIÓN DE MEMORIA ---"
    local mem_total mem_available mem_used_pct
    mem_total=$(awk '/MemTotal/ {printf "%.2f GB", $2/1024/1024}' /proc/meminfo)
    mem_available=$(awk '/MemAvailable/ {printf "%.2f GB", $2/1024/1024}' /proc/meminfo)
    mem_used_pct=$(free | awk '/Mem:/ {printf "%.1f%%", ($3/$2)*100}')
    
    echo "Total: ${mem_total}"
    echo "Disponible: ${mem_available}"
    echo "Uso actual: ${mem_used_pct}"
    echo "Swap total: $(awk '/SwapTotal/ {printf "%.2f GB", $2/1024/1024}' /proc/meminfo)"
    echo ""
}

#-------------------------------------------------------------------------------
# Función: get_disk_info
#-------------------------------------------------------------------------------
get_disk_info() {
    echo "--- INFORMACIÓN DE ALMACENAMIENTO ---"
    echo "Discos detectados:"
    lsblk -d -o NAME,SIZE,TYPE,MODEL 2>/dev/null | head -20
    echo ""
    echo "Uso de sistemas de archivos:"
    df -hT --exclude-type=tmpfs --exclude-type=devtmpfs 2>/dev/null
    echo ""
}

#-------------------------------------------------------------------------------
# Función: get_network_info
#-------------------------------------------------------------------------------
get_network_info() {
    echo "--- INFORMACIÓN DE RED ---"
    echo "Hostname: $(hostname -f 2>/dev/null || hostname)"
    echo "Interfaces activas:"
    ip -4 addr show up | awk '/inet / {print "  " $NF ": " $2}'
    echo ""
    echo "Gateway por defecto: $(ip route | awk '/default/ {print $3}' | head -1)"
    echo "DNS configurado:"
    grep -v '^#' /etc/resolv.conf 2>/dev/null | grep nameserver | awk '{print "  " $2}'
    echo ""
}

#-------------------------------------------------------------------------------
# Función: get_os_info
#-------------------------------------------------------------------------------
get_os_info() {
    echo "--- INFORMACIÓN DEL SISTEMA OPERATIVO ---"
    echo "Distribución: $(cat /etc/os-release | grep ^PRETTY_NAME | cut -d= -f2 | tr -d '"')"
    echo "Kernel: $(uname -r)"
    echo "Uptime: $(uptime -p)"
    echo "Último arranque: $(who -b | awk '{print $3, $4}')"
    echo "Zona horaria: $(timedatectl show -p Timezone --value 2>/dev/null || cat /etc/timezone)"
    echo ""
}

#-------------------------------------------------------------------------------
# Función: get_software_info
#-------------------------------------------------------------------------------
get_software_info() {
    echo "--- SOFTWARE INSTALADO (Resumen) ---"
    echo "Paquetes instalados: $(dpkg -l 2>/dev/null | grep ^ii | wc -l)"
    echo "Shell: ${SHELL} ($(bash --version | head -1 | awk '{print $4}'))"
    echo ""
    echo "Servicios activos (principales):"
    systemctl list-units --type=service --state=running --no-pager --no-legend 2>/dev/null | \
        awk '{print "  " $1}' | head -15
    echo ""
}

#-------------------------------------------------------------------------------
# PROGRAMA PRINCIPAL
#-------------------------------------------------------------------------------

# Procesar opciones con getopts
while getopts ":o:vh" opt; do
    case ${opt} in
        o) OUTPUT_FILE="${OPTARG}" ;;
        v) VERBOSE=1 ;;
        h) show_usage; exit 0 ;;
        :) log_message "ERROR" "La opción -${OPTARG} requiere un argumento"; exit 1 ;;
        \?) log_message "ERROR" "Opción inválida: -${OPTARG}"; show_usage; exit 1 ;;
    esac
done

log_message "INFO" "Iniciando inventario del sistema..."

# Verificar dependencias
check_dependencies "lsblk" "ip" "awk" "systemctl"

# Generar reporte
{
    generate_report_header "INVENTARIO DEL SISTEMA"
    get_os_info
    get_cpu_info
    get_memory_info
    get_disk_info
    get_network_info
    get_software_info
    echo "======================================================================="
    echo "  Fin del reporte - Generado en $(date '+%Y-%m-%d %H:%M:%S')"
    echo "======================================================================="
} > "${OUTPUT_FILE}"

log_message "INFO" "Reporte generado: ${OUTPUT_FILE}"

# Mostrar resumen si es verbose
if [[ ${VERBOSE} -eq 1 ]]; then
    cat "${OUTPUT_FILE}"
fi

echo "Reporte guardado en: ${OUTPUT_FILE}"
SCRIPT

chmod +x /opt/scripts/system_inventory.sh
```

2. Ejecutar el script:

```bash
/opt/scripts/system_inventory.sh -v
```

3. Verificar el reporte generado:

```bash
ls -la /opt/sysreport/inventory_*.txt
head -30 /opt/sysreport/inventory_*.txt | tail -25
```

### Salida Esperada

```
[INFO] Iniciando inventario del sistema...
[INFO] Reporte generado: /opt/sysreport/inventory_srv-linux-01_20240115_143022.txt
[INFO] Script finalizado exitosamente.
======================================================================
  INVENTARIO DEL SISTEMA
  Servidor: srv-linux-01
  Fecha: 2024-01-15 14:30:22
  Usuario: sysadmin
======================================================================

--- INFORMACIÓN DEL SISTEMA OPERATIVO ---
Distribución: Ubuntu 22.04.4 LTS
Kernel: 5.15.0-91-generic
...
```

### Verificación

```bash
# Verificar que el log registró la ejecución
grep "system_inventory" /var/log/admin_scripts.log | tail -3

# Probar depuración con bash -x (solo primeras líneas)
bash -x /opt/scripts/system_inventory.sh -o /tmp/test_inv.txt 2>&1 | head -20
```

---

## Paso 3: Script 2 — service_monitor.sh (Monitor de Servicios)

### Objetivo
Desarrollar un script que valide el estado de servicios críticos del sistema y genere alertas cuando alguno no esté funcionando correctamente.

### Instrucciones

1. Crear el script de monitoreo:

```bash
cat > /opt/scripts/service_monitor.sh << 'SCRIPT'
#!/bin/bash
#===============================================================================
# Script: service_monitor.sh
# Descripción: Monitorea servicios críticos y recursos del sistema
# Uso: ./service_monitor.sh [-s servicio1,servicio2] [-c cpu_umbral] [-m mem_umbral] [-d disk_umbral] [-h]
# Versión: 1.0
#===============================================================================

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/common_lib.sh"

trap cleanup_on_exit EXIT

# Umbrales por defecto (porcentaje)
CPU_THRESHOLD=80
MEM_THRESHOLD=85
DISK_THRESHOLD=90

# Servicios a monitorear por defecto
DEFAULT_SERVICES="ssh,cron,systemd-resolved"
SERVICES_LIST=""
REPORT_ONLY=0

#-------------------------------------------------------------------------------
# Función: show_usage
#-------------------------------------------------------------------------------
show_usage() {
    cat << USAGE
Uso: $(basename "$0") [OPCIONES]

Opciones:
  -s LISTA   Lista de servicios separados por coma (default: ${DEFAULT_SERVICES})
  -c NUM     Umbral de CPU en porcentaje (default: ${CPU_THRESHOLD})
  -m NUM     Umbral de memoria en porcentaje (default: ${MEM_THRESHOLD})
  -d NUM     Umbral de disco en porcentaje (default: ${DISK_THRESHOLD})
  -r         Solo reportar, no generar alertas
  -h         Muestra esta ayuda

Ejemplo:
  $(basename "$0") -s ssh,mysql,cron -c 75 -m 80 -d 85
USAGE
}

#-------------------------------------------------------------------------------
# Función: check_service_status
# Parámetros: $1 = nombre del servicio
# Retorno: 0 si activo, 1 si inactivo
#-------------------------------------------------------------------------------
check_service_status() {
    local service="${1}"
    local status
    
    if systemctl is-active --quiet "${service}" 2>/dev/null; then
        status="active"
        log_message "DEBUG" "Servicio ${service}: ACTIVO"
        printf "  %-30s [${GREEN}ACTIVO${NC}]\n" "${service}"
        return 0
    else
        status="inactive"
        log_message "WARN" "Servicio ${service}: INACTIVO o NO ENCONTRADO"
        printf "  %-30s [${RED}INACTIVO${NC}]\n" "${service}"
        send_alert "critical" "Servicio ${service} NO está activo en $(hostname)"
        return 1
    fi
}

#-------------------------------------------------------------------------------
# Función: check_cpu_usage
#-------------------------------------------------------------------------------
check_cpu_usage() {
    local cpu_usage
    # Obtener uso de CPU promedio (1 segundo de muestreo)
    cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print int($2 + $4)}')
    
    echo "--- MONITOREO DE CPU ---"
    echo "  Uso actual: ${cpu_usage}%"
    echo "  Umbral configurado: ${CPU_THRESHOLD}%"
    
    if [[ ${cpu_usage} -ge ${CPU_THRESHOLD} ]]; then
        echo -e "  Estado: [${RED}ALERTA${NC}] - Uso de CPU excede el umbral"
        send_alert "warning" "CPU al ${cpu_usage}% (umbral: ${CPU_THRESHOLD}%)"
        return 1
    else
        echo -e "  Estado: [${GREEN}OK${NC}]"
        return 0
    fi
}

#-------------------------------------------------------------------------------
# Función: check_memory_usage
#-------------------------------------------------------------------------------
check_memory_usage() {
    local mem_usage
    mem_usage=$(free | awk '/Mem:/ {printf "%d", ($3/$2)*100}')
    
    echo ""
    echo "--- MONITOREO DE MEMORIA ---"
    echo "  Uso actual: ${mem_usage}%"
    echo "  Umbral configurado: ${MEM_THRESHOLD}%"
    
    if [[ ${mem_usage} -ge ${MEM_THRESHOLD} ]]; then
        echo -e "  Estado: [${RED}ALERTA${NC}] - Uso de memoria excede el umbral"
        send_alert "warning" "Memoria al ${mem_usage}% (umbral: ${MEM_THRESHOLD}%)"
        return 1
    else
        echo -e "  Estado: [${GREEN}OK${NC}]"
        return 0
    fi
}

#-------------------------------------------------------------------------------
# Función: check_disk_usage
#-------------------------------------------------------------------------------
check_disk_usage() {
    local alerts=0
    
    echo ""
    echo "--- MONITOREO DE DISCO ---"
    echo "  Umbral configurado: ${DISK_THRESHOLD}%"
    echo ""
    
    while IFS= read -r line; do
        local usage mount_point
        usage=$(echo "${line}" | awk '{print $5}' | tr -d '%')
        mount_point=$(echo "${line}" | awk '{print $6}')
        
        if [[ ${usage} -ge ${DISK_THRESHOLD} ]]; then
            printf "  %-20s %3d%% [${RED}ALERTA${NC}]\n" "${mount_point}" "${usage}"
            send_alert "critical" "Disco ${mount_point} al ${usage}% (umbral: ${DISK_THRESHOLD}%)"
            alerts=$((alerts + 1))
        else
            printf "  %-20s %3d%% [${GREEN}OK${NC}]\n" "${mount_point}" "${usage}"
        fi
    done < <(df --output=pcent,target --exclude-type=tmpfs --exclude-type=devtmpfs 2>/dev/null | tail -n +2)
    
    return ${alerts}
}

#-------------------------------------------------------------------------------
# Función: check_services
#-------------------------------------------------------------------------------
check_services() {
    local failed=0
    
    echo ""
    echo "--- MONITOREO DE SERVICIOS ---"
    
    IFS=',' read -ra services <<< "${SERVICES_LIST}"
    for service in "${services[@]}"; do
        # Eliminar espacios
        service=$(echo "${service}" | xargs)
        if ! check_service_status "${service}"; then
            failed=$((failed + 1))
        fi
    done
    
    echo ""
    echo "  Total servicios verificados: ${#services[@]}"
    echo "  Servicios con problemas: ${failed}"
    
    return ${failed}
}

#-------------------------------------------------------------------------------
# PROGRAMA PRINCIPAL
#-------------------------------------------------------------------------------

while getopts ":s:c:m:d:rh" opt; do
    case ${opt} in
        s) SERVICES_LIST="${OPTARG}" ;;
        c) CPU_THRESHOLD="${OPTARG}" ;;
        m) MEM_THRESHOLD="${OPTARG}" ;;
        d) DISK_THRESHOLD="${OPTARG}" ;;
        r) REPORT_ONLY=1 ;;
        h) show_usage; exit 0 ;;
        :) log_message "ERROR" "La opción -${OPTARG} requiere un argumento"; exit 1 ;;
        \?) log_message "ERROR" "Opción inválida: -${OPTARG}"; exit 1 ;;
    esac
done

# Usar servicios por defecto si no se especificaron
[[ -z "${SERVICES_LIST}" ]] && SERVICES_LIST="${DEFAULT_SERVICES}"

log_message "INFO" "Iniciando monitoreo del sistema..."

# Generar encabezado
generate_report_header "MONITOREO DEL SISTEMA"

# Ejecutar verificaciones
total_alerts=0

check_cpu_usage || total_alerts=$((total_alerts + 1))
check_memory_usage || total_alerts=$((total_alerts + 1))
check_disk_usage || total_alerts=$((total_alerts + $?))
check_services || total_alerts=$((total_alerts + $?))

# Resumen final
echo ""
echo "======================================================================="
if [[ ${total_alerts} -eq 0 ]]; then
    echo -e "  RESULTADO: [${GREEN}TODOS LOS SISTEMAS OPERATIVOS NORMALES${NC}]"
    log_message "INFO" "Monitoreo completado: sin alertas"
else
    echo -e "  RESULTADO: [${YELLOW}${total_alerts} ALERTA(S) DETECTADA(S)${NC}]"
    log_message "WARN" "Monitoreo completado: ${total_alerts} alerta(s)"
fi
echo "======================================================================="

exit 0
SCRIPT

chmod +x /opt/scripts/service_monitor.sh
```

2. Ejecutar el script con parámetros personalizados:

```bash
/opt/scripts/service_monitor.sh -s ssh,cron,systemd-resolved -c 90 -m 90 -d 95
```

3. Probar con un servicio inexistente para validar alertas:

```bash
/opt/scripts/service_monitor.sh -s ssh,servicio_falso,cron
```

### Salida Esperada

```
======================================================================
  MONITOREO DEL SISTEMA
  Servidor: srv-linux-01
  Fecha: 2024-01-15 14:35:10
  Usuario: sysadmin
======================================================================

--- MONITOREO DE CPU ---
  Uso actual: 12%
  Umbral configurado: 90%
  Estado: [OK]

--- MONITOREO DE MEMORIA ---
  Uso actual: 45%
  Umbral configurado: 90%
  Estado: [OK]

--- MONITOREO DE DISCO ---
  /                    35% [OK]
  /boot                22% [OK]

--- MONITOREO DE SERVICIOS ---
  ssh                            [ACTIVO]
  servicio_falso                 [INACTIVO]
  cron                           [ACTIVO]

  Total servicios verificados: 3
  Servicios con problemas: 1

=======================================================================
  RESULTADO: [1 ALERTA(S) DETECTADA(S)]
=======================================================================
```

### Verificación

```bash
# Verificar alertas generadas
cat /opt/sysreport/alerts_$(date '+%Y%m%d').log

# Verificar log centralizado
grep "service_monitor" /var/log/admin_scripts.log | tail -5
```

---

## Paso 4: Script 3 — storage_report.sh (Reporte de Almacenamiento)

### Objetivo
Crear un script que reporte el estado del almacenamiento incluyendo LVM, RAID (si existe) y uso de sistemas de archivos.

### Instrucciones

1. Crear el script de reporte de almacenamiento:

```bash
cat > /opt/scripts/storage_report.sh << 'SCRIPT'
#!/bin/bash
#===============================================================================
# Script: storage_report.sh
# Descripción: Genera reporte detallado de almacenamiento (LVM, RAID, cuotas)
# Uso: ./storage_report.sh [-o archivo] [-f formato] [-h]
# Versión: 1.0
#===============================================================================

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/common_lib.sh"

trap cleanup_on_exit EXIT

OUTPUT_FILE="${REPORT_DIR}/storage_$(hostname)_$(date '+%Y%m%d_%H%M%S').txt"
FORMAT="text"

#-------------------------------------------------------------------------------
# Función: show_usage
#-------------------------------------------------------------------------------
show_usage() {
    cat << USAGE
Uso: $(basename "$0") [OPCIONES]

Opciones:
  -o FILE    Archivo de salida
  -f FORMAT  Formato de salida: text (default), json
  -h         Muestra esta ayuda
USAGE
}

#-------------------------------------------------------------------------------
# Función: report_physical_disks
#-------------------------------------------------------------------------------
report_physical_disks() {
    echo "=== DISCOS FÍSICOS ==="
    echo ""
    lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL 2>/dev/null | grep -E "^(NAME|[a-z])"
    echo ""
    
    echo "Resumen de capacidad:"
    local total_size
    total_size=$(lsblk -bdn -o SIZE 2>/dev/null | awk '{sum+=$1} END {printf "%.2f GB", sum/1024/1024/1024}')
    echo "  Capacidad total de discos: ${total_size}"
    echo ""
}

#-------------------------------------------------------------------------------
# Función: report_lvm_status
#-------------------------------------------------------------------------------
report_lvm_status() {
    echo "=== ESTADO DE LVM ==="
    echo ""
    
    # Verificar si LVM está disponible
    if ! command -v pvs &>/dev/null; then
        echo "  LVM no está instalado en este sistema."
        echo ""
        return 0
    fi
    
    echo "--- Volúmenes Físicos (PV) ---"
    if sudo pvs 2>/dev/null | grep -q "/dev/"; then
        sudo pvs --noheadings -o pv_name,vg_name,pv_size,pv_free 2>/dev/null | \
            awk '{printf "  PV: %-15s VG: %-15s Tamaño: %-10s Libre: %s\n", $1, $2, $3, $4}'
    else
        echo "  No se encontraron volúmenes físicos LVM."
    fi
    echo ""
    
    echo "--- Grupos de Volúmenes (VG) ---"
    if sudo vgs 2>/dev/null | grep -q "[a-z]"; then
        sudo vgs --noheadings -o vg_name,vg_size,vg_free,lv_count,pv_count 2>/dev/null | \
            awk '{printf "  VG: %-15s Tamaño: %-10s Libre: %-10s LVs: %s PVs: %s\n", $1, $2, $3, $4, $5}'
    else
        echo "  No se encontraron grupos de volúmenes."
    fi
    echo ""
    
    echo "--- Volúmenes Lógicos (LV) ---"
    if sudo lvs 2>/dev/null | grep -q "[a-z]"; then
        sudo lvs --noheadings -o lv_name,vg_name,lv_size,lv_attr 2>/dev/null | \
            awk '{printf "  LV: %-15s VG: %-15s Tamaño: %-10s Atributos: %s\n", $1, $2, $3, $4}'
    else
        echo "  No se encontraron volúmenes lógicos."
    fi
    echo ""
}

#-------------------------------------------------------------------------------
# Función: report_raid_status
#-------------------------------------------------------------------------------
report_raid_status() {
    echo "=== ESTADO DE RAID ==="
    echo ""
    
    if [[ -f /proc/mdstat ]]; then
        local raid_devices
        raid_devices=$(grep "^md" /proc/mdstat 2>/dev/null | wc -l)
        
        if [[ ${raid_devices} -gt 0 ]]; then
            echo "  Dispositivos RAID detectados: ${raid_devices}"
            echo ""
            cat /proc/mdstat
            echo ""
            
            # Detalle de cada dispositivo RAID
            for md in $(grep "^md" /proc/mdstat | awk '{print $1}'); do
                if command -v mdadm &>/dev/null; then
                    echo "  Detalle de /dev/${md}:"
                    sudo mdadm --detail "/dev/${md}" 2>/dev/null | \
                        grep -E "(Raid Level|Array Size|State|Active Devices|Failed Devices)" | \
                        sed 's/^/    /'
                    echo ""
                fi
            done
        else
            echo "  No se detectaron dispositivos RAID activos."
        fi
    else
        echo "  /proc/mdstat no disponible. RAID por software no configurado."
    fi
    echo ""
}

#-------------------------------------------------------------------------------
# Función: report_filesystem_usage
#-------------------------------------------------------------------------------
report_filesystem_usage() {
    echo "=== USO DE SISTEMAS DE ARCHIVOS ==="
    echo ""
    
    printf "  %-20s %-8s %-10s %-10s %-6s %s\n" \
        "DISPOSITIVO" "TIPO" "TAMAÑO" "USADO" "USO%" "MONTAJE"
    printf "  %-20s %-8s %-10s %-10s %-6s %s\n" \
        "-------------------" "-------" "---------" "---------" "-----" "-------"
    
    df -hT --exclude-type=tmpfs --exclude-type=devtmpfs --exclude-type=squashfs 2>/dev/null | \
        tail -n +2 | sort -k6 -n -r | \
        awk '{printf "  %-20s %-8s %-10s %-10s %-6s %s\n", $1, $2, $3, $4, $6, $7}'
    echo ""
    
    # Inodos
    echo "--- Uso de Inodos ---"
    df -i --exclude-type=tmpfs --exclude-type=devtmpfs --exclude-type=squashfs 2>/dev/null | \
        tail -n +2 | awk '$5+0 > 50 {printf "  %-20s Inodos usados: %s\n", $6, $5}'
    echo ""
}

#-------------------------------------------------------------------------------
# Función: report_quotas
#-------------------------------------------------------------------------------
report_quotas() {
    echo "=== CUOTAS DE DISCO ==="
    echo ""
    
    if command -v repquota &>/dev/null; then
        if sudo repquota -a 2>/dev/null | grep -q "[a-z]"; then
            sudo repquota -a 2>/dev/null | head -30
        else
            echo "  No hay cuotas configuradas o activas."
        fi
    else
        echo "  Herramientas de cuota no instaladas."
    fi
    echo ""
}

#-------------------------------------------------------------------------------
# PROGRAMA PRINCIPAL
#-------------------------------------------------------------------------------

while getopts ":o:f:h" opt; do
    case ${opt} in
        o) OUTPUT_FILE="${OPTARG}" ;;
        f) FORMAT="${OPTARG}" ;;
        h) show_usage; exit 0 ;;
        :) log_message "ERROR" "La opción -${OPTARG} requiere un argumento"; exit 1 ;;
        \?) log_message "ERROR" "Opción inválida: -${OPTARG}"; exit 1 ;;
    esac
done

log_message "INFO" "Generando reporte de almacenamiento..."

{
    generate_report_header "REPORTE DE ALMACENAMIENTO"
    report_physical_disks
    report_lvm_status
    report_raid_status
    report_filesystem_usage
    report_quotas
    echo "======================================================================="
    echo "  Fin del reporte de almacenamiento"
    echo "======================================================================="
} | tee "${OUTPUT_FILE}"

log_message "INFO" "Reporte guardado en: ${OUTPUT_FILE}"
SCRIPT

chmod +x /opt/scripts/storage_report.sh
```

2. Ejecutar el script (requiere sudo para LVM/RAID):

```bash
sudo /opt/scripts/storage_report.sh
```

### Salida Esperada

```
======================================================================
  REPORTE DE ALMACENAMIENTO
  Servidor: srv-linux-01
  Fecha: 2024-01-15 14:40:55
  Usuario: root
======================================================================

=== DISCOS FÍSICOS ===

NAME   SIZE TYPE FSTYPE MOUNTPOINT MODEL
sda     40G disk                   VBOX HARDDISK
├─sda1   1G part ext4   /boot
├─sda2  39G part LVM2_m
...
```

### Verificación

```bash
ls -la /opt/sysreport/storage_*.txt
wc -l /opt/sysreport/storage_*.txt
```

---

## Paso 5: Script 4 — log_analyzer.sh (Analizador de Logs)

### Objetivo
Desarrollar un script que procese logs del sistema utilizando grep, awk y sed para extraer información relevante y generar estadísticas.

### Instrucciones

1. Crear el script analizador de logs:

```bash
cat > /opt/scripts/log_analyzer.sh << 'SCRIPT'
#!/bin/bash
#===============================================================================
# Script: log_analyzer.sh
# Descripción: Analiza logs del sistema y genera estadísticas
# Uso: ./log_analyzer.sh [-l logfile] [-n lineas] [-p patron] [-t periodo] [-h]
# Versión: 1.0
#===============================================================================

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/common_lib.sh"

trap cleanup_on_exit EXIT

# Variables por defecto
LOG_SOURCE="/var/log/syslog"
NUM_LINES=1000
PATTERN=""
TIME_PERIOD="today"
OUTPUT_FILE="${REPORT_DIR}/log_analysis_$(date '+%Y%m%d_%H%M%S').txt"

#-------------------------------------------------------------------------------
# Función: show_usage
#-------------------------------------------------------------------------------
show_usage() {
    cat << USAGE
Uso: $(basename "$0") [OPCIONES]

Opciones:
  -l FILE      Archivo de log a analizar (default: /var/log/syslog)
  -n NUM       Número de líneas a procesar (default: 1000)
  -p PATRON    Patrón de búsqueda específico
  -t PERIODO   Período: today, yesterday, week (default: today)
  -o FILE      Archivo de salida del reporte
  -h           Muestra esta ayuda

Ejemplo:
  $(basename "$0") -l /var/log/auth.log -p "Failed" -n 500
USAGE
}

#-------------------------------------------------------------------------------
# Función: analyze_error_frequency
# Parámetros: $1 = archivo de log
#-------------------------------------------------------------------------------
analyze_error_frequency() {
    local logfile="${1}"
    
    echo "=== FRECUENCIA DE ERRORES ==="
    echo ""
    echo "Top 10 mensajes de error más frecuentes:"
    echo ""
    
    grep -i "error\|fail\|critical\|warning" "${logfile}" 2>/dev/null | \
        sed 's/[0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}T[0-9:+]*//g' | \
        sed 's/[A-Z][a-z]\{2\} [ 0-9]\{2\} [0-9:]*//g' | \
        sort | uniq -c | sort -rn | head -10 | \
        awk '{printf "  %5d veces: %s\n", $1, substr($0, index($0,$2))}'
    echo ""
}

#-------------------------------------------------------------------------------
# Función: analyze_service_events
# Parámetros: $1 = archivo de log
#-------------------------------------------------------------------------------
analyze_service_events() {
    local logfile="${1}"
    
    echo "=== EVENTOS DE SERVICIOS ==="
    echo ""
    
    echo "Servicios con más actividad:"
    grep -oP '(?<=]: )\S+' "${logfile}" 2>/dev/null | \
        sort | uniq -c | sort -rn | head -10 | \
        awk '{printf "  %5d eventos: %s\n", $1, $2}'
    echo ""
    
    echo "Servicios reiniciados:"
    grep -i "start\|stop\|restart" "${logfile}" 2>/dev/null | \
        grep -oP '\S+\.service' | sort -u | \
        awk '{printf "  - %s\n", $1}' | head -15
    echo ""
}

#-------------------------------------------------------------------------------
# Función: analyze_auth_events
#-------------------------------------------------------------------------------
analyze_auth_events() {
    local auth_log="/var/log/auth.log"
    
    echo "=== EVENTOS DE AUTENTICACIÓN ==="
    echo ""
    
    if [[ ! -r "${auth_log}" ]]; then
        echo "  No se puede leer ${auth_log} (permisos insuficientes)"
        return 0
    fi
    
    # Intentos de login fallidos
    local failed_logins
    failed_logins=$(grep -c "Failed password\|authentication failure" "${auth_log}" 2>/dev/null || echo "0")
    echo "  Intentos de login fallidos: ${failed_logins}"
    
    # Sesiones SSH abiertas
    local ssh_sessions
    ssh_sessions=$(grep -c "Accepted" "${auth_log}" 2>/dev/null || echo "0")
    echo "  Sesiones SSH aceptadas: ${ssh_sessions}"
    
    # Uso de sudo
    local sudo_usage
    sudo_usage=$(grep -c "sudo:" "${auth_log}" 2>/dev/null || echo "0")
    echo "  Comandos sudo ejecutados: ${sudo_usage}"
    
    echo ""
    
    # IPs con más intentos fallidos
    if [[ ${failed_logins} -gt 0 ]]; then
        echo "  Top IPs con intentos fallidos:"
        grep "Failed password" "${auth_log}" 2>/dev/null | \
            grep -oP '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}' | \
            sort | uniq -c | sort -rn | head -5 | \
            awk '{printf "    %5d intentos desde: %s\n", $1, $2}'
        echo ""
    fi
}

#-------------------------------------------------------------------------------
# Función: analyze_time_distribution
# Parámetros: $1 = archivo de log
#-------------------------------------------------------------------------------
analyze_time_distribution() {
    local logfile="${1}"
    
    echo "=== DISTRIBUCIÓN TEMPORAL DE EVENTOS ==="
    echo ""
    echo "Eventos por hora (últimas 24h):"
    echo ""
    
    # Extraer horas y contar eventos
    grep -oP '\d{2}(?=:\d{2}:\d{2})' "${logfile}" 2>/dev/null | \
        sort | uniq -c | \
        awk '{
            bar = "";
            for (i = 0; i < int($1/10); i++) bar = bar "#";
            printf "  %02d:00 | %5d | %s\n", $2, $1, bar
        }'
    echo ""
}

#-------------------------------------------------------------------------------
# Función: search_pattern
# Parámetros: $1 = archivo, $2 = patrón
#-------------------------------------------------------------------------------
search_pattern() {
    local logfile="${1}"
    local pattern="${2}"
    
    echo "=== BÚSQUEDA DE PATRÓN: '${pattern}' ==="
    echo ""
    
    local count
    count=$(grep -c "${pattern}" "${logfile}" 2>/dev/null || echo "0")
    echo "  Coincidencias encontradas: ${count}"
    echo ""
    
    if [[ ${count} -gt 0 ]]; then
        echo "  Últimas 10 coincidencias:"
        grep "${pattern}" "${logfile}" 2>/dev/null | tail -10 | \
            sed 's/^/    /'
        echo ""
    fi
}

#-------------------------------------------------------------------------------
# PROGRAMA PRINCIPAL
#-------------------------------------------------------------------------------

while getopts ":l:n:p:t:o:h" opt; do
    case ${opt} in
        l) LOG_SOURCE="${OPTARG}" ;;
        n) NUM_LINES="${OPTARG}" ;;
        p) PATTERN="${OPTARG}" ;;
        t) TIME_PERIOD="${OPTARG}" ;;
        o) OUTPUT_FILE="${OPTARG}" ;;
        h) show_usage; exit 0 ;;
        :) log_message "ERROR" "La opción -${OPTARG} requiere un argumento"; exit 1 ;;
        \?) log_message "ERROR" "Opción inválida: -${OPTARG}"; exit 1 ;;
    esac
done

# Validar que el archivo de log existe
if [[ ! -f "${LOG_SOURCE}" ]]; then
    log_message "ERROR" "Archivo de log no encontrado: ${LOG_SOURCE}"
    exit 1
fi

log_message "INFO" "Analizando log: ${LOG_SOURCE} (últimas ${NUM_LINES} líneas)"

# Crear archivo temporal con las líneas a analizar
TEMP_LOG=$(mktemp)
trap 'rm -f "${TEMP_LOG}"; cleanup_on_exit' EXIT

tail -n "${NUM_LINES}" "${LOG_SOURCE}" > "${TEMP_LOG}"

# Generar análisis
{
    generate_report_header "ANÁLISIS DE LOGS"
    echo "Archivo analizado: ${LOG_SOURCE}"
    echo "Líneas procesadas: ${NUM_LINES}"
    echo "Período: ${TIME_PERIOD}"
    echo ""
    
    analyze_error_frequency "${TEMP_LOG}"
    analyze_service_events "${TEMP_LOG}"
    analyze_auth_events
    analyze_time_distribution "${TEMP_LOG}"
    
    # Búsqueda de patrón si se especificó
    if [[ -n "${PATTERN}" ]]; then
        search_pattern "${TEMP_LOG}" "${PATTERN}"
    fi
    
    echo "======================================================================="
    echo "  Fin del análisis de logs"
    echo "======================================================================="
} | tee "${OUTPUT_FILE}"

log_message "INFO" "Análisis guardado en: ${OUTPUT_FILE}"
SCRIPT

chmod +x /opt/scripts/log_analyzer.sh
```

2. Ejecutar el análisis de logs:

```bash
sudo /opt/scripts/log_analyzer.sh -l /var/log/syslog -n 500 -p "error"
```

3. Analizar logs de autenticación:

```bash
sudo /opt/scripts/log_analyzer.sh -l /var/log/auth.log -n 200 -p "Failed"
```

### Salida Esperada

```
======================================================================
  ANÁLISIS DE LOGS
  Servidor: srv-linux-01
  Fecha: 2024-01-15 14:50:33
  Usuario: root
======================================================================

Archivo analizado: /var/log/syslog
Líneas procesadas: 500
Período: today

=== FRECUENCIA DE ERRORES ===

Top 10 mensajes de error más frecuentes:

     12 veces: systemd[1]: Failed to start...
      5 veces: kernel: [error] ...
...
```

### Verificación

```bash
ls -la /opt/sysreport/log_analysis_*.txt
grep "log_analyzer" /var/log/admin_scripts.log | tail -3
```

---

## Paso 6: Script 5 — db_report.sh (Reporte de Base de Datos)

### Objetivo
Crear un script que conecte a MySQL 8.0.36 en srv-linux-02 y genere reportes del inventario de servidores, con salida en múltiples formatos.

### Instrucciones

1. Crear el script de reportes de base de datos:

```bash
cat > /opt/scripts/db_report.sh << 'SCRIPT'
#!/bin/bash
#===============================================================================
# Script: db_report.sh
# Descripción: Genera reportes desde la base de datos srv_inventory en MySQL
# Uso: ./db_report.sh [-f formato] [-s estado] [-q consulta] [-h]
# Versión: 1.0
#===============================================================================

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/common_lib.sh"

trap cleanup_on_exit EXIT

# Configuración de base de datos
DB_HOST="192.168.100.20"
DB_PORT="3306"
DB_USER="dbuser_scripts"
DB_PASS="DB@Scripts2024!"
DB_NAME="srv_inventory"

# Variables
OUTPUT_FORMAT="text"
FILTER_STATUS=""
CUSTOM_QUERY=""
OUTPUT_FILE="${REPORT_DIR}/db_report_$(date '+%Y%m%d_%H%M%S').txt"

#-------------------------------------------------------------------------------
# Función: show_usage
#-------------------------------------------------------------------------------
show_usage() {
    cat << USAGE
Uso: $(basename "$0") [OPCIONES]

Opciones:
  -f FORMAT   Formato de salida: text (default), json, csv
  -s STATUS   Filtrar por estado: active, inactive, maintenance
  -q QUERY    Consulta SQL personalizada
  -o FILE     Archivo de salida
  -h          Muestra esta ayuda

Ejemplo:
  $(basename "$0") -f json -s active
  $(basename "$0") -q "SELECT hostname, ip FROM servers WHERE status='active'"
USAGE
}

#-------------------------------------------------------------------------------
# Función: test_db_connection
#-------------------------------------------------------------------------------
test_db_connection() {
    log_message "DEBUG" "Probando conexión a MySQL ${DB_HOST}:${DB_PORT}..."
    
    if ! mysql -h "${DB_HOST}" -P "${DB_PORT}" -u "${DB_USER}" -p"${DB_PASS}" \
         -e "SELECT 1;" "${DB_NAME}" &>/dev/null; then
        log_message "ERROR" "No se puede conectar a MySQL en ${DB_HOST}:${DB_PORT}"
        echo "Error: Verifique la conectividad y credenciales de MySQL."
        exit 1
    fi
    
    log_message "INFO" "Conexión a MySQL exitosa."
}

#-------------------------------------------------------------------------------
# Función: execute_query
# Parámetros: $1 = consulta SQL
# Retorno: Resultado de la consulta en stdout
#-------------------------------------------------------------------------------
execute_query() {
    local query="${1}"
    
    mysql -h "${DB_HOST}" -P "${DB_PORT}" -u "${DB_USER}" -p"${DB_PASS}" \
          --batch --skip-column-names "${DB_NAME}" -e "${query}" 2>/dev/null
}

#-------------------------------------------------------------------------------
# Función: execute_query_with_headers
# Parámetros: $1 = consulta SQL
#-------------------------------------------------------------------------------
execute_query_with_headers() {
    local query="${1}"
    
    mysql -h "${DB_HOST}" -P "${DB_PORT}" -u "${DB_USER}" -p"${DB_PASS}" \
          --table "${DB_NAME}" -e "${query}" 2>/dev/null
}

#-------------------------------------------------------------------------------
# Función: report_server_inventory
#-------------------------------------------------------------------------------
report_server_inventory() {
    local where_clause=""
    
    if [[ -n "${FILTER_STATUS}" ]]; then
        where_clause="WHERE status='${FILTER_STATUS}'"
    fi
    
    echo "=== INVENTARIO DE SERVIDORES ==="
    echo ""
    
    case "${OUTPUT_FORMAT}" in
        text)
            execute_query_with_headers "SELECT id, hostname, ip, os, status, created_at FROM servers ${where_clause} ORDER BY hostname;"
            ;;
        csv)
            echo "id,hostname,ip,os,status,created_at"
            execute_query "SELECT id, hostname, ip, os, status, created_at FROM servers ${where_clause} ORDER BY hostname;" | \
                sed 's/\t/,/g'
            ;;
        json)
            echo "["
            local first=1
            while IFS=$'\t' read -r id hostname ip os status created_at; do
                [[ ${first} -eq 0 ]] && echo ","
                first=0
                cat << JSONENTRY
  {
    "id": ${id},
    "hostname": "${hostname}",
    "ip": "${ip}",
    "os": "${os}",
    "status": "${status}",
    "created_at": "${created_at}"
  }
JSONENTRY
            done < <(execute_query "SELECT id, hostname, ip, os, status, created_at FROM servers ${where_clause} ORDER BY hostname;")
            echo ""
            echo "]"
            ;;
    esac
    echo ""
}

#-------------------------------------------------------------------------------
# Función: report_statistics
#-------------------------------------------------------------------------------
report_statistics() {
    echo "=== ESTADÍSTICAS DEL INVENTARIO ==="
    echo ""
    
    # Total de servidores
    local total
    total=$(execute_query "SELECT COUNT(*) FROM servers;")
    echo "  Total de servidores registrados: ${total}"
    echo ""
    
    # Por estado
    echo "  Distribución por estado:"
    while IFS=$'\t' read -r status count; do
        local bar
        bar=$(printf '#%.0s' $(seq 1 "${count}"))
        printf "    %-15s %3d %s\n" "${status}" "${count}" "${bar}"
    done < <(execute_query "SELECT status, COUNT(*) as cnt FROM servers GROUP BY status ORDER BY cnt DESC;")
    echo ""
    
    # Por sistema operativo
    echo "  Distribución por sistema operativo:"
    while IFS=$'\t' read -r os count; do
        printf "    %-30s %3d\n" "${os}" "${count}"
    done < <(execute_query "SELECT os, COUNT(*) as cnt FROM servers GROUP BY os ORDER BY cnt DESC;")
    echo ""
    
    # Último servidor agregado
    echo "  Último servidor registrado:"
    execute_query_with_headers "SELECT hostname, ip, created_at FROM servers ORDER BY created_at DESC LIMIT 1;"
    echo ""
}

#-------------------------------------------------------------------------------
# Función: report_custom_query
#-------------------------------------------------------------------------------
report_custom_query() {
    local query="${1}"
    
    echo "=== CONSULTA PERSONALIZADA ==="
    echo "  SQL: ${query}"
    echo ""
    
    execute_query_with_headers "${query}"
    echo ""
}

#-------------------------------------------------------------------------------
# PROGRAMA PRINCIPAL
#-------------------------------------------------------------------------------

while getopts ":f:s:q:o:h" opt; do
    case ${opt} in
        f) OUTPUT_FORMAT="${OPTARG}" ;;
        s) FILTER_STATUS="${OPTARG}" ;;
        q) CUSTOM_QUERY="${OPTARG}" ;;
        o) OUTPUT_FILE="${OPTARG}" ;;
        h) show_usage; exit 0 ;;
        :) log_message "ERROR" "La opción -${OPTARG} requiere un argumento"; exit 1 ;;
        \?) log_message "ERROR" "Opción inválida: -${OPTARG}"; exit 1 ;;
    esac
done

# Validar formato
case "${OUTPUT_FORMAT}" in
    text|json|csv) ;;
    *) log_message "ERROR" "Formato inválido: ${OUTPUT_FORMAT}. Use: text, json, csv"; exit 1 ;;
esac

log_message "INFO" "Iniciando reporte de base de datos (formato: ${OUTPUT_FORMAT})..."

# Verificar dependencias
check_dependencies "mysql"

# Probar conexión
test_db_connection

# Generar reporte
{
    generate_report_header "REPORTE DE BASE DE DATOS - srv_inventory"
    echo "Servidor MySQL: ${DB_HOST}:${DB_PORT}"
    echo "Base de datos: ${DB_NAME}"
    echo "Formato: ${OUTPUT_FORMAT}"
    [[ -n "${FILTER_STATUS}" ]] && echo "Filtro de estado: ${FILTER_STATUS}"
    echo ""
    
    if [[ -n "${CUSTOM_QUERY}" ]]; then
        report_custom_query "${CUSTOM_QUERY}"
    else
        report_server_inventory
        report_statistics
    fi
    
    echo "======================================================================="
    echo "  Fin del reporte de base de datos"
    echo "======================================================================="
} > "${OUTPUT_FILE}"

# Mostrar el reporte
cat "${OUTPUT_FILE}"

log_message "INFO" "Reporte guardado en: ${OUTPUT_FILE}"
SCRIPT

chmod +x /opt/scripts/db_report.sh
```

2. Ejecutar el reporte en formato texto:

```bash
/opt/scripts/db_report.sh -f text
```

3. Generar reporte filtrado en formato JSON:

```bash
/opt/scripts/db_report.sh -f json -s active -o /opt/sysreport/active_servers.json
```

4. Ejecutar una consulta personalizada:

```bash
/opt/scripts/db_report.sh -q "SELECT hostname, ip, status FROM servers WHERE status != 'active';"
```

### Salida Esperada

```
======================================================================
  REPORTE DE BASE DE DATOS - srv_inventory
  Servidor: srv-linux-01
  Fecha: 2024-01-15 15:00:12
  Usuario: sysadmin
======================================================================

Servidor MySQL: 192.168.100.20:3306
Base de datos: srv_inventory
Formato: text

=== INVENTARIO DE SERVIDORES ===

+----+--------------+----------------+----------------------+-------------+---------------------+
| id | hostname     | ip             | os                   | status      | created_at          |
+----+--------------+----------------+----------------------+-------------+---------------------+
|  1 | srv-linux-01 | 192.168.100.10 | Ubuntu 22.04.4 LTS   | active      | 2024-01-15 10:00:00 |
|  2 | srv-linux-02 | 192.168.100.20 | Ubuntu 22.04.4 LTS   | active      | 2024-01-15 10:00:00 |
|  3 | srv-linux-03 | 192.168.100.30 | Rocky Linux 9.3      | active      | 2024-01-15 10:00:00 |
|  4 | srv-web-01   | 192.168.100.40 | Ubuntu 22.04.4 LTS   | maintenance | 2024-01-15 10:00:00 |
|  5 | srv-db-backup| 192.168.100.50 | Rocky Linux 9.3      | inactive    | 2024-01-15 10:00:00 |
+----+--------------+----------------+----------------------+-------------+---------------------+

=== ESTADÍSTICAS DEL INVENTARIO ===

  Total de servidores registrados: 5

  Distribución por estado:
    active            3 ###
    maintenance       1 #
    inactive          1 #
...
```

### Verificación

```bash
# Verificar archivo JSON generado
cat /opt/sysreport/active_servers.json | jq '.' 2>/dev/null || echo "Verificar formato JSON"

# Verificar logs
grep "db_report" /var/log/admin_scripts.log | tail -5
```

---

## Paso 7: Script Maestro — run_all_reports.sh

### Objetivo
Crear un script orquestador que ejecute todos los scripts anteriores de forma secuencial, maneje errores individuales y genere un resumen consolidado.

### Instrucciones

1. Crear el script maestro:

```bash
cat > /opt/scripts/run_all_reports.sh << 'SCRIPT'
#!/bin/bash
#===============================================================================
# Script: run_all_reports.sh
# Descripción: Ejecuta toda la suite de scripts de automatización
# Uso: ./run_all_reports.sh [-a] [-s script1,script2] [-h]
# Versión: 1.0
#===============================================================================

set -uo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/common_lib.sh"

trap cleanup_on_exit EXIT

# Lista de scripts disponibles
declare -A SCRIPTS=(
    ["inventory"]="${SCRIPT_DIR}/system_inventory.sh"
    ["monitor"]="${SCRIPT_DIR}/service_monitor.sh"
    ["storage"]="${SCRIPT_DIR}/storage_report.sh"
    ["logs"]="${SCRIPT_DIR}/log_analyzer.sh"
    ["database"]="${SCRIPT_DIR}/db_report.sh"
)

RUN_ALL=0
SELECTED_SCRIPTS=""
RESULTS=()

#-------------------------------------------------------------------------------
# Función: show_usage
#-------------------------------------------------------------------------------
show_usage() {
    cat << USAGE
Uso: $(basename "$0") [OPCIONES]

Opciones:
  -a           Ejecutar todos los scripts
  -s LISTA     Lista de scripts separados por coma: inventory,monitor,storage,logs,database
  -h           Muestra esta ayuda

Scripts disponibles:
  inventory  - Inventario de hardware/software
  monitor    - Monitoreo de servicios y recursos
  storage    - Reporte de almacenamiento
  logs       - Análisis de logs del sistema
  database   - Reporte de base de datos MySQL
USAGE
}

#-------------------------------------------------------------------------------
# Función: run_script
# Parámetros: $1 = nombre del script, $2 = ruta del script
#-------------------------------------------------------------------------------
run_script() {
    local name="${1}"
    local script_path="${2}"
    local start_time end_time duration exit_code
    
    echo ""
    echo "─────────────────────────────────────────────────────────────────"
    echo "  Ejecutando: ${name}"
    echo "─────────────────────────────────────────────────────────────────"
    
    if [[ ! -x "${script_path}" ]]; then
        log_message "ERROR" "Script no encontrado o sin permisos: ${script_path}"
        RESULTS+=("${name}: FALLO (script no encontrado)")
        return 1
    fi
    
    start_time=$(date +%s)
    
    # Ejecutar script capturando código de salida
    if "${script_path}" -o "${REPORT_DIR}/${name}_$(date '+%Y%m%d_%H%M%S').txt" 2>&1; then
        exit_code=0
    else
        exit_code=$?
    fi
    
    end_time=$(date +%s)
    duration=$((end_time - start_time))
    
    if [[ ${exit_code} -eq 0 ]]; then
        RESULTS+=("${name}: OK (${duration}s)")
        log_message "INFO" "Script ${name} completado exitosamente en ${duration}s"
    else
        RESULTS+=("${name}: FALLO (código ${exit_code}, ${duration}s)")
        log_message "WARN" "Script ${name} falló con código ${exit_code}"
    fi
    
    return ${exit_code}
}

#-------------------------------------------------------------------------------
# PROGRAMA PRINCIPAL
#-------------------------------------------------------------------------------

while getopts ":as:h" opt; do
    case ${opt} in
        a) RUN_ALL=1 ;;
        s) SELECTED_SCRIPTS="${OPTARG}" ;;
        h) show_usage; exit 0 ;;
        :) log_message "ERROR" "La opción -${OPTARG} requiere un argumento"; exit 1 ;;
        \?) log_message "ERROR" "Opción inválida: -${OPTARG}"; exit 1 ;;
    esac
done

if [[ ${RUN_ALL} -eq 0 ]] && [[ -z "${SELECTED_SCRIPTS}" ]]; then
    show_usage
    exit 1
fi

log_message "INFO" "=== INICIO DE EJECUCIÓN DE SUITE DE SCRIPTS ==="
SUITE_START=$(date +%s)

generate_report_header "EJECUCIÓN DE SUITE DE AUTOMATIZACIÓN"

# Determinar qué scripts ejecutar
if [[ ${RUN_ALL} -eq 1 ]]; then
    scripts_to_run="inventory,monitor,storage,logs,database"
else
    scripts_to_run="${SELECTED_SCRIPTS}"
fi

# Ejecutar scripts seleccionados
IFS=',' read -ra selected <<< "${scripts_to_run}"
total_scripts=${#selected[@]}
failed=0

for script_name in "${selected[@]}"; do
    script_name=$(echo "${script_name}" | xargs)
    if [[ -n "${SCRIPTS[${script_name}]+_}" ]]; then
        if ! run_script "${script_name}" "${SCRIPTS[${script_name}]}"; then
            failed=$((failed + 1))
        fi
    else
        log_message "WARN" "Script desconocido: ${script_name}"
        RESULTS+=("${script_name}: IGNORADO (no reconocido)")
    fi
done

# Resumen final
SUITE_END=$(date +%s)
SUITE_DURATION=$((SUITE_END - SUITE_START))

echo ""
echo "═══════════════════════════════════════════════════════════════════════"
echo "  RESUMEN DE EJECUCIÓN"
echo "═══════════════════════════════════════════════════════════════════════"
echo ""
echo "  Tiempo total: ${SUITE_DURATION} segundos"
echo "  Scripts ejecutados: ${total_scripts}"
echo "  Exitosos: $((total_scripts - failed))"
echo "  Fallidos: ${failed}"
echo ""
echo "  Resultados individuales:"
for result in "${RESULTS[@]}"; do
    echo "    • ${result}"
done
echo ""
echo "  Reportes disponibles en: ${REPORT_DIR}/"
echo "═══════════════════════════════════════════════════════════════════════"

log_message "INFO" "Suite completada: ${total_scripts} scripts, ${failed} fallos, ${SUITE_DURATION}s"

[[ ${failed} -gt 0 ]] && exit 1 || exit 0
SCRIPT

chmod +x /opt/scripts/run_all_reports.sh
```

2. Ejecutar la suite completa:

```bash
sudo /opt/scripts/run_all_reports.sh -a
```

3. Ejecutar solo scripts seleccionados:

```bash
/opt/scripts/run_all_reports.sh -s inventory,monitor
```

### Salida Esperada

```
═══════════════════════════════════════════════════════════════════════
  RESUMEN DE EJECUCIÓN
═══════════════════════════════════════════════════════════════════════

  Tiempo total: 15 segundos
  Scripts ejecutados: 5
  Exitosos: 5
  Fallidos: 0

  Resultados individuales:
    • inventory: OK (3s)
    • monitor: OK (2s)
    • storage: OK (4s)
    • logs: OK (3s)
    • database: OK (3s)

  Reportes disponibles en: /opt/sysreport/
═══════════════════════════════════════════════════════════════════════
```

### Verificación

```bash
# Listar todos los reportes generados
ls -la /opt/sysreport/

# Verificar el log completo de la ejecución
grep "$(date '+%Y-%m-%d')" /var/log/admin_scripts.log | tail -20
```

---

## Paso 8: Depuración y Pruebas de Robustez

### Objetivo
Aplicar técnicas de depuración para verificar la robustez de los scripts y demostrar el uso de `bash -x`, `set -e` y `trap`.

### Instrucciones

1. Depurar un script con `bash -x`:

```bash
# Ejecutar con traza de depuración (solo primeras líneas para demostración)
bash -x /opt/scripts/system_inventory.sh -o /tmp/debug_test.txt 2>&1 | head -40
```

2. Probar manejo de errores con parámetros inválidos:

```bash
# Probar con opción inválida
/opt/scripts/db_report.sh -z 2>&1
echo "Código de salida: $?"

# Probar con formato inválido
/opt/scripts/db_report.sh -f xml 2>&1
echo "Código de salida: $?"

# Probar con archivo de log inexistente
/opt/scripts/log_analyzer.sh -l /var/log/noexiste.log 2>&1
echo "Código de salida: $?"
```

3. Verificar que `set -u` detecta variables no definidas:

```bash
# Crear script de prueba con variable no definida
cat > /tmp/test_set_u.sh << 'EOF'
#!/bin/bash
set -u
echo "Intentando usar variable no definida: ${VARIABLE_INEXISTENTE}"
EOF
chmod +x /tmp/test_set_u.sh
/tmp/test_set_u.sh 2>&1
echo "Código de salida: $?"
```

4. Verificar que `trap` funciona correctamente:

```bash
# Crear script que demuestre trap
cat > /tmp/test_trap.sh << 'EOF'
#!/bin/bash
source /opt/scripts/common_lib.sh
trap cleanup_on_exit EXIT

log_message "INFO" "Script de prueba iniciado"
echo "Simulando trabajo..."
sleep 1
echo "Forzando error..."
false  # Esto causa un exit code != 0
EOF
chmod +x /tmp/test_trap.sh
/tmp/test_trap.sh 2>&1
grep "test_trap" /var/log/admin_scripts.log | tail -3
```

### Salida Esperada

```
# Para bash -x:
+ SCRIPT_DIR=/opt/scripts
+ source /opt/scripts/common_lib.sh
++ readonly LOG_FILE=/var/log/admin_scripts.log
++ LOG_FILE=/var/log/admin_scripts.log
...

# Para parámetros inválidos:
[ERROR] Opción inválida: -z
Código de salida: 1

[ERROR] Formato inválido: xml. Use: text, json, csv
Código de salida: 1

[ERROR] Archivo de log no encontrado: /var/log/noexiste.log
Código de salida: 1

# Para set -u:
/tmp/test_set_u.sh: line 3: VARIABLE_INEXISTENTE: unbound variable
Código de salida: 1

# Para trap:
[WARN] Script finalizado con código de error: 1
```

### Verificación

```bash
# Confirmar que todos los errores fueron registrados en el log
grep -c "ERROR\|WARN" /var/log/admin_scripts.log
```

---

## Paso 9: Configurar Variables de Entorno Permanentes para los Scripts

### Objetivo
Aplicar los conceptos de la Lección 7.1 para configurar variables de entorno permanentes que los scripts utilizarán, aprovechando los archivos de inicialización del shell.

### Instrucciones

1. Agregar variables de entorno al perfil de `sysadmin`:

```bash
# Agregar configuración al .bashrc de sysadmin
