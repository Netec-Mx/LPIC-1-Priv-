# Configurar y validar servicios de sincronización horaria, registros del sistema, correo electrónico e impresión

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 89 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |

## Descripción General

Este laboratorio final integra la configuración de cuatro servicios de infraestructura esenciales en un entorno Linux: sincronización horaria (Chrony/NTP), gestión centralizada de registros (rsyslog/journald con logrotate), correo electrónico local (Postfix) e impresión en red (CUPS con impresora virtual PDF). Se validará la integración completa ejecutando los scripts de automatización del Lab 07 y verificando que los logs se centralizan, las alertas por correo se envían y los reportes de impresión se generan correctamente.

## Objetivos de Aprendizaje

- [ ] Configurar Chrony 4.2 como servidor NTP en srv-linux-01 y como cliente en srv-linux-02 para sincronización horaria precisa en la red 192.168.100.0/24.
- [ ] Implementar rsyslog 8.2112.0 con reglas personalizadas para centralizar logs remotos y configurar logrotate con rotación diaria y retención de 30 días.
- [ ] Configurar Postfix 3.6.4 en modo Internet Site para envío de alertas locales cuando los scripts de monitoreo detecten fallas en servicios.
- [ ] Instalar y administrar CUPS 2.4.1 con una impresora virtual PDF accesible para el grupo `operators`.
- [ ] Validar la integración completa de todos los servicios configurados mediante pruebas funcionales end-to-end.

## Prerrequisitos

### Conocimientos Previos

- Administración básica de servicios con systemd (systemctl enable/start/restart/status)
- Edición de archivos de configuración en Linux (vi/nano)
- Conceptos de red: puertos, protocolos UDP/TCP, firewalls con ufw
- Familiaridad con los scripts del Lab 07 (service_monitor.sh en /opt/scripts/)
- Usuarios y grupos del Lab 09 (grupo `operators` existente)

### Acceso y Recursos

- Acceso root o sudo en srv-linux-01 (192.168.100.10) y srv-linux-02 (192.168.100.20)
- Conectividad de red validada entre ambas máquinas virtuales
- Acceso a internet desde srv-linux-01 (adaptador NAT) para descarga de paquetes
- Puertos UDP 514, UDP 123, TCP 25 y TCP 631 sin bloquear en la red interna

## Entorno de Laboratorio

### Máquinas Virtuales

| Hostname | SO | IP | Rol en este lab |
|----------|----|----|-----------------|
| srv-linux-01 | Ubuntu 22.04.4 LTS | 192.168.100.10 | Servidor NTP, receptor syslog, servidor Postfix, servidor CUPS |
| srv-linux-02 | Ubuntu 22.04.4 LTS | 192.168.100.20 | Cliente NTP, emisor syslog remoto |

### Software Requerido

| Paquete | Versión | Propósito |
|---------|---------|-----------|
| chrony | 4.2 | Sincronización NTP |
| rsyslog | 8.2112.0 | Gestión de logs |
| logrotate | 3.19.0 | Rotación de logs |
| postfix | 3.6.4 | Servidor de correo local |
| mailutils | 1:3.14 | Cliente de correo CLI |
| cups | 2.4.1 | Servicio de impresión |
| cups-pdf | 3.0.1 | Impresora virtual PDF |
| ufw | 0.36.1 | Firewall |

### Preparación Inicial del Entorno

Ejecutar en **ambas máquinas** como usuario `sysadmin`:

```bash
# Verificar conectividad entre máquinas
ping -c 3 192.168.100.10
ping -c 3 192.168.100.20

# Verificar que el usuario sysadmin tiene privilegios sudo
sudo whoami
```

Ejecutar en **srv-linux-01** para garantizar acceso a repositorios:

```bash
# Actualizar índice de paquetes
sudo apt update
```

Ejecutar en **srv-linux-02**:

```bash
# Si srv-linux-02 no tiene acceso directo a internet, configurar ruta a través de srv-linux-01
# o actualizar desde caché local si está disponible
sudo apt update
```

---

## Procedimiento Paso a Paso

### Paso 1: Configurar Chrony como Servidor NTP en srv-linux-01

**Objetivo:** Configurar srv-linux-01 como servidor NTP autoritativo para la red interna 192.168.100.0/24.

**Instrucciones:**

1. Instalar Chrony en srv-linux-01:

```bash
sudo apt install chrony -y
```

2. Crear una copia de respaldo del archivo de configuración original:

```bash
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak
```

3. Editar el archivo de configuración de Chrony:

```bash
sudo nano /etc/chrony/chrony.conf
```

4. Reemplazar el contenido con la siguiente configuración:

```bash
# Fuentes NTP públicas para srv-linux-01
pool ntp.ubuntu.com iburst maxsources 4
pool 0.ubuntu.pool.ntp.org iburst maxsources 2
pool 1.ubuntu.pool.ntp.org iburst maxsources 2

# Archivo de deriva del reloj
driftfile /var/lib/chrony/chrony.drift

# Permitir salto de hora si el desfase es mayor a 1 segundo (máximo 3 intentos al inicio)
makestep 1.0 3

# Sincronizar el RTC del hardware
rtcsync

# Permitir que la red interna del laboratorio use este servidor como fuente NTP
allow 192.168.100.0/24

# Servir hora local como respaldo si se pierde conectividad externa (estrato 10)
local stratum 10

# Directorio de logs de Chrony
logdir /var/log/chrony

# Registrar mediciones y estadísticas
log measurements statistics tracking
```

5. Reiniciar el servicio Chrony:

```bash
sudo systemctl restart chrony
```

6. Habilitar el servicio para inicio automático:

```bash
sudo systemctl enable chrony
```

7. Configurar el firewall para permitir consultas NTP desde la red interna:

```bash
sudo ufw allow from 192.168.100.0/24 to any port 123 proto udp comment "NTP para red interna"
sudo ufw reload
```

**Salida Esperada:**

```
Rule added
Rule added (v6)
Firewall reloaded
```

**Verificación:**

```bash
# Verificar que chronyd está activo
sudo systemctl status chrony | grep -E "Active|Main PID"

# Verificar que el puerto UDP 123 está escuchando
sudo ss -ulnp | grep :123

# Verificar el estado de sincronización
chronyc tracking
```

Salida esperada de `chronyc tracking` (los valores variarán):

```
Reference ID    : XXXXXXXX (ntp.ubuntu.com)
Stratum         : 2
...
Leap status     : Normal
```

```bash
# Verificar las fuentes NTP
chronyc sources -v
```

Debe mostrar al menos una fuente con el símbolo `*` (seleccionada) o `^*`.

---

### Paso 2: Configurar Chrony como Cliente NTP en srv-linux-02

**Objetivo:** Configurar srv-linux-02 para sincronizar su reloj exclusivamente contra srv-linux-01.

**Instrucciones:**

1. Instalar Chrony en srv-linux-02:

```bash
sudo apt install chrony -y
```

2. Respaldar la configuración original:

```bash
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak
```

3. Editar el archivo de configuración:

```bash
sudo nano /etc/chrony/chrony.conf
```

4. Reemplazar con la siguiente configuración de cliente:

```bash
# Servidor NTP interno: srv-linux-01
server 192.168.100.10 iburst prefer

# Archivo de deriva
driftfile /var/lib/chrony/chrony.drift

# Permitir salto de hora al inicio
makestep 1.0 3

# Sincronizar RTC
rtcsync

# Logs
logdir /var/log/chrony
log measurements statistics tracking
```

5. Reiniciar y habilitar el servicio:

```bash
sudo systemctl restart chrony
sudo systemctl enable chrony
```

6. Forzar una sincronización inmediata:

```bash
sudo chronyc makestep
```

**Salida Esperada:**

```
200 OK
```

**Verificación:**

```bash
# Verificar que srv-linux-01 es la fuente activa
chronyc sources -v
```

Salida esperada:

```
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 192.168.100.10                2   6    17     1   +XXus[+XXus] +/-  XXms
```

El `^*` confirma que 192.168.100.10 es la fuente seleccionada.

```bash
# Verificar sincronización con timedatectl
timedatectl status | grep -E "synchronized|NTP"
```

Salida esperada:

```
System clock synchronized: yes
              NTP service: active
```

---

### Paso 3: Configurar rsyslog como Receptor de Logs Remotos en srv-linux-01

**Objetivo:** Habilitar la recepción de logs remotos vía UDP 514 en srv-linux-01 y crear reglas de filtrado personalizadas.

**Instrucciones:**

1. Verificar que rsyslog está instalado en srv-linux-01:

```bash
rsyslogd -v | head -2
```

2. Editar la configuración principal de rsyslog para habilitar la recepción UDP:

```bash
sudo nano /etc/rsyslog.conf
```

3. Descomentar (o agregar) las siguientes líneas en la sección de módulos:

```bash
# Habilitar recepción de syslog vía UDP puerto 514
module(load="imudp")
input(type="imudp" port="514")
```

4. Crear un archivo de configuración personalizado para logs remotos:

```bash
sudo nano /etc/rsyslog.d/50-remote-logs.conf
```

5. Agregar el siguiente contenido:

```bash
# Plantilla para separar logs por hostname remoto
template(name="RemoteHostLogs" type="string"
    string="/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log")

# Regla: almacenar logs remotos en directorios separados por host
if $fromhost-ip != '127.0.0.1' then {
    action(type="omfile" dynaFile="RemoteHostLogs")
    stop
}
```

6. Crear un archivo de configuración para filtrar logs de scripts administrativos:

```bash
sudo nano /etc/rsyslog.d/60-admin-scripts.conf
```

7. Agregar el contenido:

```bash
# Filtrar mensajes del tag 'admin_scripts' a archivo dedicado
if $syslogtag contains 'admin_scripts' then {
    action(type="omfile" file="/var/log/admin_scripts.log")
    stop
}
```

8. Crear los directorios necesarios:

```bash
sudo mkdir -p /var/log/remote
sudo chown syslog:adm /var/log/remote
```

9. Verificar la sintaxis de la configuración:

```bash
sudo rsyslogd -N1
```

**Salida Esperada:**

```
rsyslogd: version X.XXXX.X, config validation run (level 1), master config /etc/rsyslog.conf
rsyslogd: End of config validation run. Bye.
```

10. Reiniciar rsyslog:

```bash
sudo systemctl restart rsyslog
```

11. Configurar el firewall para permitir syslog remoto:

```bash
sudo ufw allow from 192.168.100.0/24 to any port 514 proto udp comment "Syslog remoto UDP"
sudo ufw reload
```

**Verificación:**

```bash
# Verificar que el puerto UDP 514 está escuchando
sudo ss -ulnp | grep :514

# Verificar estado del servicio
sudo systemctl status rsyslog | grep Active
```

---

### Paso 4: Configurar rsyslog como Emisor de Logs Remotos en srv-linux-02

**Objetivo:** Configurar srv-linux-02 para enviar sus logs a srv-linux-01 vía UDP 514.

**Instrucciones:**

1. Crear un archivo de configuración de reenvío en srv-linux-02:

```bash
sudo nano /etc/rsyslog.d/50-forward-to-central.conf
```

2. Agregar el siguiente contenido:

```bash
# Reenviar todos los logs a srv-linux-01 vía UDP
*.* @192.168.100.10:514
```

> **Nota:** El símbolo `@` indica UDP. Para TCP se usaría `@@`.

3. Reiniciar rsyslog en srv-linux-02:

```bash
sudo systemctl restart rsyslog
```

4. Generar un mensaje de prueba desde srv-linux-02:

```bash
logger -t admin_scripts "Prueba de log remoto desde srv-linux-02"
```

**Verificación (ejecutar en srv-linux-01):**

```bash
# Verificar que se recibió el log remoto
sudo ls /var/log/remote/

# Verificar el contenido del mensaje de prueba
sudo grep "Prueba de log remoto" /var/log/remote/srv-linux-02/*.log 2>/dev/null || \
sudo grep "Prueba de log remoto" /var/log/syslog
```

Debe aparecer el mensaje enviado desde srv-linux-02.

---

### Paso 5: Configurar Logrotate para /var/log/admin_scripts.log

**Objetivo:** Implementar rotación automática diaria con retención de 30 días y compresión gzip.

**Instrucciones (en srv-linux-01):**

1. Crear el archivo de configuración de logrotate:

```bash
sudo nano /etc/logrotate.d/admin_scripts
```

2. Agregar el siguiente contenido:

```bash
/var/log/admin_scripts.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 syslog adm
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

3. Crear el archivo de log si no existe (para que logrotate no genere errores):

```bash
sudo touch /var/log/admin_scripts.log
sudo chown syslog:adm /var/log/admin_scripts.log
sudo chmod 0640 /var/log/admin_scripts.log
```

4. Probar la configuración de logrotate en modo debug:

```bash
sudo logrotate -d /etc/logrotate.d/admin_scripts
```

**Salida Esperada:**

```
reading config file /etc/logrotate.d/admin_scripts
...
considering log /var/log/admin_scripts.log
  Now: ...
  log does not need rotating (log is empty)
```

5. Forzar una rotación para validar el funcionamiento:

```bash
# Agregar contenido de prueba al log
echo "$(date) - Test entry for logrotate validation" | sudo tee -a /var/log/admin_scripts.log

# Forzar rotación
sudo logrotate -f /etc/logrotate.d/admin_scripts
```

**Verificación:**

```bash
# Verificar que se creó el archivo rotado
ls -la /var/log/admin_scripts.log*
```

Salida esperada: debe existir `admin_scripts.log` (nuevo, vacío) y `admin_scripts.log.1` (con el contenido anterior).

---

### Paso 6: Configurar Postfix como Servidor de Correo Local en srv-linux-01

**Objetivo:** Instalar y configurar Postfix en modo Internet Site con dominio syslab.local para envío de alertas del sistema.

**Instrucciones:**

1. Instalar Postfix y mailutils:

```bash
# Preconfigurar las respuestas de debconf para evitar la interfaz interactiva
sudo debconf-set-selections <<< "postfix postfix/mailname string srv-linux-01.syslab.local"
sudo debconf-set-selections <<< "postfix postfix/main_mailer_type string 'Internet Site'"

sudo apt install postfix mailutils -y
```

> **Nota:** Si aparece la interfaz interactiva, seleccionar "Internet Site" y escribir `srv-linux-01.syslab.local` como nombre del sistema de correo.

2. Editar la configuración principal de Postfix:

```bash
sudo nano /etc/postfix/main.cf
```

3. Verificar y ajustar los siguientes parámetros (modificar los que difieran):

```bash
# Nombre del host
myhostname = srv-linux-01.syslab.local

# Dominio de origen para correos salientes
myorigin = syslab.local

# Dominios para los que este servidor acepta correo
mydestination = $myhostname, srv-linux-01, localhost.localdomain, localhost, syslab.local

# Interfaces en las que escucha (loopback + red interna)
inet_interfaces = loopback-only

# Protocolos
inet_protocols = ipv4

# Redes confiables
mynetworks = 127.0.0.0/8, 192.168.100.0/24

# Directorio de buzones locales (formato Maildir)
home_mailbox = Maildir/

# Tamaño máximo de buzón (50 MB)
mailbox_size_limit = 51200000

# Desactivar SMTP autenticado externo (solo uso local)
smtpd_relay_restrictions = permit_mynetworks, reject_unauth_destination
```

4. Reiniciar Postfix:

```bash
sudo systemctl restart postfix
sudo systemctl enable postfix
```

5. Crear el directorio Maildir para el usuario sysadmin:

```bash
mkdir -p ~/Maildir/{new,cur,tmp}
```

6. Enviar un correo de prueba:

```bash
echo "Prueba de correo local desde srv-linux-01" | mail -s "Test Postfix" sysadmin@syslab.local
```

**Verificación:**

```bash
# Verificar que Postfix está activo
sudo systemctl status postfix | grep Active

# Verificar la cola de correo
mailq

# Verificar que el correo llegó al buzón
ls ~/Maildir/new/

# Leer el correo (si mailutils está configurado para Maildir)
cat ~/Maildir/new/*
```

Si `mailq` muestra la cola vacía y hay un archivo en `~/Maildir/new/`, el correo se entregó exitosamente.

```bash
# Verificar logs de correo
sudo tail -5 /var/log/mail.log
```

Debe mostrar una línea con `status=sent` o `delivered`.

---

### Paso 7: Integrar Alertas por Correo en el Script de Monitoreo

**Objetivo:** Modificar el script service_monitor.sh para enviar alertas por correo cuando un servicio falle.

**Instrucciones:**

1. Verificar que el script existe:

```bash
ls -la /opt/scripts/service_monitor.sh
```

2. Crear una versión mejorada del script con notificación por correo:

```bash
sudo nano /opt/scripts/service_monitor_mail.sh
```

3. Agregar el siguiente contenido:

```bash
#!/bin/bash
# service_monitor_mail.sh - Monitoreo de servicios con alertas por correo
# Lab 10 - Integración con Postfix

LOGFILE="/var/log/admin_scripts.log"
MAIL_TO="sysadmin@syslab.local"
HOSTNAME=$(hostname)
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# Servicios a monitorear
SERVICES=("chrony" "rsyslog" "postfix" "cups")

# Función de logging con syslog
log_message() {
    local level=$1
    local message=$2
    echo "${DATE} [${level}] ${message}" >> ${LOGFILE}
    logger -t admin_scripts "${level}: ${message}"
}

# Verificar cada servicio
for service in "${SERVICES[@]}"; do
    if systemctl is-active --quiet "${service}"; then
        log_message "INFO" "Servicio ${service} está activo en ${HOSTNAME}"
    else
        log_message "ERROR" "Servicio ${service} está INACTIVO en ${HOSTNAME}"
        
        # Enviar alerta por correo
        echo "ALERTA: El servicio ${service} está inactivo en ${HOSTNAME}.
Fecha: ${DATE}
Acción requerida: Verificar y reiniciar el servicio.

-- Generado por service_monitor_mail.sh" | \
        mail -s "[ALERTA] Servicio ${service} caído en ${HOSTNAME}" ${MAIL_TO}
        
        log_message "INFO" "Alerta enviada por correo a ${MAIL_TO} para servicio ${service}"
    fi
done

log_message "INFO" "Verificación de servicios completada en ${HOSTNAME}"
```

4. Asignar permisos de ejecución:

```bash
sudo chmod 755 /opt/scripts/service_monitor_mail.sh
sudo chown sysadmin:sysadmin /opt/scripts/service_monitor_mail.sh
```

5. Ejecutar el script para validar el funcionamiento normal:

```bash
sudo /opt/scripts/service_monitor_mail.sh
```

6. Simular una falla para probar la alerta por correo (detener temporalmente un servicio):

```bash
# Detener cups temporalmente para generar una alerta
sudo systemctl stop cups

# Ejecutar el monitor
sudo /opt/scripts/service_monitor_mail.sh

# Verificar que se envió el correo de alerta
ls ~/Maildir/new/
cat ~/Maildir/new/* | grep -A5 "ALERTA"

# Reiniciar cups inmediatamente
sudo systemctl start cups
```

**Verificación:**

```bash
# Verificar el log
sudo tail -10 /var/log/admin_scripts.log

# Verificar que el correo de alerta llegó
find ~/Maildir/new/ -newer /opt/scripts/service_monitor_mail.sh -exec cat {} \;
```

---

### Paso 8: Instalar y Configurar CUPS con Impresora Virtual PDF

**Objetivo:** Instalar CUPS 2.4.1 y cups-pdf, configurar una impresora virtual PDF accesible para el grupo `operators`.

**Instrucciones:**

1. Instalar CUPS y cups-pdf en srv-linux-01:

```bash
sudo apt install cups cups-pdf -y
```

2. Habilitar e iniciar el servicio CUPS:

```bash
sudo systemctl enable --now cups
```

3. Respaldar la configuración original de CUPS:

```bash
sudo cp /etc/cups/cupsd.conf /etc/cups/cupsd.conf.bak
```

4. Editar la configuración de CUPS para permitir acceso desde la red interna:

```bash
sudo nano /etc/cups/cupsd.conf
```

5. Realizar los siguientes cambios (buscar y modificar las secciones correspondientes):

```bash
# Escuchar en todas las interfaces (o específicamente en la red interna)
Listen 192.168.100.10:631
Listen localhost:631
Listen /run/cups/cups.sock

# En la sección <Location />
<Location />
  Order allow,deny
  Allow localhost
  Allow 192.168.100.0/24
</Location>

# En la sección <Location /admin>
<Location /admin>
  Order allow,deny
  Allow localhost
  Allow 192.168.100.0/24
</Location>

# En la sección <Location /admin/conf>
<Location /admin/conf>
  AuthType Default
  Require user @SYSTEM
  Order allow,deny
  Allow localhost
  Allow 192.168.100.0/24
</Location>

# Habilitar compartir impresoras
Browsing On
BrowseLocalProtocols dnssd

# Configuración de acceso por defecto
DefaultAuthType Basic
```

6. Agregar el usuario sysadmin al grupo lpadmin para administración de CUPS:

```bash
sudo usermod -aG lpadmin sysadmin
```

7. Reiniciar CUPS:

```bash
sudo systemctl restart cups
```

8. Configurar el firewall para CUPS:

```bash
sudo ufw allow from 192.168.100.0/24 to any port 631 proto tcp comment "CUPS impresion"
sudo ufw reload
```

9. Verificar que cups-pdf creó automáticamente la impresora virtual:

```bash
lpstat -p -d
```

Si la impresora PDF no aparece, agregarla manualmente:

```bash
sudo lpadmin -p PDF -v cups-pdf:/ -E -m lsb/usr/cups-pdf/CUPS-PDF_opt.ppd
sudo lpadmin -d PDF
```

> **Nota:** La ruta del PPD puede variar. Buscar con: `find / -name "*cups-pdf*" -name "*.ppd" 2>/dev/null` o usar `-m everywhere`.

Si no se encuentra el PPD, usar el driver genérico:

```bash
sudo lpadmin -p PDF -v cups-pdf:/ -E -m raw
```

10. Configurar el directorio de salida de cups-pdf:

```bash
sudo nano /etc/cups/cups-pdf.conf
```

Verificar que la línea `Out` apunta a un directorio accesible:

```bash
Out ${HOME}/PDF
```

11. Crear el directorio PDF para el usuario sysadmin:

```bash
mkdir -p ~/PDF
```

**Verificación:**

```bash
# Listar impresoras disponibles
lpstat -p -d

# Verificar estado del servicio
sudo systemctl status cups | grep Active
```

Salida esperada de `lpstat -p -d`:

```
printer PDF is idle.  enabled since ...
system default destination: PDF
```

---

### Paso 9: Configurar Políticas de Acceso CUPS para el Grupo operators

**Objetivo:** Restringir el uso de la impresora PDF al grupo `operators`.

**Instrucciones:**

1. Verificar que el grupo `operators` existe:

```bash
getent group operators
```

Si no existe, crearlo y agregar a sysadmin:

```bash
sudo groupadd operators
sudo usermod -aG operators sysadmin
```

2. Configurar la política de acceso en la impresora PDF:

```bash
sudo lpadmin -p PDF -u allow:@operators
```

3. Verificar la configuración de acceso:

```bash
sudo cat /etc/cups/printers.conf | grep -A5 "PDF"
```

Debe incluir una línea similar a:

```
AllowUser @operators
```

4. Probar la impresión enviando un archivo de prueba:

```bash
# Crear un archivo de prueba
echo "Reporte de prueba - Lab 10 - $(date)" > /tmp/test_print.txt

# Enviar a la impresora PDF
lp -d PDF /tmp/test_print.txt
```

**Salida Esperada:**

```
request id is PDF-1 (1 file(s))
```

**Verificación:**

```bash
# Verificar que se generó el PDF (puede tardar unos segundos)
sleep 3
ls -la ~/PDF/

# Verificar el estado del trabajo de impresión
lpstat -W completed
```

---

### Paso 10: Validación de Integración Completa

**Objetivo:** Ejecutar una prueba end-to-end que valide la interacción entre todos los servicios configurados.

**Instrucciones:**

1. Verificar el estado de todos los servicios en srv-linux-01:

```bash
echo "=== Estado de Servicios en srv-linux-01 ==="
for svc in chrony rsyslog postfix cups; do
    status=$(systemctl is-active $svc)
    echo "  $svc: $status"
done
```

**Salida Esperada:**

```
=== Estado de Servicios en srv-linux-01 ===
  chrony: active
  rsyslog: active
  postfix: active
  cups: active
```

2. Verificar la sincronización NTP desde srv-linux-02 (ejecutar en srv-linux-02):

```bash
chronyc sources | grep "192.168.100.10"
```

Debe mostrar `^*` indicando sincronización activa.

3. Verificar la recepción de logs remotos (enviar desde srv-linux-02):

```bash
# En srv-linux-02:
logger -t admin_scripts "Validación final - log remoto desde srv-linux-02"
```

```bash
# En srv-linux-01 - verificar recepción:
sleep 2
sudo grep "Validación final" /var/log/remote/srv-linux-02/*.log 2>/dev/null || \
sudo grep "Validación final" /var/log/syslog
```

4. Ejecutar el script de monitoreo completo en srv-linux-01:

```bash
sudo /opt/scripts/service_monitor_mail.sh
```

5. Verificar los resultados integrados:

```bash
echo "=== Verificación de Logs ==="
sudo tail -5 /var/log/admin_scripts.log

echo ""
echo "=== Verificación de Correo ==="
ls ~/Maildir/new/ | wc -l
echo "correos en buzón"

echo ""
echo "=== Verificación NTP ==="
chronyc tracking | grep -E "Stratum|System time|Leap"

echo ""
echo "=== Verificación CUPS ==="
lpstat -p
```

6. Generar un reporte de estado y enviarlo a la impresora PDF:

```bash
# Crear script de reporte final
cat << 'EOF' > /tmp/reporte_final.sh
#!/bin/bash
REPORT="/tmp/reporte_lab10_$(date +%Y%m%d_%H%M%S).txt"

echo "============================================" > $REPORT
echo "  REPORTE DE INFRAESTRUCTURA - LAB 10" >> $REPORT
echo "  Fecha: $(date)" >> $REPORT
echo "  Host: $(hostname)" >> $REPORT
echo "============================================" >> $REPORT
echo "" >> $REPORT

echo "--- SINCRONIZACIÓN NTP ---" >> $REPORT
chronyc tracking >> $REPORT
echo "" >> $REPORT

echo "--- SERVICIOS ACTIVOS ---" >> $REPORT
for svc in chrony rsyslog postfix cups; do
    echo "  $svc: $(systemctl is-active $svc)" >> $REPORT
done
echo "" >> $REPORT

echo "--- ÚLTIMOS LOGS ADMIN ---" >> $REPORT
tail -10 /var/log/admin_scripts.log >> $REPORT 2>/dev/null
echo "" >> $REPORT

echo "--- COLA DE CORREO ---" >> $REPORT
mailq >> $REPORT
echo "" >> $REPORT

echo "============================================" >> $REPORT
echo "  FIN DEL REPORTE" >> $REPORT
echo "============================================" >> $REPORT

# Imprimir el reporte
lp -d PDF $REPORT
echo "Reporte generado: $REPORT"
echo "Enviado a impresora PDF"
EOF

chmod +x /tmp/reporte_final.sh
/tmp/reporte_final.sh
```

7. Verificar la generación del PDF:

```bash
sleep 3
ls -la ~/PDF/
```

**Salida Esperada:**

Debe existir al menos un archivo PDF en el directorio `~/PDF/`.

---

## Validación y Pruebas

Ejecutar la siguiente secuencia de validación completa en **srv-linux-01**:

```bash
echo "╔══════════════════════════════════════════════════╗"
echo "║   VALIDACIÓN COMPLETA - LAB 10                  ║"
echo "╠══════════════════════════════════════════════════╣"

# Test 1: NTP Server
echo -n "║ [1] Chrony servidor NTP activo:        "
if systemctl is-active --quiet chrony && chronyc clients 2>/dev/null | grep -q "192.168.100" 2>/dev/null; then
    echo "  ✓ PASS ║"
else
    # Verificar al menos que chrony está activo y allow está configurado
    if systemctl is-active --quiet chrony && grep -q "allow 192.168.100" /etc/chrony/chrony.conf; then
        echo "  ✓ PASS ║"
    else
        echo "  ✗ FAIL ║"
    fi
fi

# Test 2: rsyslog receptor activo
echo -n "║ [2] rsyslog escuchando UDP 514:        "
if sudo ss -ulnp | grep -q ":514"; then
    echo "  ✓ PASS ║"
else
    echo "  ✗ FAIL ║"
fi

# Test 3: Logrotate configurado
echo -n "║ [3] Logrotate para admin_scripts:      "
if [ -f /etc/logrotate.d/admin_scripts ]; then
    echo "  ✓ PASS ║"
else
    echo "  ✗ FAIL ║"
fi

# Test 4: Postfix activo
echo -n "║ [4] Postfix servicio activo:           "
if systemctl is-active --quiet postfix; then
    echo "  ✓ PASS ║"
else
    echo "  ✗ FAIL ║"
fi

# Test 5: CUPS activo con impresora PDF
echo -n "║ [5] CUPS con impresora PDF:            "
if systemctl is-active --quiet cups && lpstat -p PDF 2>/dev/null | grep -q "idle\|enabled"; then
    echo "  ✓ PASS ║"
else
    echo "  ✗ FAIL ║"
fi

# Test 6: Correo local funcional
echo -n "║ [6] Entrega de correo local:           "
echo "Validación automática Lab10" | mail -s "Test Auto" sysadmin@syslab.local 2>/dev/null
sleep 2
if ls ~/Maildir/new/ 2>/dev/null | grep -q .; then
    echo "  ✓ PASS ║"
else
    echo "  ✗ FAIL ║"
fi

# Test 7: Script de monitoreo con correo
echo -n "║ [7] Script monitor con alertas:        "
if [ -x /opt/scripts/service_monitor_mail.sh ]; then
    echo "  ✓ PASS ║"
else
    echo "  ✗ FAIL ║"
fi

echo "╠══════════════════════════════════════════════════╣"
echo "║   Validación completada                         ║"
echo "╚══════════════════════════════════════════════════╝"
```

Ejecutar en **srv-linux-02** la validación del cliente:

```bash
echo "=== Validación srv-linux-02 ==="
echo -n "NTP sincronizado con srv-linux-01: "
if chronyc sources 2>/dev/null | grep -q "\*.*192.168.100.10"; then
    echo "✓ PASS"
else
    echo "✗ FAIL (verificar chronyc sources)"
fi

echo -n "Reenvío syslog configurado: "
if grep -q "192.168.100.10" /etc/rsyslog.d/50-forward-to-central.conf 2>/dev/null; then
    echo "✓ PASS"
else
    echo "✗ FAIL"
fi
```

---

## Solución de Problemas

### Problema 1: Chrony no sincroniza — "No sources" o "Not synchronised"

**Síntomas:**
- `chronyc sources` muestra `?` en todas las fuentes o no muestra ninguna fuente.
- `chronyc tracking` muestra `Leap status: Not synchronised`.
- `timedatectl` muestra `System clock synchronized: no`.

**Causa:**
El firewall está bloqueando el tráfico UDP 123 saliente/entrante, o la configuración del servidor NTP tiene errores de sintaxis. En el caso del cliente (srv-linux-02), puede que srv-linux-01 no tenga la directiva `allow` configurada correctamente.

**Solución:**

```bash
# En srv-linux-01 (servidor): Verificar que allow está configurado
grep "allow" /etc/chrony/chrony.conf

# Verificar que el puerto está abierto
sudo ufw status | grep 123
sudo ss -ulnp | grep chronyd

# Si el puerto no aparece, verificar que chronyd escucha en la interfaz correcta
sudo systemctl restart chrony

# En srv-linux-02 (cliente): Verificar conectividad al puerto NTP
sudo apt install nmap -y
nmap -sU -p 123 192.168.100.10

# Verificar que no hay otro servicio NTP compitiendo
sudo systemctl status systemd-timesyncd
# Si está activo, deshabilitarlo (conflicto con chrony)
sudo systemctl stop systemd-timesyncd
sudo systemctl disable systemd-timesyncd
sudo systemctl restart chrony

# Forzar sincronización
sudo chronyc makestep
chronyc sources -v
```

### Problema 2: Los logs remotos no llegan a srv-linux-01

**Síntomas:**
- El directorio `/var/log/remote/` está vacío.
- `logger` desde srv-linux-02 no genera archivos en srv-linux-01.
- No hay errores visibles en rsyslog de ninguno de los dos servidores.

**Causa:**
El módulo `imudp` no está cargado correctamente en rsyslog de srv-linux-01, el firewall bloquea UDP 514, o la regla de reenvío en srv-linux-02 tiene un error de sintaxis (falta el `@` o tiene la IP incorrecta).

**Solución:**

```bash
# En srv-linux-01: Verificar que imudp está cargado
sudo rsyslogd -N1 2>&1 | grep -i error

# Verificar que el puerto está escuchando
sudo ss -ulnp | grep :514
# Si no aparece, verificar la configuración:
grep -n "imudp" /etc/rsyslog.conf
# Asegurar que las líneas NO están comentadas (sin # al inicio)

# Verificar firewall
sudo ufw status verbose | grep 514

# Si el firewall no tiene la regla:
sudo ufw allow from 192.168.100.0/24 to any port 514 proto udp
sudo ufw reload

# En srv-linux-02: Verificar la configuración de reenvío
cat /etc/rsyslog.d/50-forward-to-central.conf
# Debe contener exactamente: *.* @192.168.100.10:514
# (un solo @ para UDP, dos @@ para TCP)

# Probar conectividad UDP al puerto
echo "test" | nc -u -w1 192.168.100.10 514

# Reiniciar rsyslog en ambos servidores
# En srv-linux-01:
sudo systemctl restart rsyslog
# En srv-linux-02:
sudo systemctl restart rsyslog

# Enviar mensaje de prueba y verificar
# En srv-linux-02:
logger -t test_remoto "Mensaje de diagnóstico $(date)"
# En srv-linux-01 (esperar 2 segundos):
sudo tail -20 /var/log/syslog | grep "test_remoto"
sudo find /var/log/remote/ -name "*.log" -exec grep "diagnóstico" {} \;
```

---

## Limpieza

Si es necesario revertir la configuración del laboratorio (por ejemplo, para repetirlo desde cero):

```bash
# === En srv-linux-01 ===

# Restaurar configuración original de Chrony
sudo cp /etc/chrony/chrony.conf.bak /etc/chrony/chrony.conf
sudo systemctl restart chrony

# Eliminar configuraciones personalizadas de rsyslog
sudo rm -f /etc/rsyslog.d/50-remote-logs.conf
sudo rm -f /etc/rsyslog.d/60-admin-scripts.conf
sudo rm -rf /var/log/remote/
sudo systemctl restart rsyslog

# Eliminar configuración de logrotate
sudo rm -f /etc/logrotate.d/admin_scripts

# Desinstalar Postfix (opcional)
# sudo apt purge postfix mailutils -y

# Desinstalar CUPS (opcional)
# sudo apt purge cups cups-pdf -y

# Eliminar reglas de firewall agregadas
sudo ufw delete allow from 192.168.100.0/24 to any port 123 proto udp
sudo ufw delete allow from 192.168.100.0/24 to any port 514 proto udp
sudo ufw delete allow from 192.168.100.0/24 to any port 631 proto tcp
sudo ufw reload

# Eliminar archivos generados
rm -rf ~/Maildir ~/PDF
sudo rm -f /opt/scripts/service_monitor_mail.sh

# === En srv-linux-02 ===

# Restaurar configuración de Chrony
sudo cp /etc/chrony/chrony.conf.bak /etc/chrony/chrony.conf
sudo systemctl restart chrony

# Eliminar reenvío de logs
sudo rm -f /etc/rsyslog.d/50-forward-to-central.conf
sudo systemctl restart rsyslog
```

---

## Resumen

En este laboratorio se configuraron e integraron cuatro servicios fundamentales de infraestructura Linux:

| Servicio | Configuración Realizada | Puerto |
|----------|------------------------|--------|
| **Chrony (NTP)** | Servidor en srv-linux-01, cliente en srv-linux-02 | UDP 123 |
| **rsyslog** | Centralización de logs remotos con filtros personalizados | UDP 514 |
| **Postfix** | Correo local para alertas automáticas del sistema | TCP 25 (local) |
| **CUPS** | Impresora virtual PDF con acceso restringido por grupo | TCP 631 |

**Conceptos clave aplicados:**

- La sincronización horaria es prerrequisito para la correlación de logs entre múltiples servidores.
- La centralización de logs permite auditoría y diagnóstico desde un punto único.
- Las alertas por correo automatizan la respuesta ante fallas de servicios.
- La gestión de impresión con políticas de grupo implementa el principio de mínimo privilegio.

### Recursos Adicionales

- [Documentación oficial de Chrony](https://chrony-project.org/documentation.html)
- [Guía de rsyslog — configuración avanzada](https://www.rsyslog.com/doc/v8-stable/)
- [Postfix — documentación oficial](http://www.postfix.org/documentation.html)
- [CUPS — guía de administración](https://www.cups.org/doc/admin.html)
- [Ubuntu Server Guide — Time Synchronisation](https://documentation.ubuntu.com/server/how-to/networking/timedatectl-and-ntp/)
