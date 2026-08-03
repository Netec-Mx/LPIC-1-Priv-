# Implementar políticas de acceso, administración de usuarios y programación de tareas automáticas para garantizar la seguridad y continuidad operativa del sistema

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 112 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Servidor objetivo** | srv-linux-01 (Ubuntu 22.04.4 LTS) |
| **IP** | 192.168.100.10 |

## Descripción General

En este laboratorio implementarás una estructura organizacional completa de usuarios y grupos en srv-linux-01, configurarás políticas de seguridad de contraseñas mediante módulos PAM, programarás la ejecución automática de los scripts desarrollados en el Lab 07 utilizando `cron` y `at`, y configurarás la localización del sistema para un entorno corporativo hispanohablante. Esta práctica integra directamente los resultados de los laboratorios 06 (cuotas), 07 (scripts) y 08 (usuarios gráficos).

## Objetivos de Aprendizaje

- [ ] Implementar una estructura organizacional de usuarios y grupos con directorios home y permisos apropiados
- [ ] Configurar módulos PAM (pam_pwquality, pam_faillock, pam_pwhistory) para aplicar políticas de seguridad de contraseñas
- [ ] Programar tareas automáticas con `cron` y `at` para ejecutar scripts de administración del sistema
- [ ] Configurar localización (locale) y zona horaria del sistema para un entorno multiidioma corporativo

## Prerrequisitos

### Conocimientos requeridos

- Comprensión de la estructura de archivos `/etc/passwd`, `/etc/shadow` y `/etc/group`
- Uso de comandos `useradd`, `usermod`, `groupadd`, `passwd` y `chage`
- Conocimiento básico de expresiones cron y el servicio crontab
- Familiaridad con la edición de archivos de configuración PAM

### Acceso y recursos necesarios

| Requisito | Detalle |
|-----------|---------|
| Lab 06 completado | Cuotas de disco configuradas en `/home` |
| Lab 07 completado | Scripts `system_inventory.sh`, `log_analyzer.sh`, `storage_report.sh` en `/opt/scripts/` |
| Lab 08 completado | Usuarios gráficos creados e integrados |
| Acceso root/sudo | Usuario `sysadmin` con privilegios sudo |
| Conectividad | Acceso a repositorios para instalar paquetes |

## Entorno del Laboratorio

### Software requerido

| Paquete | Versión | Propósito |
|---------|---------|-----------|
| shadow-utils | 4.8.1 | useradd, usermod, groupadd, chage |
| libpam-pwquality | 1.4.4 | Políticas de complejidad de contraseñas |
| libpam-modules | 1.4.0 | pam_faillock, pam_pwhistory |
| cron | 3.0pl1 | Programación de tareas periódicas |
| at | 3.2.5 | Programación de tareas únicas |
| locales | 2.35 | Configuración de localización |

### Preparación inicial

Conéctate a srv-linux-01 como `sysadmin` y verifica el estado del sistema:

```bash
ssh sysadmin@192.168.100.10
```

```bash
# Verificar que los scripts del Lab 07 existen
ls -la /opt/scripts/{system_inventory.sh,log_analyzer.sh,storage_report.sh}

# Verificar que las cuotas del Lab 06 están activas
sudo repquota -a 2>/dev/null || echo "Verificar configuración de cuotas"

# Verificar conectividad a repositorios
sudo apt update -qq
```

---

## Procedimiento Paso a Paso

### Paso 1: Instalar paquetes necesarios

**Objetivo:** Instalar las dependencias de PAM y herramientas de programación de tareas requeridas para el laboratorio.

**Instrucciones:**

1. Actualiza el índice de paquetes e instala los paquetes necesarios:

```bash
sudo apt update
sudo apt install -y libpam-pwquality libpam-modules at cron locales
```

2. Verifica que los servicios `cron` y `atd` estén activos:

```bash
sudo systemctl enable --now cron
sudo systemctl enable --now atd
```

3. Confirma las versiones instaladas:

```bash
dpkg -l | grep -E "libpam-pwquality|libpam-modules|cron|at "
```

**Salida esperada:**

```
ii  at             3.2.5-1ubuntu1   amd64   Delayed job execution and batch processing
ii  cron           3.0pl1-137ubuntu3 amd64  process scheduling daemon
ii  libpam-modules 1.4.0-11ubuntu2.4 amd64  Pluggable Authentication Modules for PAM
ii  libpam-pwquality 1.4.4-1build2  amd64   PAM module to check password strength
```

**Verificación:**

```bash
sudo systemctl is-active cron && echo "cron OK"
sudo systemctl is-active atd && echo "at OK"
```

---

### Paso 2: Crear la estructura de grupos organizacionales

**Objetivo:** Crear los grupos que representan la estructura departamental de la organización.

**Instrucciones:**

1. Crea los tres grupos organizacionales con GIDs específicos:

```bash
sudo groupadd -g 3001 sysadmins
sudo groupadd -g 3002 developers
sudo groupadd -g 3003 operators
```

2. Verifica la creación en `/etc/group`:

```bash
grep -E "^(sysadmins|developers|operators):" /etc/group
```

**Salida esperada:**

```
sysadmins:x:3001:
developers:x:3002:
operators:x:3003:
```

3. Documenta la estructura de grupos creada:

```bash
echo "=== Estructura de Grupos Organizacionales ===" | sudo tee /opt/sysreport/grupos_org.txt
echo "Fecha: $(date)" | sudo tee -a /opt/sysreport/grupos_org.txt
getent group sysadmins developers operators | sudo tee -a /opt/sysreport/grupos_org.txt
```

**Verificación:**

```bash
getent group sysadmins developers operators | wc -l
# Debe mostrar: 3
```

---

### Paso 3: Crear usuarios del grupo sysadmins

**Objetivo:** Crear los usuarios administrativos y asignarlos al grupo `sysadmins`.

**Instrucciones:**

1. Agrega el usuario existente `sysadmin` al grupo `sysadmins` como grupo secundario:

```bash
sudo usermod -aG sysadmins sysadmin
```

2. Crea el usuario `admin_corp` con los parámetros corporativos:

```bash
sudo useradd \
  -m \
  -d /home/admin_corp \
  -s /bin/bash \
  -g sysadmins \
  -G sudo \
  -c "Administrador Corporativo - Infraestructura" \
  admin_corp
```

3. Establece la contraseña para `admin_corp`:

```bash
echo "admin_corp:Admin@Corp2024!" | sudo chpasswd
```

4. Configura la política de expiración para `admin_corp`:

```bash
sudo chage -M 90 -W 14 -m 7 admin_corp
```

5. Verifica la configuración:

```bash
id admin_corp
sudo chage -l admin_corp
```

**Salida esperada:**

```
uid=1001(admin_corp) gid=3001(sysadmins) groups=3001(sysadmins),27(sudo)
```

```
Last password change                : [fecha actual]
Password expires                    : [fecha + 90 días]
Password inactive                   : never
Account expires                     : never
Minimum number of days between password change : 7
Maximum number of days between password change : 90
Number of days of warning before password expires : 14
```

**Verificación:**

```bash
grep "^admin_corp:" /etc/passwd
# admin_corp:x:1001:3001:Administrador Corporativo - Infraestructura:/home/admin_corp:/bin/bash
```

---

### Paso 4: Crear usuarios del grupo developers

**Objetivo:** Crear las tres cuentas de desarrolladores con configuración estandarizada.

**Instrucciones:**

1. Crea los usuarios dev01, dev02 y dev03 usando un bucle:

```bash
for i in 01 02 03; do
  sudo useradd \
    -m \
    -d /home/dev${i} \
    -s /bin/bash \
    -g developers \
    -c "Desarrollador ${i} - Equipo de Desarrollo" \
    dev${i}
  echo "dev${i}:Dev${i}@2024!" | sudo chpasswd
  sudo chage -M 60 -W 7 -m 5 dev${i}
  echo "Usuario dev${i} creado exitosamente"
done
```

2. Verifica la creación de los tres usuarios:

```bash
grep "^dev0[1-3]:" /etc/passwd
```

**Salida esperada:**

```
dev01:x:1002:3002:Desarrollador 01 - Equipo de Desarrollo:/home/dev01:/bin/bash
dev02:x:1003:3002:Desarrollador 02 - Equipo de Desarrollo:/home/dev02:/bin/bash
dev03:x:1004:3002:Desarrollador 03 - Equipo de Desarrollo:/home/dev03:/bin/bash
```

3. Verifica los directorios home creados:

```bash
ls -la /home/ | grep dev
```

**Salida esperada:**

```
drwxr-x--- 2 dev01 developers 4096 [fecha] dev01
drwxr-x--- 2 dev02 developers 4096 [fecha] dev02
drwxr-x--- 2 dev03 developers 4096 [fecha] dev03
```

**Verificación:**

```bash
for i in 01 02 03; do
  echo "--- dev${i} ---"
  id dev${i}
done
```

---

### Paso 5: Crear usuarios del grupo operators

**Objetivo:** Crear las cuentas de operadores con permisos limitados.

**Instrucciones:**

1. Crea los usuarios op01 y op02:

```bash
for i in 01 02; do
  sudo useradd \
    -m \
    -d /home/op${i} \
    -s /bin/bash \
    -g operators \
    -c "Operador ${i} - Equipo de Operaciones" \
    op${i}
  echo "op${i}:Op${i}@2024!" | sudo chpasswd
  sudo chage -M 45 -W 7 -m 3 op${i}
  echo "Usuario op${i} creado exitosamente"
done
```

2. Verifica la creación:

```bash
grep "^op0[1-2]:" /etc/passwd
```

**Salida esperada:**

```
op01:x:1005:3003:Operador 01 - Equipo de Operaciones:/home/op01:/bin/bash
op02:x:1006:3003:Operador 02 - Equipo de Operaciones:/home/op02:/bin/bash
```

3. Verifica las membresías de grupo completas:

```bash
getent group sysadmins developers operators
```

**Salida esperada:**

```
sysadmins:x:3001:sysadmin
developers:x:3002:
operators:x:3003:
```

> **Nota:** Los usuarios dev01-dev03 y op01-op02 no aparecen listados en la línea del grupo porque `sysadmins`/`developers`/`operators` es su grupo primario (definido en `/etc/passwd`), no secundario.

**Verificación:**

```bash
# Confirmar que todos los usuarios tienen su grupo primario correcto
id dev01 | grep "gid=3002(developers)" && echo "OK"
id op01 | grep "gid=3003(operators)" && echo "OK"
```

---

### Paso 6: Analizar la estructura de archivos del sistema

**Objetivo:** Examinar manualmente los archivos `/etc/passwd`, `/etc/shadow` y `/etc/group` para comprender su estructura interna tras las creaciones realizadas.

**Instrucciones:**

1. Examina las entradas relevantes de `/etc/passwd`:

```bash
echo "=== /etc/passwd - Usuarios creados ==="
grep -E "^(admin_corp|dev0[1-3]|op0[1-2]):" /etc/passwd
```

2. Examina las entradas correspondientes en `/etc/shadow`:

```bash
echo "=== /etc/shadow - Políticas de contraseña ==="
sudo grep -E "^(admin_corp|dev0[1-3]|op0[1-2]):" /etc/shadow
```

**Salida esperada (formato):**

```
admin_corp:$6$...:19XXX:7:90:14:::
dev01:$6$...:19XXX:5:60:7:::
dev02:$6$...:19XXX:5:60:7:::
dev03:$6$...:19XXX:5:60:7:::
op01:$6$...:19XXX:3:45:7:::
op02:$6$...:19XXX:3:45:7:::
```

3. Documenta la estructura para referencia:

```bash
echo "=== /etc/group - Grupos organizacionales ==="
grep -E "^(sysadmins|developers|operators):" /etc/group

echo ""
echo "=== Resumen de UIDs asignados ==="
awk -F: '/^(admin_corp|dev0[1-3]|op0[1-2]):/ {printf "%-12s UID=%-6s GID=%-6s Shell=%s\n", $1, $3, $4, $7}' /etc/passwd
```

**Verificación:**

```bash
# Verificar que no hay UIDs duplicados
awk -F: '{print $3}' /etc/passwd | sort -n | uniq -d | wc -l
# Debe mostrar: 0
```

---

### Paso 7: Configurar política de complejidad de contraseñas con pam_pwquality

**Objetivo:** Configurar PAM para exigir contraseñas de mínimo 12 caracteres con requisitos de complejidad.

**Instrucciones:**

1. Realiza un respaldo del archivo de configuración PAM de contraseñas:

```bash
sudo cp /etc/pam.d/common-password /etc/pam.d/common-password.bak
```

2. Edita la configuración de `pam_pwquality`:

```bash
sudo nano /etc/security/pwquality.conf
```

3. Agrega o modifica las siguientes directivas al final del archivo:

```ini
# Política de contraseñas corporativa - Lab 09
minlen = 12
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
minclass = 3
maxrepeat = 3
maxclassrepeat = 4
gecoscheck = 1
dictcheck = 1
```

> **Explicación de parámetros:**
> - `minlen = 12`: longitud mínima de 12 caracteres
> - `dcredit = -1`: al menos 1 dígito
> - `ucredit = -1`: al menos 1 mayúscula
> - `lcredit = -1`: al menos 1 minúscula
> - `ocredit = -1`: al menos 1 carácter especial
> - `minclass = 3`: mínimo 3 clases de caracteres distintas
> - `maxrepeat = 3`: máximo 3 caracteres consecutivos iguales
> - `gecoscheck = 1`: no permitir partes del nombre del usuario

4. Verifica que la línea de `pam_pwquality` existe en `/etc/pam.d/common-password`:

```bash
grep "pam_pwquality" /etc/pam.d/common-password
```

**Salida esperada:**

```
password    requisite     pam_pwquality.so retry=3
```

Si la línea no incluye `retry=3`, modifícala:

```bash
sudo sed -i 's/pam_pwquality.so.*/pam_pwquality.so retry=3/' /etc/pam.d/common-password
```

**Verificación:**

```bash
# Probar que la política se aplica (esto debe fallar con contraseña débil)
echo "dev01:abc" | sudo chpasswd 2>&1 | grep -i "password" || echo "Política activa - contraseña rechazada"

# Verificar la configuración
sudo grep -v "^#\|^$" /etc/security/pwquality.conf
```

---

### Paso 8: Configurar bloqueo por intentos fallidos con pam_faillock

**Objetivo:** Configurar el bloqueo automático de cuentas tras 3 intentos fallidos de autenticación.

**Instrucciones:**

1. Respalda los archivos de configuración PAM relevantes:

```bash
sudo cp /etc/pam.d/common-auth /etc/pam.d/common-auth.bak
sudo cp /etc/pam.d/common-account /etc/pam.d/common-account.bak
```

2. Crea el archivo de configuración de faillock:

```bash
sudo nano /etc/security/faillock.conf
```

3. Agrega el siguiente contenido:

```ini
# Configuración de bloqueo por intentos fallidos - Lab 09
# Bloquear después de 3 intentos fallidos
deny = 3
# Tiempo de bloqueo: 600 segundos (10 minutos)
unlock_time = 600
# Ventana de tiempo para contar intentos: 900 segundos (15 minutos)
fail_interval = 900
# No bloquear al usuario root
even_deny_root = false
# Directorio para almacenar registros de fallos
dir = /var/run/faillock
```

4. Edita `/etc/pam.d/common-auth` para agregar `pam_faillock`:

```bash
sudo nano /etc/pam.d/common-auth
```

Agrega las siguientes líneas **antes** de la línea `pam_unix.so`:

```
auth    required    pam_faillock.so preauth
```

Y **después** de la línea `pam_unix.so`, agrega:

```
auth    [default=die] pam_faillock.so authfail
```

El archivo debe quedar similar a:

```
# /etc/pam.d/common-auth
auth    required    pam_faillock.so preauth
auth    [success=1 default=ignore]  pam_unix.so nullok
auth    [default=die] pam_faillock.so authfail
auth    requisite   pam_deny.so
auth    required    pam_permit.so
```

5. Edita `/etc/pam.d/common-account` para agregar el reset de faillock:

```bash
sudo nano /etc/pam.d/common-account
```

Agrega al inicio del archivo (después de los comentarios):

```
account required    pam_faillock.so
```

6. Crea el directorio de faillock si no existe:

```bash
sudo mkdir -p /var/run/faillock
```

**Verificación:**

```bash
# Verificar que la configuración es sintácticamente correcta
sudo grep "pam_faillock" /etc/pam.d/common-auth
sudo grep "pam_faillock" /etc/pam.d/common-account
cat /etc/security/faillock.conf | grep -v "^#\|^$"
```

**Salida esperada:**

```
auth    required    pam_faillock.so preauth
auth    [default=die] pam_faillock.so authfail
account required    pam_faillock.so
deny = 3
unlock_time = 600
fail_interval = 900
even_deny_root = false
dir = /var/run/faillock
```

---

### Paso 9: Configurar historial de contraseñas con pam_pwhistory

**Objetivo:** Impedir que los usuarios reutilicen sus últimas 5 contraseñas.

**Instrucciones:**

1. Edita `/etc/pam.d/common-password`:

```bash
sudo nano /etc/pam.d/common-password
```

2. Agrega la siguiente línea **antes** de la línea que contiene `pam_unix.so`:

```
password    required    pam_pwhistory.so remember=5 use_authtok enforce_for_root
```

3. Crea el directorio para almacenar el historial si no existe:

```bash
sudo mkdir -p /etc/security/opasswd
sudo touch /etc/security/opasswd
sudo chmod 600 /etc/security/opasswd
```

4. Verifica la configuración final de `/etc/pam.d/common-password`:

```bash
grep -v "^#\|^$" /etc/pam.d/common-password
```

**Salida esperada:**

```
password    requisite     pam_pwquality.so retry=3
password    required      pam_pwhistory.so remember=5 use_authtok enforce_for_root
password    [success=1 default=ignore]  pam_unix.so obscure use_authtok try_first_pass yescrypt
password    requisite     pam_deny.so
password    required      pam_permit.so
```

**Verificación:**

```bash
# Confirmar que el archivo opasswd tiene permisos correctos
ls -la /etc/security/opasswd
# -rw------- 1 root root 0 [fecha] /etc/security/opasswd
```

---

### Paso 10: Forzar cambio de contraseña en primer inicio de sesión

**Objetivo:** Configurar todos los usuarios nuevos para que deban cambiar su contraseña en el primer login.

**Instrucciones:**

1. Aplica la expiración inmediata de contraseña a todos los usuarios creados:

```bash
for user in admin_corp dev01 dev02 dev03 op01 op02; do
  sudo chage -d 0 ${user}
  echo "Forzado cambio de contraseña para: ${user}"
done
```

2. Verifica el estado de las contraseñas:

```bash
for user in admin_corp dev01 dev02 dev03 op01 op02; do
  echo -n "${user}: "
  sudo chage -l ${user} | grep "Last password change"
done
```

**Salida esperada:**

```
admin_corp: Last password change                    : password must be changed
dev01: Last password change                         : password must be changed
dev02: Last password change                         : password must be changed
dev03: Last password change                         : password must be changed
op01: Last password change                          : password must be changed
op02: Last password change                          : password must be changed
```

**Verificación:**

```bash
sudo grep -E "^(admin_corp|dev0[1-3]|op0[1-2]):" /etc/shadow | awk -F: '{print $1, "último_cambio="$3}'
# Todos deben mostrar último_cambio=0
```

---

### Paso 11: Configurar delegación de privilegios con sudo

**Objetivo:** Configurar reglas sudo específicas para cada grupo organizacional.

**Instrucciones:**

1. Crea un archivo de configuración sudo para el grupo sysadmins:

```bash
sudo visudo -f /etc/sudoers.d/sysadmins
```

Contenido:

```
# Privilegios para el grupo sysadmins - acceso completo
%sysadmins ALL=(ALL:ALL) ALL
```

2. Crea un archivo para el grupo developers:

```bash
sudo visudo -f /etc/sudoers.d/developers
```

Contenido:

```
# Privilegios para el grupo developers - acceso limitado
%developers ALL=(ALL) /usr/bin/systemctl status *, /usr/bin/journalctl, /usr/bin/docker *
%developers ALL=(ALL) NOPASSWD: /opt/scripts/system_inventory.sh
```

3. Crea un archivo para el grupo operators:

```bash
sudo visudo -f /etc/sudoers.d/operators
```

Contenido:

```
# Privilegios para el grupo operators - operaciones básicas
%operators ALL=(ALL) /usr/bin/systemctl restart *, /usr/bin/systemctl status *
%operators ALL=(ALL) /opt/scripts/log_analyzer.sh, /opt/scripts/storage_report.sh
%operators ALL=(ALL) NOPASSWD: /usr/bin/tail -f /var/log/*
```

4. Establece los permisos correctos:

```bash
sudo chmod 440 /etc/sudoers.d/sysadmins
sudo chmod 440 /etc/sudoers.d/developers
sudo chmod 440 /etc/sudoers.d/operators
```

5. Valida la sintaxis de sudoers:

```bash
sudo visudo -c
```

**Salida esperada:**

```
/etc/sudoers: parsed OK
/etc/sudoers.d/developers: parsed OK
/etc/sudoers.d/operators: parsed OK
/etc/sudoers.d/sysadmins: parsed OK
```

**Verificación:**

```bash
# Verificar archivos creados
ls -la /etc/sudoers.d/ | grep -E "(sysadmins|developers|operators)"
```

---

### Paso 12: Programar scripts con cron

**Objetivo:** Configurar la ejecución automática de los scripts del Lab 07 según el calendario establecido.

**Instrucciones:**

1. Verifica que los scripts existen y son ejecutables:

```bash
ls -la /opt/scripts/{system_inventory.sh,log_analyzer.sh,storage_report.sh}
```

Si alguno no tiene permisos de ejecución:

```bash
sudo chmod +x /opt/scripts/{system_inventory.sh,log_analyzer.sh,storage_report.sh}
```

2. Edita el crontab del usuario root para las tareas del sistema:

```bash
sudo crontab -e
```

3. Agrega las siguientes entradas al crontab:

```cron
# ============================================
# Tareas programadas - Lab 09
# ============================================

# system_inventory.sh - Ejecución diaria a las 6:00 AM
0 6 * * * /opt/scripts/system_inventory.sh >> /var/log/admin_scripts.log 2>&1

# log_analyzer.sh - Ejecución cada hora (minuto 15)
15 * * * * /opt/scripts/log_analyzer.sh >> /var/log/admin_scripts.log 2>&1

# storage_report.sh - Ejecución cada lunes a las 7:00 AM
0 7 * * 1 /opt/scripts/storage_report.sh >> /var/log/admin_scripts.log 2>&1
```

4. Crea el archivo de log si no existe y establece permisos:

```bash
sudo touch /var/log/admin_scripts.log
sudo chmod 640 /var/log/admin_scripts.log
sudo chown root:sysadmins /var/log/admin_scripts.log
```

5. Verifica que el crontab fue guardado correctamente:

```bash
sudo crontab -l
```

**Salida esperada:**

```
# system_inventory.sh - Ejecución diaria a las 6:00 AM
0 6 * * * /opt/scripts/system_inventory.sh >> /var/log/admin_scripts.log 2>&1
# log_analyzer.sh - Ejecución cada hora (minuto 15)
15 * * * * /opt/scripts/log_analyzer.sh >> /var/log/admin_scripts.log 2>&1
# storage_report.sh - Ejecución cada lunes a las 7:00 AM
0 7 * * 1 /opt/scripts/storage_report.sh >> /var/log/admin_scripts.log 2>&1
```

**Verificación:**

```bash
# Ejecutar manualmente uno de los scripts para confirmar que funciona
sudo /opt/scripts/system_inventory.sh >> /var/log/admin_scripts.log 2>&1
echo "Exit code: $?"
tail -5 /var/log/admin_scripts.log
```

---

### Paso 13: Programar tarea única con at

**Objetivo:** Programar un respaldo inmediato (en 5 minutos) del directorio `/opt/scripts` usando el comando `at`.

**Instrucciones:**

1. Verifica que el directorio de backups existe:

```bash
sudo mkdir -p /opt/backups
```

2. Programa la tarea de respaldo con `at`:

```bash
echo "tar -czf /opt/backups/scripts_backup_\$(date +%Y%m%d_%H%M%S).tar.gz /opt/scripts/ && echo 'Backup completado: \$(date)' >> /var/log/admin_scripts.log" | sudo at now + 5 minutes
```

3. Verifica que la tarea fue programada:

```bash
sudo atq
```

**Salida esperada:**

```
1   [fecha y hora]  a root
```

4. Para verificar el contenido de la tarea programada:

```bash
sudo at -c $(sudo atq | awk '{print $1}' | head -1) | tail -5
```

5. Programa una segunda tarea de backup para las 23:00 de hoy (como ejemplo de programación por hora):

```bash
echo "tar -czf /opt/backups/scripts_nightly_\$(date +%Y%m%d).tar.gz /opt/scripts/" | sudo at 23:00
```

6. Lista todas las tareas programadas:

```bash
sudo atq
```

**Verificación:**

```bash
# Esperar 5 minutos y verificar que el backup se creó
# (o verificar después de completar los pasos siguientes)
sleep 10 && ls -la /opt/backups/ 2>/dev/null || echo "Backup pendiente (esperar 5 min)"
```

---

### Paso 14: Configurar localización del sistema (locale)

**Objetivo:** Configurar el sistema para utilizar el locale `es_MX.UTF-8` como idioma principal.

**Instrucciones:**

1. Verifica la configuración actual de locale:

```bash
locale
```

2. Genera el locale `es_MX.UTF-8`:

```bash
sudo locale-gen es_MX.UTF-8
```

**Salida esperada:**

```
Generating locales (this might take a while)...
  es_MX.UTF-8... done
Generation complete.
```

3. Configura el locale predeterminado del sistema:

```bash
sudo update-locale LANG=es_MX.UTF-8 LC_ALL=es_MX.UTF-8
```

4. Verifica que el archivo de configuración fue actualizado:

```bash
cat /etc/default/locale
```

**Salida esperada:**

```
LANG=es_MX.UTF-8
LC_ALL=es_MX.UTF-8
```

5. Aplica el cambio en la sesión actual para verificar:

```bash
export LANG=es_MX.UTF-8
export LC_ALL=es_MX.UTF-8
locale
```

**Verificación:**

```bash
locale | grep "es_MX"
# Todas las variables deben mostrar es_MX.UTF-8
```

---

### Paso 15: Configurar zona horaria del sistema

**Objetivo:** Establecer la zona horaria `America/Mexico_City` utilizando `timedatectl`.

**Instrucciones:**

1. Verifica la zona horaria actual:

```bash
timedatectl
```

2. Establece la zona horaria de México:

```bash
sudo timedatectl set-timezone America/Mexico_City
```

3. Verifica el cambio:

```bash
timedatectl
```

**Salida esperada:**

```
               Local time: [día] [fecha] [hora] CST
           Universal time: [día] [fecha] [hora] UTC
                 RTC time: [día] [fecha] [hora]
                Time zone: America/Mexico_City (CST, -0600)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

4. Confirma con el comando `date`:

```bash
date
```

**Salida esperada (formato con locale español):**

```
[día de la semana] [día] de [mes] de [año], [hora] CST
```

5. Verifica que el enlace simbólico de localtime es correcto:

```bash
ls -la /etc/localtime
```

**Salida esperada:**

```
lrwxrwxrwx 1 root root 39 [fecha] /etc/localtime -> ../usr/share/zoneinfo/America/Mexico_City
```

**Verificación:**

```bash
timedatectl | grep "Time zone" | grep "America/Mexico_City" && echo "Zona horaria OK"
```

---

### Paso 16: Generar reporte final de configuración

**Objetivo:** Documentar toda la configuración realizada en un reporte consolidado.

**Instrucciones:**

1. Crea el script de reporte final:

```bash
sudo tee /opt/sysreport/lab09_report.sh << 'EOF'
#!/bin/bash
# Reporte de configuración - Lab 09
REPORT="/opt/sysreport/lab09_final_report.txt"

echo "================================================================" > $REPORT
echo "  REPORTE FINAL - LAB 09: Políticas de Acceso y Automatización" >> $REPORT
echo "  Fecha: $(date)" >> $REPORT
echo "  Servidor: $(hostname)" >> $REPORT
echo "================================================================" >> $REPORT

echo -e "\n--- USUARIOS CREADOS ---" >> $REPORT
for user in admin_corp dev01 dev02 dev03 op01 op02; do
  echo "$(id $user)" >> $REPORT
done

echo -e "\n--- GRUPOS ORGANIZACIONALES ---" >> $REPORT
getent group sysadmins developers operators >> $REPORT

echo -e "\n--- POLÍTICAS DE CONTRASEÑA ---" >> $REPORT
grep -v "^#\|^$" /etc/security/pwquality.conf >> $REPORT

echo -e "\n--- CONFIGURACIÓN FAILLOCK ---" >> $REPORT
grep -v "^#\|^$" /etc/security/faillock.conf >> $REPORT

echo -e "\n--- TAREAS CRON (root) ---" >> $REPORT
crontab -l 2>/dev/null | grep -v "^#\|^$" >> $REPORT

echo -e "\n--- TAREAS AT PENDIENTES ---" >> $REPORT
atq >> $REPORT

echo -e "\n--- LOCALIZACIÓN ---" >> $REPORT
locale >> $REPORT

echo -e "\n--- ZONA HORARIA ---" >> $REPORT
timedatectl >> $REPORT

echo -e "\n--- ARCHIVOS SUDOERS PERSONALIZADOS ---" >> $REPORT
ls -la /etc/sudoers.d/ >> $REPORT

echo "================================================================" >> $REPORT
echo "  Reporte generado exitosamente" >> $REPORT
echo "================================================================" >> $REPORT

cat $REPORT
EOF
```

2. Ejecuta el reporte:

```bash
sudo chmod +x /opt/sysreport/lab09_report.sh
sudo /opt/sysreport/lab09_report.sh
```

3. Verifica que el reporte se generó correctamente:

```bash
ls -la /opt/sysreport/lab09_final_report.txt
wc -l /opt/sysreport/lab09_final_report.txt
```

---

## Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que todo el laboratorio se completó correctamente:

### Prueba 1: Estructura de usuarios y grupos

```bash
echo "=== Verificación de usuarios ==="
for user in admin_corp dev01 dev02 dev03 op01 op02; do
  if id "$user" &>/dev/null; then
    echo "[OK] $user existe"
  else
    echo "[FALLO] $user NO existe"
  fi
done

echo -e "\n=== Verificación de grupos ==="
for group in sysadmins developers operators; do
  if getent group "$group" &>/dev/null; then
    echo "[OK] $group existe (GID: $(getent group $group | cut -d: -f3))"
  else
    echo "[FALLO] $group NO existe"
  fi
done
```

### Prueba 2: Políticas PAM

```bash
echo "=== Verificación PAM ==="
grep -q "pam_pwquality" /etc/pam.d/common-password && echo "[OK] pam_pwquality configurado" || echo "[FALLO] pam_pwquality"
grep -q "pam_faillock" /etc/pam.d/common-auth && echo "[OK] pam_faillock configurado" || echo "[FALLO] pam_faillock"
grep -q "pam_pwhistory" /etc/pam.d/common-password && echo "[OK] pam_pwhistory configurado" || echo "[FALLO] pam_pwhistory"
grep -q "minlen = 12" /etc/security/pwquality.conf && echo "[OK] Longitud mínima 12 caracteres" || echo "[FALLO] minlen"
grep -q "deny = 3" /etc/security/faillock.conf && echo "[OK] Bloqueo tras 3 intentos" || echo "[FALLO] deny"
```

### Prueba 3: Tareas programadas

```bash
echo "=== Verificación CRON ==="
sudo crontab -l | grep -q "system_inventory" && echo "[OK] system_inventory.sh programado (6:00 AM diario)" || echo "[FALLO]"
sudo crontab -l | grep -q "log_analyzer" && echo "[OK] log_analyzer.sh programado (cada hora)" || echo "[FALLO]"
sudo crontab -l | grep -q "storage_report" && echo "[OK] storage_report.sh programado (lunes 7:00 AM)" || echo "[FALLO]"

echo -e "\n=== Verificación AT ==="
sudo atq | wc -l | xargs -I{} echo "[INFO] {} tarea(s) pendiente(s) en at"
```

### Prueba 4: Localización y zona horaria

```bash
echo "=== Verificación de Localización ==="
grep -q "es_MX.UTF-8" /etc/default/locale && echo "[OK] Locale es_MX.UTF-8 configurado" || echo "[FALLO]"
timedatectl | grep -q "America/Mexico_City" && echo "[OK] Zona horaria America/Mexico_City" || echo "[FALLO]"
```

### Prueba 5: Configuración sudo

```bash
echo "=== Verificación SUDO ==="
sudo visudo -c 2>&1 | grep -q "parsed OK" && echo "[OK] Sintaxis sudoers válida" || echo "[FALLO] Error en sudoers"
for f in sysadmins developers operators; do
  [ -f "/etc/sudoers.d/$f" ] && echo "[OK] /etc/sudoers.d/$f existe" || echo "[FALLO] $f"
done
```

---

## Solución de Problemas

### Problema 1: El usuario no puede autenticarse después de configurar pam_faillock

**Síntomas:**
- Un usuario legítimo no puede iniciar sesión a pesar de ingresar la contraseña correcta
- El mensaje de error indica "Account locked due to X failed logins"

**Causa:**
El usuario excedió el número de intentos fallidos (3) durante pruebas o por error tipográfico, y la cuenta fue bloqueada por `pam_faillock`.

**Solución:**

```bash
# Ver el estado de bloqueo del usuario
sudo faillock --user dev01

# Resetear el contador de intentos fallidos
sudo faillock --user dev01 --reset

# Verificar que la cuenta fue desbloqueada
sudo faillock --user dev01
# Debe mostrar la lista vacía o sin entradas recientes
```

Si `faillock` no está disponible como comando, revisa el directorio de registros:

```bash
sudo ls -la /var/run/faillock/
sudo rm -f /var/run/faillock/dev01
```

---

### Problema 2: Las tareas cron no se ejecutan en el horario programado

**Síntomas:**
- El archivo `/var/log/admin_scripts.log` no se actualiza en los horarios configurados
- Los scripts funcionan correctamente cuando se ejecutan manualmente

**Causa:**
Los scripts utilizan rutas relativas o variables de entorno que no están disponibles en el entorno limitado de cron (cron ejecuta con un PATH mínimo: `/usr/bin:/bin`).

**Solución:**

1. Agrega la variable PATH al inicio del crontab:

```bash
sudo crontab -e
```

Agrega en la primera línea:

```cron
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
SHELL=/bin/bash
MAILTO=""
```

2. Verifica que los scripts usan rutas absolutas internamente:

```bash
head -5 /opt/scripts/system_inventory.sh
# Debe usar #!/bin/bash y rutas absolutas como /usr/bin/df, /usr/bin/free, etc.
```

3. Verifica los logs del servicio cron para diagnosticar:

```bash
sudo grep CRON /var/log/syslog | tail -20
```

4. Confirma que los permisos del script permiten ejecución por root:

```bash
ls -la /opt/scripts/*.sh
# Deben tener permiso de ejecución (x)
sudo chmod +x /opt/scripts/*.sh
```

---

## Limpieza

> **Nota:** Este laboratorio crea infraestructura que será utilizada en laboratorios posteriores. **NO elimines los usuarios, grupos ni configuraciones PAM.** Solo realiza limpieza de archivos temporales.

```bash
# Eliminar tareas at que ya no sean necesarias (solo las de prueba)
# Listar tareas pendientes
sudo atq

# Si hay tareas de prueba que deseas eliminar (reemplaza N con el número de tarea):
# sudo atrm N

# Verificar que los backups de configuración PAM existen por seguridad
ls -la /etc/pam.d/*.bak

# Limpiar archivos temporales de la sesión
rm -f /tmp/lab09_*
```

---

## Resumen

En este laboratorio se implementaron los siguientes componentes de administración del sistema:

| Componente | Configuración realizada |
|------------|------------------------|
| **Usuarios** | 6 cuentas creadas (admin_corp, dev01-03, op01-02) con políticas de expiración diferenciadas |
| **Grupos** | 3 grupos organizacionales (sysadmins GID:3001, developers GID:3002, operators GID:3003) |
| **PAM - pwquality** | Contraseñas mínimo 12 caracteres, 3 clases de caracteres, sin repeticiones excesivas |
| **PAM - faillock** | Bloqueo automático tras 3 intentos fallidos, desbloqueo a los 10 minutos |
| **PAM - pwhistory** | Historial de últimas 5 contraseñas, reutilización prohibida |
| **Cron** | 3 tareas programadas: inventario (diario 6AM), logs (cada hora), almacenamiento (lunes 7AM) |
| **At** | Backup programado de /opt/scripts a /opt/backups |
| **Locale** | es_MX.UTF-8 como idioma del sistema |
| **Timezone** | America/Mexico_City (CST/CDT) |
| **Sudo** | Delegación de privilegios diferenciada por grupo organizacional |

### Recursos adicionales

- `man 5 passwd` — Formato del archivo /etc/passwd
- `man 5 shadow` — Formato del archivo /etc/shadow
- `man pam_faillock` — Documentación del módulo de bloqueo
- `man pam_pwquality` — Documentación de políticas de contraseña
- `man 5 crontab` — Formato de archivos crontab
- `man at` — Programación de tareas únicas
- Guía de seguridad Ubuntu: https://ubuntu.com/security/certifications/docs

---
