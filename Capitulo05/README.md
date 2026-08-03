# Resolver requerimientos administrativos y operativos utilizando herramientas GNU/Linux, combinando comandos, filtros, expresiones regulares y procesamiento de texto para obtener información crítica del sistema

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 111 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Servidor principal** | srv-linux-01 (Ubuntu 22.04.4 LTS, 192.168.100.10) |

---

## Descripción General

Este laboratorio simula ocho tickets de soporte reales que requieren el uso combinado de herramientas GNU/Linux para navegación avanzada de archivos, procesamiento de texto, expresiones regulares y compresión de datos. Trabajarás en `srv-linux-01` extrayendo información crítica de logs, archivos de configuración y reportes del sistema, redirigiendo toda la salida a `/opt/sysreport/text_processing_exercises/`. Los scripts y técnicas desarrollados aquí serán reutilizados en el laboratorio de scripting Bash (Lab 07).

---

## Objetivos de Aprendizaje

- [ ] Navegar y administrar eficientemente la jerarquía de archivos Linux usando comandos avanzados de búsqueda y manipulación con `find`, `ls` y rutas absolutas/relativas.
- [ ] Procesar y filtrar información del sistema usando tuberías, redirecciones y herramientas GNU (`grep`, `sed`, `awk`, `sort`, `uniq`).
- [ ] Construir expresiones regulares complejas para extracción y validación de datos en archivos de configuración y logs.
- [ ] Comprimir, archivar y gestionar copias de respaldo usando `tar`, `gzip`, `bzip2` y `xz`.

---

## Prerrequisitos

### Conocimiento Previo

- Manejo básico de la línea de comandos Linux (comandos `cd`, `ls`, `pwd`, `cat`)
- Comprensión de la jerarquía de directorios estándar de Linux (FHS)
- Familiaridad con redirecciones básicas (`>`, `>>`, `|`)

### Acceso y Recursos

| Requisito | Detalle |
|-----------|---------|
| Lab 01 completado | Reportes disponibles en `/opt/sysreport/` |
| Lab 02 completado | Logs de recuperación en `/var/log/` |
| Lab 04 completado | Herramientas `curl`, `wget` y `jq` instaladas |
| Acceso SSH a srv-linux-01 | Usuario: `sysadmin` / Contraseña: `Linux@Admin2024!` |

---

## Entorno del Laboratorio

### Software Requerido

| Herramienta | Versión | Paquete |
|-------------|---------|---------|
| grep / egrep | 3.7 | grep |
| sed | 4.8 | sed |
| gawk | 5.1.0 | gawk |
| coreutils (sort, uniq, cut, tr, tee, head, tail, wc, comm) | 8.32 | coreutils |
| find / xargs | 4.8.0 | findutils |
| tar | 1.34 | tar |
| gzip | 1.10 | gzip |
| bzip2 | 1.0.8 | bzip2 |
| xz | 5.2.5 | xz-utils |
| diff | 3.8 | diffutils |
| tree | 1.8.0 | tree |

### Preparación del Entorno

Conéctate a `srv-linux-01` e inicia sesión como `sysadmin`:

```bash
ssh sysadmin@192.168.100.10
```

Ejecuta el bloque de preparación para crear la estructura de trabajo y datos de ejemplo:

```bash
# Crear directorio de trabajo principal
sudo mkdir -p /opt/sysreport/text_processing_exercises
sudo chown sysadmin:sysadmin /opt/sysreport/text_processing_exercises

# Verificar herramientas instaladas
for cmd in grep sed awk find xargs tar gzip bzip2 xz diff comm tree; do
    printf "%-10s: %s\n" "$cmd" "$(which $cmd 2>/dev/null || echo 'NO ENCONTRADO')"
done

# Instalar tree si no está disponible
sudo apt install -y tree 2>/dev/null

# Generar datos de ejemplo si los labs anteriores no generaron suficientes logs
# Crear archivo de log simulado de SSH para el ejercicio 2
if [ ! -f /var/log/auth.log ] || [ $(wc -l < /var/log/auth.log) -lt 10 ]; then
    sudo bash -c 'cat > /tmp/auth_sample.log << "EOF"
Jan 15 08:23:01 srv-linux-01 sshd[1234]: Accepted publickey for sysadmin from 192.168.100.20 port 52341 ssh2
Jan 15 08:25:33 srv-linux-01 sshd[1235]: Failed password for invalid user admin from 10.0.0.55 port 43210 ssh2
Jan 15 09:01:12 srv-linux-01 sshd[1240]: Accepted password for sysadmin from 192.168.100.30 port 60122 ssh2
Jan 15 09:15:44 srv-linux-01 sshd[1242]: Failed password for root from 203.0.113.45 port 33221 ssh2
Jan 15 09:15:47 srv-linux-01 sshd[1242]: Failed password for root from 203.0.113.45 port 33221 ssh2
Jan 15 09:15:50 srv-linux-01 sshd[1242]: Failed password for root from 203.0.113.45 port 33221 ssh2
Jan 15 10:00:01 srv-linux-01 sshd[1250]: Accepted publickey for sysadmin from 192.168.100.20 port 52455 ssh2
Jan 15 10:33:22 srv-linux-01 sshd[1255]: Failed password for invalid user test from 198.51.100.77 port 44556 ssh2
Jan 15 11:12:05 srv-linux-01 sshd[1260]: Accepted password for dbuser from 192.168.100.10 port 55667 ssh2
Jan 15 11:45:33 srv-linux-01 sshd[1265]: Failed password for root from 203.0.113.45 port 33445 ssh2
Jan 15 12:00:01 srv-linux-01 sshd[1270]: Accepted publickey for sysadmin from 192.168.100.30 port 60200 ssh2
Jan 15 13:22:15 srv-linux-01 sshd[1280]: Failed password for invalid user admin from 10.0.0.55 port 43500 ssh2
Jan 15 14:05:30 srv-linux-01 sshd[1285]: Connection closed by 172.16.0.100 port 22 [preauth]
Jan 15 15:10:45 srv-linux-01 sshd[1290]: Failed password for nobody from 198.51.100.77 port 44600 ssh2
EOF'
    sudo cp /tmp/auth_sample.log /opt/sysreport/text_processing_exercises/auth_sample.log
fi

# Crear directorio de trabajo para cada ejercicio
for i in $(seq 1 8); do
    mkdir -p /opt/sysreport/text_processing_exercises/ejercicio_${i}
done

echo "=== Entorno preparado correctamente ==="
ls -la /opt/sysreport/text_processing_exercises/
```

**Salida esperada:**

```
=== Entorno preparado correctamente ===
total 40
drwxr-xr-x 10 sysadmin sysadmin 4096 ... .
drwxr-xr-x  4 sysadmin sysadmin 4096 ... ..
-rw-r--r--  1 root     root     1247 ... auth_sample.log
drwxr-xr-x  2 sysadmin sysadmin 4096 ... ejercicio_1
drwxr-xr-x  2 sysadmin sysadmin 4096 ... ejercicio_2
...
drwxr-xr-x  2 sysadmin sysadmin 4096 ... ejercicio_8
```

---

## Procedimiento Paso a Paso

### Ejercicio 1: Búsqueda de archivos de configuración modificados en las últimas 24 horas

**Objetivo:** Utilizar `find` con criterios de tiempo y tipo para localizar archivos de configuración recientemente modificados, aplicando redirección de salida y manejo de errores.

**Instrucciones:**

1. Navega al directorio de trabajo del ejercicio:

```bash
cd /opt/sysreport/text_processing_exercises/ejercicio_1
pwd
```

2. Busca todos los archivos con extensión `.conf` modificados en las últimas 24 horas dentro de `/etc`:

```bash
sudo find /etc -type f -name "*.conf" -mtime -1 2>/dev/null | sort > configs_modificados_24h.txt
```

3. Busca archivos de configuración (`.conf`, `.cfg`, `.ini`) modificados en los últimos 7 días, incluyendo tamaño y fecha:

```bash
sudo find /etc -type f \( -name "*.conf" -o -name "*.cfg" -o -name "*.ini" \) \
    -mtime -7 -exec ls -lh {} \; 2>/dev/null | \
    awk '{print $5, $6, $7, $8, $9}' | sort -k4 > configs_modificados_7dias.txt
```

4. Genera un resumen con el conteo total de archivos encontrados:

```bash
echo "=== REPORTE: Archivos de configuración modificados ===" > reporte_configs.txt
echo "Fecha de generación: $(date '+%Y-%m-%d %H:%M:%S')" >> reporte_configs.txt
echo "Servidor: $(hostname)" >> reporte_configs.txt
echo "---" >> reporte_configs.txt
echo "Archivos .conf modificados en 24h: $(wc -l < configs_modificados_24h.txt)" >> reporte_configs.txt
echo "Archivos .conf/.cfg/.ini modificados en 7 días: $(wc -l < configs_modificados_7dias.txt)" >> reporte_configs.txt
echo "---" >> reporte_configs.txt
echo "Detalle (últimas 24 horas):" >> reporte_configs.txt
cat configs_modificados_24h.txt >> reporte_configs.txt
```

5. Encuentra archivos de configuración de más de 10 KB que no pertenezcan a root:

```bash
sudo find /etc -type f -name "*.conf" -size +10k ! -user root 2>/dev/null > configs_no_root.txt
echo "Archivos .conf >10KB no propiedad de root: $(wc -l < configs_no_root.txt)" >> reporte_configs.txt
```

6. Visualiza el reporte final:

```bash
cat reporte_configs.txt
```

**Salida esperada (ejemplo):**

```
=== REPORTE: Archivos de configuración modificados ===
Fecha de generación: 2024-03-15 10:30:45
Servidor: srv-linux-01
---
Archivos .conf modificados en 24h: 3
Archivos .conf/.cfg/.ini modificados en 7 días: 12
---
Detalle (últimas 24 horas):
/etc/resolv.conf
/etc/systemd/resolved.conf
/etc/sysctl.conf
```

**Verificación:**

```bash
# Verificar que los archivos de salida existen y tienen contenido
test -s reporte_configs.txt && echo "✓ Reporte generado correctamente" || echo "✗ Error: reporte vacío"
test -f configs_modificados_24h.txt && echo "✓ Archivo de 24h existe" || echo "✗ Error"
```

---

### Ejercicio 2: Extracción de IPs únicas de logs de SSH

**Objetivo:** Utilizar `grep` con expresiones regulares, `awk` para extracción de campos, y `sort`/`uniq` para obtener estadísticas de conexiones SSH.

**Instrucciones:**

1. Navega al directorio del ejercicio:

```bash
cd /opt/sysreport/text_processing_exercises/ejercicio_2
```

2. Define la fuente de logs (usa el archivo real o el de muestra):

```bash
# Usar auth.log real si existe y tiene contenido SSH, si no usar la muestra
if sudo grep -q "sshd" /var/log/auth.log 2>/dev/null; then
    LOG_SOURCE="/var/log/auth.log"
else
    LOG_SOURCE="/opt/sysreport/text_processing_exercises/auth_sample.log"
fi
echo "Usando fuente de logs: $LOG_SOURCE"
```

3. Extrae todas las direcciones IP que aparecen en conexiones SSH (exitosas y fallidas):

```bash
sudo grep "sshd" "$LOG_SOURCE" | \
    grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' | \
    sort | uniq -c | sort -rn > ips_todas_ssh.txt
cat ips_todas_ssh.txt
```

4. Extrae solo las IPs de intentos fallidos:

```bash
sudo grep "sshd" "$LOG_SOURCE" | grep -i "failed" | \
    grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' | \
    sort | uniq -c | sort -rn > ips_fallidas.txt
cat ips_fallidas.txt
```

5. Extrae las IPs de conexiones exitosas:

```bash
sudo grep "sshd" "$LOG_SOURCE" | grep -i "accepted" | \
    grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' | \
    sort | uniq -c | sort -rn > ips_exitosas.txt
cat ips_exitosas.txt
```

6. Identifica IPs con más de 2 intentos fallidos (posibles ataques de fuerza bruta):

```bash
awk '$1 > 2 {print $0}' ips_fallidas.txt > ips_sospechosas.txt
cat ips_sospechosas.txt
```

7. Genera el reporte consolidado con formato profesional:

```bash
cat > reporte_ssh.txt << 'HEADER'
╔══════════════════════════════════════════════════════════════╗
║          REPORTE DE ACTIVIDAD SSH - srv-linux-01            ║
╚══════════════════════════════════════════════════════════════╝
HEADER

echo "Generado: $(date '+%Y-%m-%d %H:%M:%S')" >> reporte_ssh.txt
echo "Fuente: $LOG_SOURCE" >> reporte_ssh.txt
echo "" >> reporte_ssh.txt
echo "--- IPs con conexiones EXITOSAS ---" >> reporte_ssh.txt
cat ips_exitosas.txt >> reporte_ssh.txt
echo "" >> reporte_ssh.txt
echo "--- IPs con intentos FALLIDOS ---" >> reporte_ssh.txt
cat ips_fallidas.txt >> reporte_ssh.txt
echo "" >> reporte_ssh.txt
echo "--- IPs SOSPECHOSAS (>2 fallos) ---" >> reporte_ssh.txt
if [ -s ips_sospechosas.txt ]; then
    cat ips_sospechosas.txt >> reporte_ssh.txt
else
    echo "(ninguna)" >> reporte_ssh.txt
fi
echo "" >> reporte_ssh.txt
echo "Total IPs únicas: $(cat ips_todas_ssh.txt | wc -l)" >> reporte_ssh.txt

cat reporte_ssh.txt
```

**Salida esperada:**

```
╔══════════════════════════════════════════════════════════════╗
║          REPORTE DE ACTIVIDAD SSH - srv-linux-01            ║
╚══════════════════════════════════════════════════════════════╝
Generado: 2024-03-15 10:45:22
Fuente: /opt/sysreport/text_processing_exercises/auth_sample.log

--- IPs con conexiones EXITOSAS ---
      3 192.168.100.20
      2 192.168.100.30
      1 192.168.100.10

--- IPs con intentos FALLIDOS ---
      4 203.0.113.45
      2 10.0.0.55
      2 198.51.100.77

--- IPs SOSPECHOSAS (>2 fallos) ---
      4 203.0.113.45

Total IPs únicas: 6
```

**Verificación:**

```bash
# Verificar que la expresión regular captura IPs válidas
grep -cE '^[[:space:]]+[0-9]+ [0-9]+\.' ips_todas_ssh.txt > /dev/null && \
    echo "✓ Formato de IPs correcto" || echo "✗ Error en formato"
test -s reporte_ssh.txt && echo "✓ Reporte SSH generado" || echo "✗ Error"
```

---

### Ejercicio 3: Análisis de uso de disco con reporte formateado

**Objetivo:** Combinar `du`, `sort`, `head`, `awk` y redirecciones para generar un reporte profesional de uso de disco.

**Instrucciones:**

1. Navega al directorio del ejercicio:

```bash
cd /opt/sysreport/text_processing_exercises/ejercicio_3
```

2. Obtén los 15 directorios que más espacio consumen en `/var`:

```bash
sudo du -sh /var/*/ 2>/dev/null | sort -rh | head -15 > top15_var.txt
cat top15_var.txt
```

3. Genera un reporte formateado con `awk` añadiendo porcentajes:

```bash
# Obtener el total de /var en KB para calcular porcentajes
TOTAL_KB=$(sudo du -s /var 2>/dev/null | awk '{print $1}')

sudo du -s /var/*/ 2>/dev/null | sort -rn | head -15 | \
    awk -v total="$TOTAL_KB" 'BEGIN {
        printf "%-40s %10s %8s\n", "DIRECTORIO", "TAMAÑO", "% USO"
        printf "%-40s %10s %8s\n", "----------------------------------------", "----------", "--------"
    }
    {
        pct = ($1 / total) * 100
        # Convertir KB a formato legible
        size = $1
        unit = "KB"
        if (size > 1024) { size = size / 1024; unit = "MB" }
        if (size > 1024) { size = size / 1024; unit = "GB" }
        printf "%-40s %7.1f %s %7.1f%%\n", $2, size, unit, pct
    }' > uso_disco_formateado.txt

cat uso_disco_formateado.txt
```

4. Encuentra los 10 archivos individuales más grandes del sistema:

```bash
sudo find / -type f -exec du -k {} + 2>/dev/null | \
    sort -rn | head -10 | \
    awk '{
        size = $1
        unit = "KB"
        if (size > 1024) { size = size / 1024; unit = "MB" }
        if (size > 1024) { size = size / 1024; unit = "GB" }
        printf "%7.1f %s  %s\n", size, unit, $2
    }' > top10_archivos_grandes.txt

cat top10_archivos_grandes.txt
```

5. Genera el reporte final combinado:

```bash
{
    echo "=== REPORTE DE USO DE DISCO - $(hostname) ==="
    echo "Fecha: $(date '+%Y-%m-%d %H:%M:%S')"
    echo ""
    echo "--- Espacio total del sistema ---"
    df -h / | tail -1 | awk '{printf "Usado: %s de %s (%s)\n", $3, $2, $5}'
    echo ""
    echo "--- Top 15 directorios en /var ---"
    cat uso_disco_formateado.txt
    echo ""
    echo "--- Top 10 archivos más grandes del sistema ---"
    cat top10_archivos_grandes.txt
} > reporte_disco.txt

cat reporte_disco.txt
```

**Salida esperada (ejemplo):**

```
=== REPORTE DE USO DE DISCO - srv-linux-01 ===
Fecha: 2024-03-15 11:00:15

--- Espacio total del sistema ---
Usado: 8.2G de 39G (22%)

--- Top 15 directorios en /var ---
DIRECTORIO                               TAMAÑO      % USO
---------------------------------------- ---------- --------
/var/lib/                                   3.2 GB   62.5%
/var/log/                                 512.0 MB   10.0%
/var/cache/                               256.0 MB    5.0%
...
```

**Verificación:**

```bash
test -s reporte_disco.txt && echo "✓ Reporte de disco generado" || echo "✗ Error"
wc -l reporte_disco.txt
```

---

### Ejercicio 4: Filtrado y transformación de /etc/passwd con awk

**Objetivo:** Utilizar `awk` como lenguaje de procesamiento de campos para extraer, transformar y reportar información de usuarios del sistema.

**Instrucciones:**

1. Navega al directorio del ejercicio:

```bash
cd /opt/sysreport/text_processing_exercises/ejercicio_4
```

2. Lista todos los usuarios con UID >= 1000 (usuarios regulares) mostrando usuario, UID y shell:

```bash
awk -F: '$3 >= 1000 {printf "%-20s UID: %-6d Shell: %s\n", $1, $3, $7}' /etc/passwd > usuarios_regulares.txt
cat usuarios_regulares.txt
```

3. Identifica usuarios con shell `/bin/bash` o `/bin/sh` (usuarios interactivos):

```bash
awk -F: '$7 ~ /\/(bash|sh)$/ {print $1, $3, $6, $7}' /etc/passwd | \
    sort -t' ' -k2 -n > usuarios_interactivos.txt
cat usuarios_interactivos.txt
```

4. Encuentra usuarios del sistema (UID < 1000) que tengan un shell válido (potencial riesgo):

```bash
awk -F: '$3 < 1000 && $7 !~ /(nologin|false|sync|halt|shutdown)$/ {
    printf "⚠️  %-15s UID: %-5d Shell: %s\n", $1, $3, $7
}' /etc/passwd > usuarios_sistema_con_shell.txt
cat usuarios_sistema_con_shell.txt
```

5. Genera estadísticas de shells utilizados:

```bash
awk -F: '{print $7}' /etc/passwd | sort | uniq -c | sort -rn > estadisticas_shells.txt
cat estadisticas_shells.txt
```

6. Crea un reporte en formato CSV para posible importación a herramientas de auditoría:

```bash
echo "username,uid,gid,home,shell,type" > usuarios_auditoria.csv
awk -F: '{
    type = ($3 >= 1000) ? "regular" : "system"
    printf "%s,%d,%d,%s,%s,%s\n", $1, $3, $4, $6, $7, type
}' /etc/passwd >> usuarios_auditoria.csv

echo "--- Primeras 10 líneas del CSV ---"
head -10 usuarios_auditoria.csv

echo ""
echo "--- Resumen ---"
echo "Total usuarios: $(tail -n +2 usuarios_auditoria.csv | wc -l)"
echo "Usuarios regulares: $(grep ',regular$' usuarios_auditoria.csv | wc -l)"
echo "Usuarios de sistema: $(grep ',system$' usuarios_auditoria.csv | wc -l)"
```

7. Combina con información de grupos usando un join lógico:

```bash
awk -F: '$3 >= 1000 {
    # Buscar el nombre del grupo primario
    cmd = "getent group " $4 " | cut -d: -f1"
    cmd | getline grupo
    close(cmd)
    printf "%-15s Grupo: %-15s Home: %s\n", $1, grupo, $6
}' /etc/passwd > usuarios_con_grupo.txt
cat usuarios_con_grupo.txt
```

**Salida esperada (ejemplo):**

```
sysadmin             UID: 1000   Shell: /bin/bash
--- Resumen ---
Total usuarios: 35
Usuarios regulares: 2
Usuarios de sistema: 33
```

**Verificación:**

```bash
# Verificar que el CSV tiene el formato correcto
head -1 usuarios_auditoria.csv | grep -q "username,uid,gid,home,shell,type" && \
    echo "✓ CSV con headers correctos" || echo "✗ Error en CSV"
test -s usuarios_regulares.txt && echo "✓ Usuarios regulares extraídos" || echo "✗ Error"
```

---

### Ejercicio 5: Búsqueda de patrones en logs con expresiones regulares extendidas

**Objetivo:** Construir expresiones regulares POSIX extendidas (ERE) complejas para extraer y validar datos específicos de logs y archivos de configuración.

**Instrucciones:**

1. Navega al directorio del ejercicio:

```bash
cd /opt/sysreport/text_processing_exercises/ejercicio_5
```

2. Crea un archivo de datos de prueba con diversos formatos para practicar regex:

```bash
cat > datos_prueba.txt << 'EOF'
# Direcciones IP válidas e inválidas
192.168.1.1
10.0.0.255
256.1.2.3
172.16.0.100
999.999.999.999
192.168.100.10

# Direcciones email
admin@empresa.com
usuario.nombre@dominio.co.cr
invalido@
test@server.local
root@192.168.1.1

# Fechas en diversos formatos
2024-03-15
15/03/2024
03-15-2024
2024/13/45
2024-12-31

# Direcciones MAC
AA:BB:CC:DD:EE:FF
00:1A:2B:3C:4D:5E
GG:HH:II:JJ:KK:LL
a1:b2:c3:d4:e5:f6

# Números de teléfono
+506 8888-1234
(506) 2222-3333
88881234
+1-800-555-0199

# URLs
https://www.ejemplo.com/path?query=value
http://192.168.1.1:8080/admin
ftp://files.server.local/pub
not_a_url
EOF
```

3. Extrae direcciones IPv4 válidas (0-255 en cada octeto):

```bash
grep -E '^([0-9]{1,2}|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.([0-9]{1,2}|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.([0-9]{1,2}|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.([0-9]{1,2}|1[0-9]{2}|2[0-4][0-9]|25[0-5])$' datos_prueba.txt > ips_validas.txt
echo "IPs válidas encontradas:"
cat ips_validas.txt
```

4. Extrae direcciones de correo electrónico válidas:

```bash
grep -E '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$' datos_prueba.txt > emails_validos.txt
echo "Emails válidos:"
cat emails_validos.txt
```

5. Extrae fechas en formato ISO (YYYY-MM-DD) con validación básica:

```bash
grep -E '^[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[01])$' datos_prueba.txt > fechas_iso.txt
echo "Fechas ISO válidas:"
cat fechas_iso.txt
```

6. Extrae direcciones MAC válidas (case-insensitive):

```bash
grep -iE '^([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}$' datos_prueba.txt > macs_validas.txt
echo "MACs válidas:"
cat macs_validas.txt
```

7. Busca en logs del sistema líneas que contengan errores o advertencias:

```bash
sudo grep -iE '(error|fail|warn|crit|alert|emerg)' /var/log/syslog 2>/dev/null | \
    tail -20 > errores_syslog.txt

# Contar por severidad
echo "--- Conteo por tipo de mensaje ---" > resumen_errores.txt
for pattern in error fail warn crit alert emerg; do
    count=$(sudo grep -ic "$pattern" /var/log/syslog 2>/dev/null || echo "0")
    printf "%-10s: %s\n" "$pattern" "$count" >> resumen_errores.txt
done
cat resumen_errores.txt
```

8. Extrae URLs de archivos usando regex extendida:

```bash
grep -E '^(https?|ftp)://[a-zA-Z0-9./_:?=&%-]+$' datos_prueba.txt > urls_validas.txt
echo "URLs válidas:"
cat urls_validas.txt
```

9. Busca en `/etc` archivos que contengan direcciones IP hardcodeadas:

```bash
sudo grep -rlE '([0-9]{1,3}\.){3}[0-9]{1,3}' /etc/ 2>/dev/null | \
    head -20 > archivos_con_ips.txt
echo "Archivos en /etc con IPs hardcodeadas: $(wc -l < archivos_con_ips.txt)"
cat archivos_con_ips.txt
```

**Salida esperada:**

```
IPs válidas encontradas:
192.168.1.1
10.0.0.255
172.16.0.100
192.168.100.10

Emails válidos:
admin@empresa.com
usuario.nombre@dominio.co.cr
test@server.local

Fechas ISO válidas:
2024-03-15
2024-12-31

MACs válidas:
AA:BB:CC:DD:EE:FF
00:1A:2B:3C:4D:5E
a1:b2:c3:d4:e5:f6
```

**Verificación:**

```bash
# Verificar que 256.1.2.3 y 999.999.999.999 NO aparecen en IPs válidas
! grep -q "256\|999" ips_validas.txt && echo "✓ Regex de IP filtra correctamente" || echo "✗ Error: IPs inválidas incluidas"
# Verificar que invalido@ NO aparece en emails
! grep -q "invalido@$" emails_validos.txt && echo "✓ Regex de email filtra correctamente" || echo "✗ Error"
```

---

### Ejercicio 6: Generación de reporte de servicios activos con sed y sort

**Objetivo:** Utilizar `sed` para transformar texto, `systemctl` para obtener estado de servicios, y combinar con `sort` y `awk` para generar reportes estructurados.

**Instrucciones:**

1. Navega al directorio del ejercicio:

```bash
cd /opt/sysreport/text_processing_exercises/ejercicio_6
```

2. Obtén la lista de todos los servicios activos y formatea con `sed`:

```bash
systemctl list-units --type=service --state=running --no-pager --no-legend | \
    sed 's/●//g' | \
    awk '{print $1}' | \
    sed 's/\.service$//' | \
    sort > servicios_activos.txt

echo "Servicios activos: $(wc -l < servicios_activos.txt)"
head -20 servicios_activos.txt
```

3. Obtén servicios fallidos y formatea la salida:

```bash
systemctl list-units --type=service --state=failed --no-pager --no-legend | \
    sed 's/●//g' | \
    awk '{print $1}' | \
    sed 's/\.service$//' | \
    sort > servicios_fallidos.txt

echo "Servicios fallidos: $(wc -l < servicios_fallidos.txt)"
cat servicios_fallidos.txt
```

4. Genera un reporte de servicios habilitados vs deshabilitados:

```bash
systemctl list-unit-files --type=service --no-pager --no-legend | \
    awk '{print $2, $1}' | \
    sort | \
    awk '{
        status[$1]++
        services[$1] = services[$1] "\n    " $2
    }
    END {
        for (s in status) {
            printf "Estado: %-12s Cantidad: %d\n", s, status[s]
        }
    }' > resumen_servicios_estado.txt

cat resumen_servicios_estado.txt
```

5. Usa `sed` para transformar la salida de `systemctl status` en formato legible:

```bash
# Obtener información detallada de servicios clave
for svc in ssh cron systemd-resolved; do
    systemctl status "$svc" 2>/dev/null | \
        sed -n '1,5p' | \
        sed 's/^[[:space:]]*//' | \
        sed "s/^/[$svc] /"
    echo ""
done > detalle_servicios_clave.txt

cat detalle_servicios_clave.txt
```

6. Crea un reporte final con formato de tabla usando `sed` y `awk`:

```bash
{
    echo "╔════════════════════════════════════════════════════════════╗"
    echo "║       INVENTARIO DE SERVICIOS - $(hostname)              ║"
    echo "╠════════════════════════════════════════════════════════════╣"
    printf "║ %-25s │ %-10s │ %-12s ║\n" "SERVICIO" "ESTADO" "HABILITADO"
    echo "╠════════════════════════════════════════════════════════════╣"
    
    systemctl list-units --type=service --no-pager --no-legend | \
        awk '{print $1, $3, $4}' | \
        sed 's/\.service//' | \
        while read svc load active; do
            enabled=$(systemctl is-enabled "$svc.service" 2>/dev/null || echo "unknown")
            printf "║ %-25s │ %-10s │ %-12s ║\n" "$svc" "$active" "$enabled"
        done | head -20
    
    echo "╚════════════════════════════════════════════════════════════╝"
} > reporte_servicios.txt

cat reporte_servicios.txt
```

**Salida esperada (ejemplo):**

```
╔════════════════════════════════════════════════════════════╗
║       INVENTARIO DE SERVICIOS - srv-linux-01              ║
╠════════════════════════════════════════════════════════════╣
║ SERVICIO                  │ ESTADO     │ HABILITADO   ║
╠════════════════════════════════════════════════════════════╣
║ cron                      │ running    │ enabled      ║
║ dbus                      │ running    │ static       ║
║ ssh                       │ running    │ enabled      ║
...
╚════════════════════════════════════════════════════════════╝
```

**Verificación:**

```bash
test -s servicios_activos.txt && echo "✓ Lista de servicios activos generada" || echo "✗ Error"
grep -q "ssh\|sshd" servicios_activos.txt && echo "✓ SSH detectado como activo" || echo "✗ SSH no detectado"
```

---

### Ejercicio 7: Compresión incremental de logs con tar y gzip

**Objetivo:** Dominar las herramientas de archivado y compresión (`tar`, `gzip`, `bzip2`, `xz`) para crear respaldos eficientes de logs del sistema.

**Instrucciones:**

1. Navega al directorio del ejercicio:

```bash
cd /opt/sysreport/text_processing_exercises/ejercicio_7
```

2. Crea un archivo tar sin compresión de los logs del sistema:

```bash
sudo tar -cvf logs_sistema.tar /var/log/*.log 2>/dev/null
ls -lh logs_sistema.tar
```

3. Comprime el mismo contenido con gzip, bzip2 y xz para comparar:

```bash
# Compresión con gzip
sudo tar -czvf logs_sistema.tar.gz /var/log/*.log 2>/dev/null

# Compresión con bzip2
sudo tar -cjvf logs_sistema.tar.bz2 /var/log/*.log 2>/dev/null

# Compresión con xz
sudo tar -cJvf logs_sistema.tar.xz /var/log/*.log 2>/dev/null
```

4. Compara los tamaños y ratios de compresión:

```bash
{
    echo "=== COMPARATIVA DE COMPRESIÓN ==="
    echo ""
    printf "%-30s %10s %10s\n" "ARCHIVO" "TAMAÑO" "RATIO"
    printf "%-30s %10s %10s\n" "------------------------------" "----------" "----------"
    
    TAR_SIZE=$(stat -c%s logs_sistema.tar)
    
    for file in logs_sistema.tar logs_sistema.tar.gz logs_sistema.tar.bz2 logs_sistema.tar.xz; do
        if [ -f "$file" ]; then
            SIZE=$(stat -c%s "$file")
            HUMAN=$(ls -lh "$file" | awk '{print $5}')
            if [ "$TAR_SIZE" -gt 0 ]; then
                RATIO=$(echo "scale=1; ($SIZE * 100) / $TAR_SIZE" | bc)
            else
                RATIO="N/A"
            fi
            printf "%-30s %10s %9s%%\n" "$file" "$HUMAN" "$RATIO"
        fi
    done
} | tee comparativa_compresion.txt
```

5. Crea un respaldo incremental basado en fecha (solo archivos modificados hoy):

```bash
# Crear respaldo solo de logs modificados hoy
sudo find /var/log -type f -name "*.log" -mtime -1 -print0 | \
    sudo xargs -0 tar -czvf backup_logs_$(date +%Y%m%d).tar.gz 2>/dev/null

ls -lh backup_logs_*.tar.gz
```

6. Lista el contenido de un archivo comprimido sin extraerlo:

```bash
echo "--- Contenido de logs_sistema.tar.gz ---"
tar -tzvf logs_sistema.tar.gz | head -20
```

7. Extrae un archivo específico del tar:

```bash
# Crear directorio de extracción
mkdir -p extraccion_parcial

# Extraer solo syslog del archivo comprimido (si existe)
tar -xzvf logs_sistema.tar.gz -C extraccion_parcial "var/log/syslog" 2>/dev/null || \
    echo "syslog no encontrado en el archivo, extrayendo primer archivo disponible..."

ls -la extraccion_parcial/var/log/ 2>/dev/null
```

8. Crea un script de respaldo reutilizable:

```bash
cat > /opt/sysreport/text_processing_exercises/ejercicio_7/backup_logs.sh << 'SCRIPT'
#!/bin/bash
# Script de respaldo de logs con compresión
# Uso: ./backup_logs.sh [directorio_origen] [directorio_destino]

ORIGEN="${1:-/var/log}"
DESTINO="${2:-/opt/backups}"
FECHA=$(date +%Y%m%d_%H%M%S)
ARCHIVO="backup_logs_${FECHA}.tar.xz"

mkdir -p "$DESTINO"

echo "[$(date)] Iniciando respaldo de $ORIGEN..."
sudo find "$ORIGEN" -type f -name "*.log" -mtime -7 -print0 | \
    sudo xargs -0 tar -cJf "${DESTINO}/${ARCHIVO}" 2>/dev/null

if [ $? -eq 0 ]; then
    SIZE=$(ls -lh "${DESTINO}/${ARCHIVO}" | awk '{print $5}')
    echo "[$(date)] Respaldo completado: ${DESTINO}/${ARCHIVO} ($SIZE)"
else
    echo "[$(date)] ERROR: Fallo en el respaldo"
    exit 1
fi

# Eliminar respaldos con más de 30 días
find "$DESTINO" -name "backup_logs_*.tar.xz" -mtime +30 -delete
echo "[$(date)] Limpieza de respaldos antiguos completada"
SCRIPT

chmod +x /opt/sysreport/text_processing_exercises/ejercicio_7/backup_logs.sh
echo "✓ Script de respaldo creado"
cat backup_logs.sh
```

**Salida esperada (ejemplo):**

```
=== COMPARATIVA DE COMPRESIÓN ===

ARCHIVO                        TAMAÑO      RATIO
------------------------------ ---------- ----------
logs_sistema.tar                  2.1M     100.0%
logs_sistema.tar.gz               185K       8.8%
logs_sistema.tar.bz2              142K       6.7%
logs_sistema.tar.xz               128K       6.1%
```

**Verificación:**

```bash
# Verificar integridad del archivo gzip
gzip -t logs_sistema.tar.gz && echo "✓ Archivo gzip íntegro" || echo "✗ Archivo corrupto"
# Verificar que el script es ejecutable
test -x backup_logs.sh && echo "✓ Script ejecutable" || echo "✗ Script sin permisos"
```

---

### Ejercicio 8: Comparación de archivos de configuración con diff y comm

**Objetivo:** Utilizar `diff`, `comm` y `sdiff` para comparar archivos de configuración, identificar cambios y generar reportes de diferencias.

**Instrucciones:**

1. Navega al directorio del ejercicio:

```bash
cd /opt/sysreport/text_processing_exercises/ejercicio_8
```

2. Crea dos versiones de un archivo de configuración para simular cambios:

```bash
# Versión "original" (antes de cambios)
cat > config_original.conf << 'EOF'
# Configuración del servidor
server_name = srv-linux-01
listen_port = 8080
max_connections = 100
timeout = 30
log_level = info
log_file = /var/log/app.log
database_host = localhost
database_port = 3306
database_name = srv_inventory
database_user = dbuser_scripts
enable_ssl = false
ssl_cert = /etc/ssl/certs/app.crt
ssl_key = /etc/ssl/private/app.key
workers = 4
max_memory = 512M
EOF

# Versión "modificada" (después de cambios)
cat > config_modificada.conf << 'EOF'
# Configuración del servidor - Actualizada 2024-03-15
server_name = srv-linux-01
listen_port = 443
max_connections = 250
timeout = 60
log_level = warning
log_file = /var/log/app.log
database_host = 192.168.100.10
database_port = 3306
database_name = srv_inventory
database_user = dbuser_scripts
database_password = DB@Scripts2024!
enable_ssl = true
ssl_cert = /etc/ssl/certs/app.crt
ssl_key = /etc/ssl/private/app.key
workers = 8
max_memory = 1024M
cache_enabled = true
cache_ttl = 3600
EOF
```

3. Compara los archivos con `diff` en formato unificado:

```bash
diff -u config_original.conf config_modificada.conf > diferencias_unified.txt
cat diferencias_unified.txt
```

4. Genera un reporte con `diff` en formato contextual:

```bash
diff -c config_original.conf config_modificada.conf > diferencias_context.txt
head -40 diferencias_context.txt
```

5. Usa `sdiff` para ver comparación lado a lado:

```bash
sdiff -w 100 config_original.conf config_modificada.conf > comparacion_lado_a_lado.txt
cat comparacion_lado_a_lado.txt
```

6. Usa `comm` para identificar líneas únicas en cada archivo (requiere archivos ordenados):

```bash
# Ordenar ambos archivos (comm requiere entrada ordenada)
sort config_original.conf > config_original_sorted.txt
sort config_modificada.conf > config_modificada_sorted.txt

# Líneas solo en el original (eliminadas)
comm -23 config_original_sorted.txt config_modificada_sorted.txt > lineas_eliminadas.txt

# Líneas solo en el modificado (añadidas)
comm -13 config_original_sorted.txt config_modificada_sorted.txt > lineas_añadidas.txt

# Líneas comunes
comm -12 config_original_sorted.txt config_modificada_sorted.txt > lineas_comunes.txt

echo "Líneas eliminadas: $(wc -l < lineas_eliminadas.txt)"
echo "Líneas añadidas: $(wc -l < lineas_añadidas.txt)"
echo "Líneas comunes: $(wc -l < lineas_comunes.txt)"
```

7. Genera un reporte profesional de cambios:

```bash
{
    echo "╔══════════════════════════════════════════════════════════════╗"
    echo "║         REPORTE DE CAMBIOS EN CONFIGURACIÓN                ║"
    echo "╚══════════════════════════════════════════════════════════════╝"
    echo ""
    echo "Archivo original:  config_original.conf"
    echo "Archivo modificado: config_modificada.conf"
    echo "Fecha de análisis: $(date '+%Y-%m-%d %H:%M:%S')"
    echo ""
    echo "━━━ RESUMEN ━━━"
    echo "  Líneas en original:  $(wc -l < config_original.conf)"
    echo "  Líneas en modificado: $(wc -l < config_modificada.conf)"
    echo "  Parámetros cambiados: $(diff config_original.conf config_modificada.conf | grep -c '^[<>]')"
    echo ""
    echo "━━━ PARÁMETROS MODIFICADOS ━━━"
    diff config_original.conf config_modificada.conf | \
        grep '^[<>]' | \
        sed 's/^< /  [ANTES] /; s/^> /  [AHORA] /' | \
        grep -v '^--$'
    echo ""
    echo "━━━ PARÁMETROS NUEVOS (añadidos) ━━━"
    diff config_original.conf config_modificada.conf | \
        grep '^>' | \
        sed 's/^> /  [+] /'
} > reporte_cambios.txt

cat reporte_cambios.txt
```

8. Compara archivos reales del sistema (ejemplo: resolv.conf actual vs respaldo):

```bash
# Crear respaldo actual de resolv.conf
cp /etc/resolv.conf resolv_actual.txt

# Simular una versión anterior
sed 's/nameserver/# nameserver/' /etc/resolv.conf > resolv_anterior.txt
echo "nameserver 8.8.8.8" >> resolv_anterior.txt

echo "--- Diferencias en resolv.conf ---"
diff --color=always resolv_anterior.txt resolv_actual.txt 2>/dev/null || \
    diff resolv_anterior.txt resolv_actual.txt
```

**Salida esperada (ejemplo del diff unificado):**

```
--- config_original.conf	2024-03-15 11:30:00.000000000 -0600
+++ config_modificada.conf	2024-03-15 11:30:00.000000000 -0600
@@ -1,16 +1,20 @@
-# Configuración del servidor
+# Configuración del servidor - Actualizada 2024-03-15
 server_name = srv-linux-01
-listen_port = 8080
-max_connections = 100
-timeout = 30
-log_level = info
+listen_port = 443
+max_connections = 250
+timeout = 60
+log_level = warning
 log_file = /var/log/app.log
-database_host = localhost
+database_host = 192.168.100.10
...
```

**Verificación:**

```bash
test -s diferencias_unified.txt && echo "✓ Diff unificado generado" || echo "✗ Error"
test -s reporte_cambios.txt && echo "✓ Reporte de cambios generado" || echo "✗ Error"
# Verificar que se detectaron cambios
[ $(wc -l < diferencias_unified.txt) -gt 5 ] && echo "✓ Cambios detectados correctamente" || echo "✗ Error"
```

---

## Validación y Pruebas Finales

Ejecuta el siguiente script de validación integral para confirmar que todos los ejercicios se completaron correctamente:

```bash
#!/bin/bash
# Script de validación final del Lab 05-00-01
echo "╔══════════════════════════════════════════════════════════════╗"
echo "║      VALIDACIÓN FINAL - Lab 05-00-01                       ║"
echo "╚══════════════════════════════════════════════════════════════╝"
echo ""

BASE="/opt/sysreport/text_processing_exercises"
PASS=0
FAIL=0

check() {
    local desc="$1"
    local file="$2"
    if [ -s "$file" ]; then
        printf "  ✓ %-55s [OK]\n" "$desc"
        ((PASS++))
    else
        printf "  ✗ %-55s [FALLO]\n" "$desc"
        ((FAIL++))
    fi
}

echo "--- Ejercicio 1: Búsqueda de archivos de configuración ---"
check "Reporte de configs generado" "$BASE/ejercicio_1/reporte_configs.txt"
check "Lista de configs 24h" "$BASE/ejercicio_1/configs_modificados_24h.txt"

echo ""
echo "--- Ejercicio 2: Extracción de IPs SSH ---"
check "IPs totales SSH" "$BASE/ejercicio_2/ips_todas_ssh.txt"
check "IPs fallidas" "$BASE/ejercicio_2/ips_fallidas.txt"
check "Reporte SSH" "$BASE/ejercicio_2/reporte_ssh.txt"

echo ""
echo "--- Ejercicio 3: Análisis de uso de disco ---"
check "Reporte de disco" "$BASE/ejercicio_3/reporte_disco.txt"
check "Top 15 /var" "$BASE/ejercicio_3/top15_var.txt"

echo ""
echo "--- Ejercicio 4: Procesamiento de /etc/passwd ---"
check "Usuarios regulares" "$BASE/ejercicio_4/usuarios_regulares.txt"
check "CSV de auditoría" "$BASE/ejercicio_4/usuarios_auditoria.csv"
check "Estadísticas de shells" "$BASE/ejercicio_4/estadisticas_shells.txt"

echo ""
echo "--- Ejercicio 5: Expresiones regulares ---"
check "IPs válidas extraídas" "$BASE/ejercicio_5/ips_validas.txt"
check "Emails válidos" "$BASE/ejercicio_5/emails_validos.txt"
check "Fechas ISO" "$BASE/ejercicio_5/fechas_iso.txt"
check "MACs válidas" "$BASE/ejercicio_5/macs_validas.txt"

echo ""
echo "--- Ejercicio 6: Reporte de servicios ---"
check "Servicios activos" "$BASE/ejercicio_6/servicios_activos.txt"
check "Reporte de servicios" "$BASE/ejercicio_6/reporte_servicios.txt"

echo ""
echo "--- Ejercicio 7: Compresión de logs ---"
check "Comparativa de compresión" "$BASE/ejercicio_7/comparativa_compresion.txt"
check "Script de backup" "$BASE/ejercicio_7/backup_logs.sh"

echo ""
echo "--- Ejercicio 8: Comparación de archivos ---"
check "Diff unificado" "$BASE/ejercicio_8/diferencias_unified.txt"
check "Reporte de cambios" "$BASE/ejercicio_8/reporte_cambios.txt"
check "Comparación lado a lado" "$BASE/ejercicio_8/comparacion_lado_a_lado.txt"

echo ""
echo "═══════════════════════════════════════════════════════"
echo "  RESULTADOS: $PASS aprobados / $FAIL fallidos de $((PASS + FAIL)) total"
echo "═══════════════════════════════════════════════════════"

if [ $FAIL -eq 0 ]; then
    echo "  🎉 ¡LABORATORIO COMPLETADO EXITOSAMENTE!"
else
    echo "  ⚠️  Revisa los ejercicios con fallo y repite los pasos."
fi
```

Guarda y ejecuta:

```bash
cat > /opt/sysreport/text_processing_exercises/validacion_final.sh << 'HEREDOC'
# (pegar el script anterior aquí)
HEREDOC
chmod +x /opt/sysreport/text_processing_exercises/validacion_final.sh
bash /opt/sysreport/text_processing_exercises/validacion_final.sh
```

**Resultado esperado:** Todos los checks deben mostrar `[OK]`.

---

## Resolución de Problemas

### Problema 1: `grep` no encuentra coincidencias con expresiones regulares extendidas

**Síntomas:** Al ejecutar un `grep` con patrones como `(error|fail)`, el comando no devuelve resultados aunque los patrones existen en el archivo.

**Causa:** Se está usando `grep` básico que interpreta los paréntesis y la barra vertical como caracteres literales. Las expresiones regulares extendidas requieren `grep -E` (o `egrep`).

**Solución:**

```bash
# INCORRECTO - grep básico trata () y | como literales
grep '(error|fail)' /var/log/syslog

# CORRECTO - usar -E para ERE
grep -E '(error|fail)' /var/log/syslog

# ALTERNATIVA - escapar metacaracteres en BRE
grep '\(error\|fail\)' /var/log/syslog
```

---

### Problema 2: `tar` falla con "Cannot stat: No such file or directory" al comprimir logs

**Síntomas:** Al ejecutar `tar -czvf backup.tar.gz /var/log/*.log`, se obtienen mensajes de error `tar: /var/log/*.log: Cannot stat: No such file or directory` o el archivo resultante está vacío.

**Causa:** El glob `*.log` se expande en el shell antes de pasarse a `tar`. Si se ejecuta con `sudo` y el glob no coincide con archivos accesibles por el usuario actual, o si no hay archivos `.log` directamente en `/var/log/` (pueden estar en subdirectorios), la expansión falla.

**Solución:**

```bash
# Opción 1: Usar find + xargs para garantizar que se encuentren los archivos
sudo find /var/log -maxdepth 1 -type f -name "*.log" -print0 | \
    sudo xargs -0 tar -czvf backup.tar.gz

# Opción 2: Ejecutar todo el comando como root para que el glob se expanda correctamente
sudo bash -c 'tar -czvf /tmp/backup.tar.gz /var/log/*.log'

# Opción 3: Verificar primero que existen archivos
ls /var/log/*.log 2>/dev/null && \
    sudo tar -czvf backup.tar.gz /var/log/*.log || \
    echo "No se encontraron archivos .log en /var/log/"
```

---

## Limpieza

Los archivos generados en este laboratorio son necesarios para el Lab 07 (scripting Bash). **No elimines** el directorio `/opt/sysreport/text_processing_exercises/`.

Si necesitas liberar espacio de los archivos tar de prueba:

```bash
# Eliminar solo los archivos tar grandes del ejercicio 7 (conservar el script)
rm -f /opt/sysreport/text_processing_exercises/ejercicio_7/logs_sistema.tar
rm -f /opt/sysreport/text_processing_exercises/ejercicio_7/logs_sistema.tar.bz2

# Verificar espacio recuperado
du -sh /opt/sysreport/text_processing_exercises/
```

Para una limpieza completa (solo si no continuarás con labs posteriores):

```bash
# ⚠️ SOLO si no necesitas los resultados para el Lab 07
# sudo rm -rf /opt/sysreport/text_processing_exercises/
```

---

## Resumen

En este laboratorio aplicaste herramientas GNU/Linux para resolver ocho escenarios reales de administración de sistemas:

| Ejercicio | Herramientas principales | Habilidad practicada |
|-----------|--------------------------|---------------------|
| 1 | `find`, redirecciones, `wc` | Búsqueda avanzada por criterios de tiempo |
| 2 | `grep -oE`, `sort`, `uniq`, `awk` | Extracción de datos con regex y estadísticas |
| 3 | `du`, `sort`, `awk`, `df` | Análisis y formateo de información de disco |
| 4 | `awk -F:`, `sort`, `getent` | Procesamiento de campos delimitados |
| 5 | `grep -E`, regex POSIX ERE | Validación de datos con expresiones regulares |
| 6 | `systemctl`, `sed`, `sort`, `awk` | Transformación de texto y generación de reportes |
| 7 | `tar`, `gzip`, `bzip2`, `xz`, `find` | Archivado, compresión y respaldos |
| 8 | `diff`, `comm`, `sdiff`, `sed` | Comparación y análisis de cambios |

### Conceptos clave reforzados

- **Tuberías (`|`)**: encadenar la salida de un comando como entrada del siguiente
- **Redirecciones (`>`, `>>`, `2>`, `&>`)**: controlar el flujo de datos y errores
- **Expresiones regulares extendidas**: patrones potentes para búsqueda y validación
- **`find` + `-exec`/`xargs`**: búsqueda y acción combinadas en un solo flujo
- **Comparación de archivos**: `diff` para cambios línea a línea, `comm` para conjuntos ordenados

### Recursos adicionales

- `man grep` — Referencia completa de opciones y sintaxis de regex
- `info sed` — Manual detallado de stream editor
- `man 7 regex` — Documentación de expresiones regulares POSIX
- GNU Coreutils Manual: https://www.gnu.org/software/coreutils/manual/
- The AWK Programming Language (Aho, Kernighan, Weinberger)
