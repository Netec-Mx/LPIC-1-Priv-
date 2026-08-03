# Implementar controles de acceso, cifrado y endurecimiento del sistema para reducir la superficie de ataque y proteger los recursos críticos del servidor

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 73 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio implementarás una estrategia integral de seguridad sobre la VM `srv-linux` (Ubuntu 22.04.4 LTS). Partiendo de la infraestructura SSH configurada en el Laboratorio 11, aplicarás permisos POSIX avanzados con ACLs para control de acceso basado en roles, configurarás políticas de sudo con mínimo privilegio, endurecerás el sistema con UFW, fail2ban y auditd, cifrarás archivos con GPG y dispositivos de bloque con LUKS, y finalizarás con una auditoría de seguridad usando Lynis.

## Objetivos de Aprendizaje

- [ ] Implementar permisos POSIX y ACLs en `/srv/proyecto/` para controlar acceso granular de desarrolladores, auditores y administradores
- [ ] Configurar sudo mediante archivos en `/etc/sudoers.d/` aplicando el principio de mínimo privilegio
- [ ] Endurecer el sistema con UFW (puerto 2222), fail2ban y reglas de auditoría con auditd
- [ ] Cifrar archivos sensibles con GPG (RSA-4096 y AES-256) y firmar digitalmente documentos
- [ ] Crear y gestionar un volumen cifrado LUKS2 en `/dev/sdb` montado en `/mnt/secure_vault/`

## Prerrequisitos

### Conocimientos Previos
- Laboratorio 11 completado con SSH funcional en puerto 2222
- Permisos Unix básicos (chmod, chown, grupos)
- Conceptos de cifrado simétrico y asimétrico
- Familiaridad con gestión de servicios systemd

### Acceso Requerido
- VM `srv-linux` (Ubuntu 22.04.4 LTS) con IP 192.168.100.10
- VM `cli-linux` para pruebas de conectividad SSH
- Usuario `labuser` con sudo configurado y llaves SSH distribuidas
- Archivo `/home/labuser/lab11/testfile_10MB.bin` presente en `srv-linux`
- Disco virtual secundario de 2 GB (`/dev/sdb`) añadido a `srv-linux`

## Entorno del Laboratorio

### Máquinas Virtuales

| VM | Sistema Operativo | IP | Rol |
|----|------------------|-----|-----|
| srv-linux | Ubuntu 22.04.4 LTS | 192.168.100.10 | Servidor principal |
| cli-linux | Ubuntu 22.04.4 LTS | 192.168.100.20 | Cliente SSH |

### Software Requerido

| Paquete | Versión | Propósito |
|---------|---------|-----------|
| acl | 2.3.1 | ACLs POSIX |
| sudo | 1.9.9p2 | Delegación de privilegios |
| ufw | 0.36.1 | Firewall simplificado |
| fail2ban | 0.11.2 | Protección contra fuerza bruta |
| auditd | 3.0.7 | Auditoría del kernel |
| gnupg | 2.2.27 | Cifrado GPG |
| cryptsetup | 2.4.3 | LUKS2 |
| lynis | 3.0.8 | Auditoría de seguridad |

### Preparación Inicial

Ejecutar en `srv-linux` como `labuser`:

```bash
# Verificar disco secundario disponible
lsblk | grep sdb

# Instalar todos los paquetes necesarios
sudo apt update
sudo apt install -y acl ufw fail2ban auditd audispd-plugins gnupg2 cryptsetup lynis
```

---

## Procedimiento Paso a Paso

### Paso 1: Crear estructura de directorios y grupos para control de acceso basado en roles

**Objetivo:** Establecer la estructura de directorios `/srv/proyecto/` con grupos de trabajo diferenciados (desarrolladores, auditores, administradores) y usuarios asignados a cada rol.

**Instrucciones:**

1. Crear los grupos de trabajo:

```bash
sudo groupadd desarrolladores
sudo groupadd auditores
sudo groupadd administradores
```

2. Crear usuarios para cada rol:

```bash
# Desarrolladores
sudo useradd -m -s /bin/bash -G desarrolladores dev01
sudo useradd -m -s /bin/bash -G desarrolladores dev02

# Auditor
sudo useradd -m -s /bin/bash -G auditores auditor01

# Administrador adicional
sudo useradd -m -s /bin/bash -G administradores admin01

# Asignar contraseñas
echo "dev01:Dev@2024!" | sudo chpasswd
echo "dev02:Dev@2024!" | sudo chpasswd
echo "auditor01:Audit@2024!" | sudo chpasswd
echo "admin01:Admin@2024!" | sudo chpasswd
```

3. Crear la estructura de directorios del proyecto:

```bash
sudo mkdir -p /srv/proyecto/{codigo,documentos,logs,config,backups}
```

4. Asignar propietario root y grupo administradores al directorio raíz del proyecto:

```bash
sudo chown -R root:administradores /srv/proyecto
```

5. Configurar permisos base con SGID para herencia de grupo:

```bash
# Directorio raíz: administradores tienen acceso total
sudo chmod 2770 /srv/proyecto

# Código: desarrolladores trabajan aquí
sudo chown root:desarrolladores /srv/proyecto/codigo
sudo chmod 2770 /srv/proyecto/codigo

# Documentos: lectura para todos los roles, escritura para desarrolladores
sudo chown root:desarrolladores /srv/proyecto/documentos
sudo chmod 2775 /srv/proyecto/documentos

# Logs: solo lectura para auditores
sudo chown root:administradores /srv/proyecto/logs
sudo chmod 2750 /srv/proyecto/logs

# Config: solo administradores
sudo chown root:administradores /srv/proyecto/config
sudo chmod 2770 /srv/proyecto/config

# Backups: sticky bit + SGID para protección
sudo chown root:administradores /srv/proyecto/backups
sudo chmod 3770 /srv/proyecto/backups
```

**Salida esperada:**

```
$ ls -la /srv/proyecto/
total 28
drwxrws---  7 root administradores 4096 jun 15 10:00 .
drwxr-xr-x  3 root root           4096 jun 15 10:00 ..
drwxrws--T  2 root administradores 4096 jun 15 10:00 backups
drwxrws---  2 root desarrolladores 4096 jun 15 10:00 codigo
drwxrws---  2 root administradores 4096 jun 15 10:00 config
drwxrwsr-x  2 root desarrolladores 4096 jun 15 10:00 documentos
drwxr-s---  2 root administradores 4096 jun 15 10:00 logs
```

**Verificación:**

```bash
# Verificar SGID activo (la 's' en posición de grupo)
stat -c "%a %U:%G %n" /srv/proyecto/codigo
# Esperado: 2770 root:desarrolladores /srv/proyecto/codigo
```

---

### Paso 2: Implementar ACLs para acceso granular entre roles

**Objetivo:** Usar ACLs POSIX para otorgar permisos cruzados entre roles sin modificar la propiedad de los directorios. Los auditores necesitan lectura en logs y documentos; los desarrolladores necesitan lectura en config.

**Instrucciones:**

1. Otorgar acceso de lectura y traversal al grupo `auditores` sobre `/srv/proyecto/logs`:

```bash
# Primero dar acceso al directorio padre para que auditores puedan navegar
sudo setfacl -m g:auditores:rx /srv/proyecto

# ACL en logs: auditores pueden leer
sudo setfacl -m g:auditores:rx /srv/proyecto/logs

# ACL por defecto para archivos futuros en logs
sudo setfacl -d -m g:auditores:r /srv/proyecto/logs
```

2. Otorgar acceso de lectura a auditores sobre documentos:

```bash
sudo setfacl -m g:auditores:rx /srv/proyecto/documentos
sudo setfacl -d -m g:auditores:r /srv/proyecto/documentos
```

3. Otorgar lectura a desarrolladores sobre config (solo lectura, sin escritura):

```bash
sudo setfacl -m g:desarrolladores:rx /srv/proyecto/config
sudo setfacl -d -m g:desarrolladores:r /srv/proyecto/config
```

4. Otorgar acceso completo a `labuser` sobre toda la estructura (administrador del lab):

```bash
sudo setfacl -R -m u:labuser:rwx /srv/proyecto
sudo setfacl -R -d -m u:labuser:rwx /srv/proyecto
```

5. Verificar las ACLs configuradas:

```bash
getfacl /srv/proyecto/logs
getfacl /srv/proyecto/config
```

**Salida esperada:**

```
$ getfacl /srv/proyecto/logs
# file: srv/proyecto/logs
# owner: root
# group: administradores
# flags: -s-
user::rwx
user:labuser:rwx
group::r-x
group:auditores:r-x
mask::rwx
other::---
default:user::rwx
default:user:labuser:rwx
default:group::r-x
default:group:auditores:r--
default:mask::rwx
default:other::---
```

**Verificación:**

```bash
# Probar acceso como auditor01
sudo -u auditor01 ls /srv/proyecto/logs
# Debe funcionar (salida vacía o listar archivos)

sudo -u auditor01 touch /srv/proyecto/logs/test.txt
# Debe fallar: Permission denied

# Probar acceso como dev01 a config
sudo -u dev01 ls /srv/proyecto/config
# Debe funcionar

sudo -u dev01 touch /srv/proyecto/config/test.txt
# Debe fallar: Permission denied

# Verificar indicador '+' en ls
ls -la /srv/proyecto/ | grep "+"
```

---

### Paso 3: Configurar sudo con políticas de mínimo privilegio

**Objetivo:** Crear archivos de configuración en `/etc/sudoers.d/` que permitan a cada rol ejecutar únicamente los comandos necesarios para su función, sin acceso root completo.

**Instrucciones:**

1. Crear política para desarrolladores (pueden reiniciar servicios de aplicación y ver logs):

```bash
sudo visudo -f /etc/sudoers.d/desarrolladores
```

Contenido del archivo:

```
# Política sudo para grupo desarrolladores
# Pueden reiniciar servicios de aplicación y leer logs del sistema
%desarrolladores ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart apache2, \
                                      /usr/bin/systemctl status apache2, \
                                      /usr/bin/journalctl -u apache2*, \
                                      /usr/bin/tail -f /var/log/syslog, \
                                      /usr/bin/cat /srv/proyecto/logs/*
```

2. Crear política para auditores (solo lectura de logs y estado del sistema):

```bash
sudo visudo -f /etc/sudoers.d/auditores
```

Contenido del archivo:

```
# Política sudo para grupo auditores
# Solo pueden consultar estado del sistema y leer logs
%auditores ALL=(ALL) NOPASSWD: /usr/bin/journalctl --no-edit, \
                                /usr/bin/systemctl status *, \
                                /usr/sbin/auditctl -l, \
                                /usr/sbin/ausearch *, \
                                /usr/bin/last, \
                                /usr/bin/lastlog
```

3. Crear política para administradores (gestión completa de servicios y usuarios):

```bash
sudo visudo -f /etc/sudoers.d/administradores
```

Contenido del archivo:

```
# Política sudo para grupo administradores
# Gestión de servicios, usuarios y paquetes
%administradores ALL=(ALL) NOPASSWD: /usr/bin/systemctl *, \
                                      /usr/sbin/useradd *, \
                                      /usr/sbin/usermod *, \
                                      /usr/bin/apt update, \
                                      /usr/bin/apt install *, \
                                      /usr/sbin/ufw *
```

4. Establecer permisos correctos en los archivos sudoers:

```bash
sudo chmod 440 /etc/sudoers.d/desarrolladores
sudo chmod 440 /etc/sudoers.d/auditores
sudo chmod 440 /etc/sudoers.d/administradores
```

5. Validar la sintaxis de sudo:

```bash
sudo visudo -c
```

**Salida esperada:**

```
$ sudo visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/administradores: parsed OK
/etc/sudoers.d/auditores: parsed OK
/etc/sudoers.d/desarrolladores: parsed OK
```

**Verificación:**

```bash
# Probar que dev01 puede ver estado de servicios
sudo -u dev01 sudo systemctl status apache2 2>/dev/null || echo "Apache no instalado - comportamiento esperado"

# Probar que auditor01 puede ejecutar last
sudo -u auditor01 sudo last -5

# Probar que dev01 NO puede ejecutar comandos de administrador
sudo -u dev01 sudo useradd testuser 2>&1 | grep -i "not allowed\|sorry"
```

---

### Paso 4: Endurecer SSH y configurar UFW

**Objetivo:** Reforzar la configuración SSH del Laboratorio 11 con parámetros adicionales de seguridad y activar UFW con reglas restrictivas.

**Instrucciones:**

1. Aplicar hardening adicional a SSH (el puerto 2222, sin root y sin password ya están del Lab 11):

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.lab12

sudo tee -a /etc/ssh/sshd_config.d/hardening.conf << 'EOF'
# Hardening adicional Lab 12
AllowUsers labuser admin01
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no
Banner /etc/ssh/banner.txt
EOF
```

2. Crear el banner de advertencia:

```bash
sudo tee /etc/ssh/banner.txt << 'EOF'
*******************************************************************
*  AVISO: Sistema monitoreado. Acceso no autorizado prohibido.    *
*  Todas las sesiones son registradas y auditadas.                *
*******************************************************************
EOF
```

3. Reiniciar SSH y verificar:

```bash
sudo systemctl restart sshd
sudo systemctl status sshd
```

4. Configurar UFW con reglas restrictivas:

```bash
# Establecer políticas por defecto
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir SSH en puerto 2222
sudo ufw allow 2222/tcp comment "SSH personalizado"

# Permitir tráfico de loopback
sudo ufw allow in on lo

# Activar UFW
sudo ufw --force enable

# Verificar estado
sudo ufw status verbose
```

**Salida esperada:**

```
$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW IN    Anywhere                   # SSH personalizado
Anywhere on lo             ALLOW IN    Anywhere
2222/tcp (v6)              ALLOW IN    Anywhere (v6)              # SSH personalizado
Anywhere (v6) on lo        ALLOW IN    Anywhere (v6)
```

**Verificación:**

```bash
# Desde cli-linux, verificar que SSH sigue funcionando
# (ejecutar en cli-linux)
ssh -p 2222 labuser@192.168.100.10 "echo 'Conexión exitosa desde cli-linux'"
```

---

### Paso 5: Instalar y configurar fail2ban y auditd

**Objetivo:** Proteger el servicio SSH contra ataques de fuerza bruta con fail2ban y habilitar auditoría de accesos a archivos críticos con auditd.

**Instrucciones:**

1. Configurar fail2ban para SSH en puerto 2222:

```bash
sudo tee /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 600
findtime = 300
maxretry = 3
banaction = ufw

[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 1800
EOF
```

2. Reiniciar y verificar fail2ban:

```bash
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
sudo fail2ban-client status sshd
```

**Salida esperada:**

```
$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed:	0
|  |- Total failed:	0
|  `- File list:	/var/log/auth.log
`- Actions
   |- Currently banned:	0
   |- Total banned:	0
   `- Banned IP list:
```

3. Configurar auditd con reglas para monitorear archivos críticos:

```bash
# Habilitar y arrancar auditd
sudo systemctl enable auditd
sudo systemctl start auditd

# Agregar reglas de auditoría
sudo tee /etc/audit/rules.d/lab12-security.rules << 'EOF'
# Monitorear cambios en archivos de autenticación
-w /etc/passwd -p wa -k auth_changes
-w /etc/shadow -p wa -k auth_changes
-w /etc/group -p wa -k auth_changes
-w /etc/sudoers -p wa -k sudo_changes
-w /etc/sudoers.d/ -p wa -k sudo_changes

# Monitorear accesos al directorio del proyecto
-w /srv/proyecto/config -p rwxa -k proyecto_config
-w /srv/proyecto/logs -p wa -k proyecto_logs

# Monitorear uso de comandos privilegiados
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -k privileged_commands
EOF
```

4. Cargar las reglas y verificar:

```bash
sudo augenrules --load
sudo auditctl -l
```

**Salida esperada:**

```
$ sudo auditctl -l
-w /etc/passwd -p wa -k auth_changes
-w /etc/shadow -p wa -k auth_changes
-w /etc/group -p wa -k auth_changes
-w /etc/sudoers -p wa -k sudo_changes
-w /etc/sudoers.d/ -p wa -k sudo_changes
-w /srv/proyecto/config -p rwxa -k proyecto_config
-w /srv/proyecto/logs -p wa -k proyecto_logs
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -k privileged_commands
```

**Verificación:**

```bash
# Generar un evento de auditoría
sudo touch /srv/proyecto/config/test_audit.txt
sleep 2
sudo ausearch -k proyecto_config --start recent | tail -5
# Debe mostrar el evento de creación del archivo

# Limpiar archivo de prueba
sudo rm /srv/proyecto/config/test_audit.txt
```

---

### Paso 6: Cifrar archivos con GPG (asimétrico y simétrico)

**Objetivo:** Generar un par de llaves GPG RSA-4096 para `labuser`, cifrar el archivo `testfile_10MB.bin` del Lab 11 con cifrado asimétrico, y practicar cifrado simétrico AES-256 y firma digital.

**Instrucciones:**

1. Generar par de llaves GPG para labuser (modo no interactivo):

```bash
# Crear archivo de parámetros para generación batch
cat > /tmp/gpg-key-params << 'EOF'
%no-protection
Key-Type: RSA
Key-Length: 4096
Subkey-Type: RSA
Subkey-Length: 4096
Name-Real: Lab User
Name-Email: labuser@srv-linux.local
Expire-Date: 1y
%commit
EOF

# Generar la llave
gpg --batch --generate-key /tmp/gpg-key-params

# Limpiar archivo de parámetros
rm /tmp/gpg-key-params
```

2. Verificar la llave generada:

```bash
gpg --list-keys
gpg --list-secret-keys
```

**Salida esperada:**

```
$ gpg --list-keys
/home/labuser/.gnupg/pubring.kbx
---------------------------------
pub   rsa4096 2024-06-15 [SC] [expires: 2025-06-15]
      ABCDEF1234567890ABCDEF1234567890ABCDEF12
uid           [ultimate] Lab User <labuser@srv-linux.local>
sub   rsa4096 2024-06-15 [E] [expires: 2025-06-15]
```

3. Cifrar el archivo testfile_10MB.bin con cifrado asimétrico (llave pública):

```bash
# Verificar que el archivo existe
ls -lh /home/labuser/lab11/testfile_10MB.bin

# Cifrar con la llave pública de labuser
gpg --encrypt --recipient labuser@srv-linux.local \
    --output /home/labuser/lab11/testfile_10MB.bin.gpg \
    /home/labuser/lab11/testfile_10MB.bin

# Verificar el archivo cifrado
ls -lh /home/labuser/lab11/testfile_10MB.bin.gpg
```

4. Descifrar para verificar integridad:

```bash
gpg --decrypt --output /tmp/testfile_decrypted.bin \
    /home/labuser/lab11/testfile_10MB.bin.gpg

# Comparar checksums
sha256sum /home/labuser/lab11/testfile_10MB.bin /tmp/testfile_decrypted.bin
rm /tmp/testfile_decrypted.bin
```

5. Practicar cifrado simétrico con AES-256:

```bash
# Crear un archivo de ejemplo con datos sensibles
echo "Credenciales del servidor: root:R00t@Linux2024!" > /home/labuser/lab11/credenciales.txt

# Cifrar con AES-256 (se pedirá passphrase, usar: GPG@Secure2024!)
gpg --symmetric --cipher-algo AES256 \
    --output /home/labuser/lab11/credenciales.txt.gpg \
    --batch --passphrase "GPG@Secure2024!" \
    /home/labuser/lab11/credenciales.txt

# Eliminar el archivo en texto plano
shred -u /home/labuser/lab11/credenciales.txt

# Verificar que solo queda el archivo cifrado
ls /home/labuser/lab11/credenciales*
```

6. Practicar firma digital:

```bash
# Crear un documento para firmar
echo "Reporte de auditoría - Fecha: $(date)" > /home/labuser/lab11/reporte_auditoria.txt

# Firmar el documento (firma separada)
gpg --detach-sign --armor \
    --output /home/labuser/lab11/reporte_auditoria.txt.sig \
    /home/labuser/lab11/reporte_auditoria.txt

# Verificar la firma
gpg --verify /home/labuser/lab11/reporte_auditoria.txt.sig \
    /home/labuser/lab11/reporte_auditoria.txt
```

**Salida esperada:**

```
$ gpg --verify /home/labuser/lab11/reporte_auditoria.txt.sig /home/labuser/lab11/reporte_auditoria.txt
gpg: Signature made Thu 15 Jun 2024 10:30:00 AM UTC
gpg:                using RSA key ABCDEF1234567890ABCDEF1234567890ABCDEF12
gpg: Good signature from "Lab User <labuser@srv-linux.local>" [ultimate]
```

**Verificación:**

```bash
# Exportar llave pública (útil para compartir)
gpg --export --armor labuser@srv-linux.local > /home/labuser/lab11/labuser_pubkey.asc
file /home/labuser/lab11/labuser_pubkey.asc
# Debe indicar: PGP public key block
```

---

### Paso 7: Crear y gestionar volumen cifrado LUKS en /dev/sdb

**Objetivo:** Crear un volumen LUKS2 en el disco secundario `/dev/sdb`, formatearlo con ext4, montarlo en `/mnt/secure_vault/` y configurarlo para almacenamiento seguro de llaves y datos confidenciales.

**Instrucciones:**

1. Verificar el disco disponible y que no tiene particiones:

```bash
sudo lsblk /dev/sdb
sudo wipefs -a /dev/sdb
```

2. Inicializar el volumen LUKS2 (passphrase: `LUKS@Vault2024!`):

```bash
# Formatear como LUKS2
echo -n "LUKS@Vault2024!" | sudo cryptsetup luksFormat --type luks2 \
    --cipher aes-xts-plain64 \
    --key-size 512 \
    --hash sha512 \
    --iter-time 3000 \
    /dev/sdb -

# Confirmar que se creó correctamente
sudo cryptsetup luksDump /dev/sdb | head -20
```

**Salida esperada:**

```
$ sudo cryptsetup luksDump /dev/sdb | head -20
LUKS header information
Version:       	2
Epoch:         	3
Metadata area: 	16384 [bytes]
Keyslots area: 	16744448 [bytes]
UUID:          	xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Label:         	(no label)
Subsystem:     	(no subsystem)
Flags:       	(no flags)

Data segments:
  0: crypt
	offset: 16777216 [bytes]
	length: (whole device)
	cipher: aes-xts-plain64
	sector: 512
```

3. Abrir (desbloquear) el volumen LUKS:

```bash
echo -n "LUKS@Vault2024!" | sudo cryptsetup luksOpen /dev/sdb secure_vault -
```

4. Crear sistema de archivos ext4 en el volumen descifrado:

```bash
sudo mkfs.ext4 -L "SecureVault" /dev/mapper/secure_vault
```

5. Crear punto de montaje y montar:

```bash
sudo mkdir -p /mnt/secure_vault
sudo mount /dev/mapper/secure_vault /mnt/secure_vault

# Establecer permisos restrictivos
sudo chown labuser:administradores /mnt/secure_vault
sudo chmod 700 /mnt/secure_vault
```

6. Verificar el montaje:

```bash
df -h /mnt/secure_vault
lsblk /dev/sdb
```

**Salida esperada:**

```
$ lsblk /dev/sdb
NAME              MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sdb                 8:16   0    2G  0 disk
└─secure_vault    253:0    0    2G  0 crypt /mnt/secure_vault
```

7. Almacenar datos sensibles en el vault:

```bash
# Copiar la llave privada GPG exportada al vault
gpg --export-secret-keys --armor labuser@srv-linux.local > /mnt/secure_vault/labuser_private_key.asc

# Copiar el archivo cifrado de credenciales
cp /home/labuser/lab11/credenciales.txt.gpg /mnt/secure_vault/

# Crear un archivo de notas seguras
cat > /mnt/secure_vault/notas_seguras.txt << 'EOF'
=== VAULT DE SEGURIDAD ===
Fecha de creación: $(date)
Propósito: Almacenamiento de llaves privadas y credenciales cifradas
Acceso: Solo labuser y administradores autorizados
EOF

# Verificar contenido
ls -la /mnt/secure_vault/
```

8. Crear script para montaje/desmontaje seguro:

```bash
sudo tee /opt/scripts/vault_manager.sh << 'SCRIPT'
#!/bin/bash
# Script de gestión del vault LUKS
VAULT_DEV="/dev/sdb"
VAULT_NAME="secure_vault"
VAULT_MOUNT="/mnt/secure_vault"

case "$1" in
    open)
        echo "Abriendo vault..."
        sudo cryptsetup luksOpen $VAULT_DEV $VAULT_NAME
        sudo mount /dev/mapper/$VAULT_NAME $VAULT_MOUNT
        echo "Vault montado en $VAULT_MOUNT"
        ;;
    close)
        echo "Cerrando vault..."
        sudo umount $VAULT_MOUNT
        sudo cryptsetup luksClose $VAULT_NAME
        echo "Vault cerrado y bloqueado"
        ;;
    status)
        if [ -e /dev/mapper/$VAULT_NAME ]; then
            echo "Vault: ABIERTO"
            df -h $VAULT_MOUNT
        else
            echo "Vault: CERRADO"
        fi
        ;;
    *)
        echo "Uso: $0 {open|close|status}"
        exit 1
        ;;
esac
SCRIPT

sudo chmod 750 /opt/scripts/vault_manager.sh
```

**Verificación:**

```bash
# Probar desmontaje y cierre seguro
sudo umount /mnt/secure_vault
sudo cryptsetup luksClose secure_vault

# Verificar que está cerrado
lsblk /dev/sdb
# No debe mostrar el mapper

# Reabrir para continuar el lab
echo -n "LUKS@Vault2024!" | sudo cryptsetup luksOpen /dev/sdb secure_vault -
sudo mount /dev/mapper/secure_vault /mnt/secure_vault
```

---

### Paso 8: Auditoría de seguridad con Lynis y plan de remediación

**Objetivo:** Ejecutar una auditoría completa del sistema con Lynis, analizar los hallazgos críticos y documentar un plan de remediación.

**Instrucciones:**

1. Ejecutar auditoría completa con Lynis:

```bash
sudo lynis audit system --no-colors 2>&1 | tee /opt/sysreport/lynis_audit_lab12.txt
```

2. Extraer el puntaje de hardening y hallazgos críticos:

```bash
# Ver puntaje
grep "Hardening index" /opt/sysreport/lynis_audit_lab12.txt

# Ver advertencias
grep -A1 "Warning" /opt/sysreport/lynis_audit_lab12.txt | head -30

# Ver sugerencias prioritarias
grep "suggestion\[\]" /var/log/lynis.log | head -20
```

**Salida esperada (ejemplo):**

```
Hardening index : 72 [##############        ]
```

3. Remediar hallazgos comunes identificados por Lynis:

```bash
# Remediación 1: Configurar políticas de contraseña en PAM
sudo tee /etc/security/pwquality.conf << 'EOF'
minlen = 12
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1
maxrepeat = 3
EOF

# Remediación 2: Deshabilitar servicios innecesarios
sudo systemctl disable --now cups-browsed.service 2>/dev/null
sudo systemctl disable --now avahi-daemon.service 2>/dev/null

# Remediación 3: Configurar permisos restrictivos en archivos cron
sudo chmod 600 /etc/crontab
sudo chmod 700 /etc/cron.d
sudo chmod 700 /etc/cron.daily
sudo chmod 700 /etc/cron.hourly

# Remediación 4: Asegurar que /tmp tiene opciones de montaje seguras
echo "tmpfs /tmp tmpfs defaults,noexec,nosuid,nodev 0 0" | sudo tee -a /etc/fstab

# Remediación 5: Configurar parámetros de kernel para seguridad de red
sudo tee /etc/sysctl.d/99-security.conf << 'EOF'
# Deshabilitar IP forwarding (si no es router)
net.ipv4.ip_forward = 0
# Ignorar ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
# Habilitar protección SYN flood
net.ipv4.tcp_syncookies = 1
# Registrar paquetes marcianos
net.ipv4.conf.all.log_martians = 1
# Deshabilitar source routing
net.ipv4.conf.all.accept_source_route = 0
EOF

sudo sysctl --system
```

4. Documentar el plan de remediación:

```bash
sudo mkdir -p /opt/sysreport
cat > /opt/sysreport/plan_remediacion_lab12.md << 'EOF'
# Plan de Remediación - Auditoría Lynis Lab 12

## Fecha: $(date +%Y-%m-%d)
## Servidor: srv-linux (192.168.100.10)

### Controles Implementados:
1. [x] Permisos POSIX y ACLs en /srv/proyecto/
2. [x] Sudo con mínimo privilegio (/etc/sudoers.d/)
3. [x] SSH hardening (puerto 2222, AllowUsers, MaxAuthTries)
4. [x] UFW activo (deny incoming, allow 2222/tcp)
5. [x] fail2ban protegiendo SSH
6. [x] auditd monitoreando archivos críticos
7. [x] GPG para cifrado de archivos sensibles
8. [x] LUKS2 para cifrado de dispositivo de bloque
9. [x] Políticas de contraseña en PAM
10. [x] Parámetros sysctl de seguridad de red

### Hallazgos Pendientes (Prioridad Media):
- Implementar AIDE para integridad de archivos
- Configurar logrotate para logs de auditoría
- Habilitar AppArmor en modo enforce para servicios

### Puntaje Lynis Objetivo: >= 80
EOF
```

5. Re-ejecutar Lynis para verificar mejora:

```bash
sudo lynis audit system --no-colors --quick 2>&1 | grep "Hardening index"
```

**Verificación:**

```bash
# El puntaje debe haber mejorado respecto a la primera ejecución
cat /opt/sysreport/plan_remediacion_lab12.md
ls -la /opt/sysreport/lynis_audit_lab12.txt
```

---

## Validación y Pruebas Finales

Ejecutar la siguiente secuencia de validaciones para confirmar que todos los controles están operativos:

```bash
echo "=== VALIDACIÓN INTEGRAL DEL LABORATORIO 12 ==="
echo ""

echo "1. ACLs en /srv/proyecto/:"
getfacl /srv/proyecto/logs | grep -c "auditores"
# Esperado: al menos 1

echo "2. Archivos sudoers válidos:"
sudo visudo -c 2>&1 | grep -c "parsed OK"
# Esperado: al menos 4

echo "3. UFW activo:"
sudo ufw status | grep -c "Status: active"
# Esperado: 1

echo "4. fail2ban protegiendo SSH:"
sudo fail2ban-client status sshd | grep -c "enabled"
# Esperado: al menos 0 (jail activa)

echo "5. Reglas de auditoría cargadas:"
sudo auditctl -l | grep -c "auth_changes\|proyecto_config"
# Esperado: al menos 3

echo "6. Llave GPG presente:"
gpg --list-keys | grep -c "labuser@srv-linux.local"
# Esperado: 1

echo "7. Archivo cifrado GPG existe:"
ls /home/labuser/lab11/testfile_10MB.bin.gpg > /dev/null 2>&1 && echo "OK" || echo "FALTA"

echo "8. Volumen LUKS montado:"
mount | grep -c "secure_vault"
# Esperado: 1

echo "9. Datos en vault:"
ls /mnt/secure_vault/ | wc -l
# Esperado: al menos 3

echo "10. Lynis report generado:"
ls /opt/sysreport/lynis_audit_lab12.txt > /dev/null 2>&1 && echo "OK" || echo "FALTA"

echo ""
echo "=== VALIDACIÓN COMPLETA ==="
```

---

## Solución de Problemas

### Problema 1: fail2ban no inicia — "No file(s) found for glob /var/log/auth.log"

**Síntomas:** El servicio fail2ban falla al iniciar. `systemctl status fail2ban` muestra error relacionado con la ruta del log. `journalctl -u fail2ban` indica que no encuentra `/var/log/auth.log`.

**Causa:** En algunas configuraciones de Ubuntu 22.04 con journald como único backend de logging, el archivo `/var/log/auth.log` no existe porque rsyslog no está instalado o no está configurado para escribir logs de autenticación.

**Solución:**

```bash
# Verificar si rsyslog está activo
sudo systemctl status rsyslog

# Si no está instalado:
sudo apt install -y rsyslog
sudo systemctl enable --now rsyslog

# Verificar que el archivo se crea
sleep 5
ls -la /var/log/auth.log

# Reiniciar fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

### Problema 2: cryptsetup luksOpen falla con "No key available with this passphrase"

**Síntomas:** Al intentar abrir el volumen LUKS con `cryptsetup luksOpen`, el sistema rechaza la passphrase aunque se ingrese correctamente. El error indica que no hay slot de llave que coincida.

**Causa:** Al usar `echo -n` para pasar la passphrase por pipe, si se omite el `-n`, se incluye un carácter de nueva línea (`\n`) que se convierte en parte de la passphrase almacenada. Al intentar abrir manualmente (sin pipe), la passphrase no coincide porque no incluye ese `\n`.

**Solución:**

```bash
# Opción 1: Usar printf sin newline para pasar la passphrase
printf '%s' "LUKS@Vault2024!" | sudo cryptsetup luksOpen /dev/sdb secure_vault -

# Opción 2: Si el volumen se creó con newline accidental, agregar una nueva llave correcta
# (requiere la passphrase original con el \n)
echo "LUKS@Vault2024!" | sudo cryptsetup luksAddKey /dev/sdb - <<< "NuevaPass@2024!"

# Opción 3: Recrear el volumen (si no hay datos importantes)
sudo cryptsetup luksFormat --type luks2 /dev/sdb
# Esta vez usar printf para evitar el problema:
printf '%s' "LUKS@Vault2024!" | sudo cryptsetup luksFormat --type luks2 /dev/sdb -
```

---

## Limpieza

Si necesitas revertir los cambios del laboratorio (solo en entornos de práctica):

```bash
# Cerrar vault LUKS
sudo umount /mnt/secure_vault 2>/dev/null
sudo cryptsetup luksClose secure_vault 2>/dev/null

# Desactivar UFW (PRECAUCIÓN: deja el sistema sin firewall)
# sudo ufw disable

# Eliminar usuarios de prueba
sudo userdel -r dev01 2>/dev/null
sudo userdel -r dev02 2>/dev/null
sudo userdel -r auditor01 2>/dev/null
sudo userdel -r admin01 2>/dev/null

# Eliminar grupos
sudo groupdel desarrolladores 2>/dev/null
sudo groupdel auditores 2>/dev/null
sudo groupdel administradores 2>/dev/null

# Eliminar archivos sudoers personalizados
sudo rm -f /etc/sudoers.d/{desarrolladores,auditores,administradores}

# Eliminar estructura de proyecto
sudo rm -rf /srv/proyecto

# NOTA: NO ejecutar limpieza si planeas usar esta configuración
# como base para laboratorios futuros
```

---

## Resumen

En este laboratorio se implementó una estrategia de defensa en profundidad que abarca múltiples capas de seguridad:

| Capa | Control Implementado | Herramienta |
|------|---------------------|-------------|
| Acceso a archivos | Permisos POSIX + ACLs | chmod, setfacl |
| Privilegios | Sudo con mínimo privilegio | sudoers.d |
| Red | Firewall restrictivo | UFW |
| Servicios | Protección contra fuerza bruta | fail2ban |
| Auditoría | Registro de accesos críticos | auditd |
| Datos en reposo (archivos) | Cifrado asimétrico/simétrico | GPG |
| Datos en reposo (disco) | Cifrado de dispositivo de bloque | LUKS2 |
| Evaluación continua | Auditoría de hardening | Lynis |

### Conceptos Clave Consolidados

- **Principio de mínimo privilegio:** Cada usuario y proceso debe tener únicamente los permisos estrictamente necesarios para su función.
- **Defensa en profundidad:** Múltiples capas de seguridad garantizan que la falla de un control no comprometa todo el sistema.
- **Cifrado en reposo:** Tanto GPG (archivos individuales) como LUKS (dispositivos completos) protegen datos confidenciales contra acceso físico no autorizado.
- **Auditoría continua:** Las herramientas como auditd y Lynis permiten detectar desviaciones y mantener una postura de seguridad verificable.

### Recursos Adicionales

- `man 5 acl` — Documentación de ACLs POSIX
- `man cryptsetup` — Referencia completa de LUKS
- [CIS Benchmarks para Ubuntu](https://www.cisecurity.org/benchmark/ubuntu_linux) — Guías de hardening profesional
- `man sudoers` — Sintaxis completa de archivos sudoers
