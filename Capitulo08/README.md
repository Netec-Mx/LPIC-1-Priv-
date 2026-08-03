# Configurar entornos gráficos, gestores de sesión y perfiles de usuario para habilitar estaciones de trabajo Linux funcionales

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 49 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **VM principal** | srv-linux-01 (Ubuntu 22.04.4 LTS) |
| **IP** | 192.168.100.10 |

## Descripción General

En este laboratorio instalarás y configurarás un entorno de escritorio GNOME completo sobre Ubuntu Server 22.04.4 LTS, administrarás el gestor de visualización GDM3 para controlar el acceso gráfico, crearás perfiles de usuario corporativos con configuraciones diferenciadas y explorarás las diferencias prácticas entre sesiones X11 y Wayland. Al finalizar, la VM srv-linux-01 funcionará como una estación de trabajo gráfica lista para uso corporativo.

## Objetivos de Aprendizaje

- [ ] Instalar y configurar el entorno de escritorio GNOME 42.9 sobre Ubuntu Server 22.04.4 LTS con el target gráfico de systemd activo.
- [ ] Administrar GDM3 42.0 para controlar sesiones gráficas, incluyendo la selección entre X11 y Wayland.
- [ ] Crear y configurar perfiles de usuario corporativos con políticas diferenciadas mediante dconf/gsettings.
- [ ] Comparar la arquitectura X11 vs Wayland identificando el protocolo activo y sus características.
- [ ] Automatizar la configuración de perfiles GNOME mediante un script Bash reutilizable.

## Prerrequisitos

### Conocimiento previo
- Gestión de paquetes APT (instalación, actualización de repositorios)
- Administración de servicios con systemctl (targets, enable/disable)
- Creación y gestión de usuarios Linux (useradd, passwd)
- Arquitectura del sistema gráfico Linux (X11 y Wayland — Lección 8.1)

### Acceso requerido
- VM srv-linux-01 operativa con Ubuntu Server 22.04.4 LTS
- Acceso como usuario `sysadmin` con privilegios sudo
- Conexión a internet (adaptador NAT activo en VirtualBox)
- Mínimo 5 GB de espacio libre en disco para la instalación de GNOME
- Aceleración gráfica habilitada en la configuración de VirtualBox (Display → Graphics Controller: VMSVGA, Video Memory: 128 MB)

## Entorno del Laboratorio

### Configuración de Hardware Virtual (VirtualBox)

| Parámetro | Valor requerido |
|-----------|----------------|
| RAM asignada | Mínimo 2048 MB (recomendado 4096 MB) |
| Controlador gráfico | VMSVGA |
| Memoria de video | 128 MB |
| Aceleración 3D | Habilitada |
| Disco sistema (/dev/sda) | 40 GB |

### Software involucrado

| Componente | Versión |
|------------|---------|
| Ubuntu Server | 22.04.4 LTS |
| GNOME Desktop | 42.9 |
| GDM3 | 42.0 |
| X.Org Server | 1.21.1.3 |
| Wayland (libwayland-server) | 1.20.0 |
| dconf-cli | 0.40.0 |
| systemd | 249.11 |
| VirtualBox Guest Additions | 7.0.14 |

### Preparación inicial

Conéctate a srv-linux-01 por SSH o accede a la consola directamente:

```bash
ssh sysadmin@192.168.100.10
```

Verifica el espacio disponible y actualiza los repositorios:

```bash
df -h /
sudo apt update && sudo apt upgrade -y
```

---

## Procedimiento Paso a Paso

### Paso 1: Verificar el estado actual del sistema y el target de systemd

**Objetivo:** Confirmar que el sistema opera en modo texto (multi-user.target) antes de la instalación del entorno gráfico.

**Instrucciones:**

1. Verifica el target activo de systemd:

```bash
systemctl get-default
```

2. Confirma que no existe ningún servidor X ni compositor Wayland en ejecución:

```bash
ps aux | grep -E "Xorg|Xwayland|wayland|gdm" | grep -v grep
```

3. Verifica que las variables de entorno gráficas no están definidas:

```bash
echo "XDG_SESSION_TYPE: $XDG_SESSION_TYPE"
echo "DISPLAY: $DISPLAY"
echo "WAYLAND_DISPLAY: $WAYLAND_DISPLAY"
```

4. Verifica los módulos de kernel gráficos disponibles:

```bash
lsmod | grep -E "drm|vbox"
```

**Salida esperada:**

```
# systemctl get-default
multi-user.target

# Variables de entorno (vacías o "tty")
XDG_SESSION_TYPE: tty
DISPLAY:
WAYLAND_DISPLAY:

# Módulos DRM cargados
drm_vram_helper        ...
drm_ttm_helper         ...
drm_kms_helper         ...
drm                    ...
vboxvideo              ...
```

**Verificación:** El target debe ser `multi-user.target` y no deben existir procesos gráficos activos. Los módulos DRM y vboxvideo deben estar cargados.

---

### Paso 2: Instalar el entorno de escritorio GNOME y GDM3

**Objetivo:** Instalar el paquete completo del escritorio GNOME junto con GDM3 como gestor de visualización.

**Instrucciones:**

1. Instala el metapaquete del escritorio GNOME de Ubuntu:

```bash
sudo apt install -y ubuntu-desktop-minimal
```

> **Nota:** `ubuntu-desktop-minimal` instala GNOME 42.9 con GDM3 y las aplicaciones esenciales sin incluir software adicional voluminoso. La descarga es de aproximadamente 1.5-2 GB.

2. Verifica que GDM3 se instaló correctamente:

```bash
dpkg -l | grep gdm3
gdm3 --version 2>/dev/null || dpkg -s gdm3 | grep Version
```

3. Verifica la instalación de los componentes de GNOME:

```bash
dpkg -l | grep -E "gnome-shell|gnome-session|mutter"
```

4. Confirma que los paquetes de X.Org y Wayland están presentes:

```bash
dpkg -l | grep -E "xorg|xwayland|libwayland"
```

5. Instala herramientas adicionales necesarias para la configuración:

```bash
sudo apt install -y dconf-cli dconf-editor gnome-tweaks
```

**Salida esperada:**

```
# dpkg -s gdm3 | grep Version
Version: 42.0-1ubuntu7

# dpkg -l | grep gnome-shell
ii  gnome-shell    42.9-0ubuntu2    amd64    graphical shell for the GNOME desktop

# dpkg -l | grep xwayland
ii  xwayland       2:22.1.1-1      amd64    X server for running X clients under Wayland
```

**Verificación:** Los paquetes `gdm3`, `gnome-shell`, `gnome-session`, `mutter`, `xorg`, `xwayland` y `libwayland-server0` deben aparecer como instalados (estado `ii`).

---

### Paso 3: Configurar el target gráfico y habilitar GDM3

**Objetivo:** Cambiar el target predeterminado de systemd a graphical.target y configurar GDM3 como gestor de display activo.

**Instrucciones:**

1. Establece el target gráfico como predeterminado:

```bash
sudo systemctl set-default graphical.target
```

2. Verifica el cambio:

```bash
systemctl get-default
```

3. Confirma que GDM3 está habilitado como servicio:

```bash
systemctl is-enabled gdm3
```

4. Si no está habilitado, habilítalo:

```bash
sudo systemctl enable gdm3
```

5. Verifica que GDM3 es el gestor de display predeterminado:

```bash
cat /etc/X11/default-display-manager
```

6. Inicia el entorno gráfico sin reiniciar (para validación inmediata):

```bash
sudo systemctl isolate graphical.target
```

> **Nota:** Si estás conectado por SSH, la sesión SSH permanecerá activa. El entorno gráfico se mostrará en la consola de VirtualBox.

7. Verifica que GDM3 está en ejecución:

```bash
systemctl status gdm3 --no-pager
```

**Salida esperada:**

```
# systemctl get-default
graphical.target

# systemctl is-enabled gdm3
enabled

# cat /etc/X11/default-display-manager
/usr/sbin/gdm3

# systemctl status gdm3
● gdm.service - GNOME Display Manager
     Loaded: loaded (/lib/systemd/system/gdm.service; enabled; ...)
     Active: active (running) since ...
```

**Verificación:** El target debe ser `graphical.target`, GDM3 debe estar `enabled` y `active (running)`.

---

### Paso 4: Configurar sesiones X11 y Wayland en GDM3

**Objetivo:** Configurar GDM3 para ofrecer ambas opciones de sesión (X11 y Wayland) y comprender cómo seleccionar entre ellas.

**Instrucciones:**

1. Examina el archivo de configuración principal de GDM3:

```bash
cat /etc/gdm3/custom.conf
```

2. Verifica que Wayland está habilitado (opción por defecto en Ubuntu 22.04). El archivo debe contener:

```bash
sudo grep -n "WaylandEnable" /etc/gdm3/custom.conf
```

3. Para asegurar que **ambas sesiones** están disponibles, edita el archivo de configuración:

```bash
sudo nano /etc/gdm3/custom.conf
```

Asegúrate de que la sección `[daemon]` contenga:

```ini
[daemon]
WaylandEnable=true

[security]

[xdmcp]

[chooser]

[debug]
```

4. Verifica las sesiones de escritorio disponibles:

```bash
ls /usr/share/xsessions/
ls /usr/share/wayland-sessions/
```

5. Examina el contenido de un archivo de sesión X11:

```bash
cat /usr/share/xsessions/ubuntu-xorg.desktop
```

6. Examina el contenido de un archivo de sesión Wayland:

```bash
cat /usr/share/wayland-sessions/ubuntu-wayland.desktop 2>/dev/null || \
cat /usr/share/wayland-sessions/ubuntu.desktop
```

7. Para **deshabilitar Wayland temporalmente** (útil en entornos virtualizados con problemas gráficos), modifica:

```bash
sudo sed -i 's/^#WaylandEnable=false/WaylandEnable=false/' /etc/gdm3/custom.conf
```

8. Para **rehabilitar Wayland**:

```bash
sudo sed -i 's/^WaylandEnable=false/#WaylandEnable=false/' /etc/gdm3/custom.conf
```

> **Importante para VirtualBox:** En entornos virtualizados, Wayland puede presentar problemas de rendimiento. Para este laboratorio, mantendremos Wayland habilitado pero usaremos X11 como sesión principal por estabilidad.

9. Reinicia GDM3 para aplicar cambios:

```bash
sudo systemctl restart gdm3
```

**Salida esperada:**

```
# ls /usr/share/xsessions/
ubuntu-xorg.desktop

# ls /usr/share/wayland-sessions/
ubuntu.desktop

# cat /usr/share/xsessions/ubuntu-xorg.desktop
[Desktop Entry]
Name=Ubuntu on Xorg
Exec=env GNOME_SHELL_SESSION_MODE=ubuntu /usr/bin/gnome-session --session=ubuntu
...
```

**Verificación:** Deben existir archivos `.desktop` tanto en `/usr/share/xsessions/` como en `/usr/share/wayland-sessions/`, confirmando que ambas opciones están disponibles para los usuarios en la pantalla de login.

---

### Paso 5: Crear usuarios corporativos con perfiles diferenciados

**Objetivo:** Crear tres usuarios corporativos (user_corp1, user_access1, user_admin1) con configuraciones específicas para diferentes roles organizacionales.

**Instrucciones:**

1. Crea el usuario estándar corporativo:

```bash
sudo useradd -m -s /bin/bash -c "Usuario Corporativo Estándar" user_corp1
sudo passwd user_corp1
# Contraseña: Corp@User2024!
```

2. Crea el usuario con necesidades de accesibilidad:

```bash
sudo useradd -m -s /bin/bash -c "Usuario con Accesibilidad" user_access1
sudo passwd user_access1
# Contraseña: Access@User2024!
```

3. Crea el usuario administrador gráfico:

```bash
sudo useradd -m -s /bin/bash -c "Administrador Gráfico" user_admin1
sudo passwd user_admin1
# Contraseña: Admin@Graph2024!
```

4. Agrega user_admin1 al grupo sudo para privilegios administrativos:

```bash
sudo usermod -aG sudo user_admin1
```

5. Verifica la creación de los usuarios:

```bash
id user_corp1
id user_access1
id user_admin1
```

6. Confirma que los directorios home se crearon:

```bash
ls -la /home/ | grep -E "user_corp1|user_access1|user_admin1"
```

**Salida esperada:**

```
# id user_corp1
uid=1001(user_corp1) gid=1001(user_corp1) groups=1001(user_corp1)

# id user_admin1
uid=1003(user_admin1) gid=1003(user_admin1) groups=1003(user_admin1),27(sudo)

# ls -la /home/
drwxr-x--- 2 user_corp1   user_corp1   4096 ... user_corp1
drwxr-x--- 2 user_access1 user_access1 4096 ... user_access1
drwxr-x--- 2 user_admin1  user_admin1  4096 ... user_admin1
```

**Verificación:** Los tres usuarios deben existir con UIDs únicos, user_admin1 debe pertenecer al grupo `sudo`, y cada uno debe tener su directorio home creado.

---

### Paso 6: Configurar perfiles GNOME con dconf/gsettings

**Objetivo:** Aplicar configuraciones GNOME diferenciadas para cada usuario utilizando dconf y gsettings, incluyendo políticas de pantalla de bloqueo y timeout de sesión.

**Instrucciones:**

1. Crea el directorio para perfiles dconf del sistema:

```bash
sudo mkdir -p /etc/dconf/profile
sudo mkdir -p /etc/dconf/db/local.d
sudo mkdir -p /etc/dconf/db/local.d/locks
```

2. Crea el perfil dconf para los usuarios:

```bash
sudo tee /etc/dconf/profile/user << 'EOF'
user-db:user
system-db:local
EOF
```

3. Configura las políticas corporativas globales (pantalla de bloqueo y timeout):

```bash
sudo tee /etc/dconf/db/local.d/00-corporate-policy << 'EOF'
# Política corporativa: pantalla de bloqueo
[org/gnome/desktop/session]
idle-delay=uint32 600

[org/gnome/desktop/screensaver]
lock-enabled=true
lock-delay=uint32 0
idle-activation-enabled=true

[org/gnome/desktop/notifications]
show-in-lock-screen=false

[org/gnome/desktop/privacy]
remember-recent-files=false
remove-old-temp-files=true
old-files-age=uint32 7
EOF
```

4. Bloquea ciertas configuraciones para que los usuarios no puedan modificarlas:

```bash
sudo tee /etc/dconf/db/local.d/locks/00-corporate-locks << 'EOF'
/org/gnome/desktop/screensaver/lock-enabled
/org/gnome/desktop/screensaver/lock-delay
/org/gnome/desktop/session/idle-delay
EOF
```

5. Actualiza la base de datos dconf del sistema:

```bash
sudo dconf update
```

6. Configura el perfil específico de **user_corp1** (tema claro, sin animaciones):

```bash
sudo -u user_corp1 dbus-launch gsettings set org.gnome.desktop.interface color-scheme 'default'
sudo -u user_corp1 dbus-launch gsettings set org.gnome.desktop.interface enable-animations false
sudo -u user_corp1 dbus-launch gsettings set org.gnome.desktop.interface clock-show-seconds true
sudo -u user_corp1 dbus-launch gsettings set org.gnome.desktop.interface clock-format '24h'
```

7. Configura el perfil de **user_access1** (accesibilidad habilitada):

```bash
sudo -u user_access1 dbus-launch gsettings set org.gnome.desktop.interface text-scaling-factor 1.5
sudo -u user_access1 dbus-launch gsettings set org.gnome.desktop.a11y.magnifier mag-factor 2.0
sudo -u user_access1 dbus-launch gsettings set org.gnome.desktop.a11y.magnifier mouse-tracking 'proportional'
sudo -u user_access1 dbus-launch gsettings set org.gnome.desktop.a11y always-show-universal-access-status true
sudo -u user_access1 dbus-launch gsettings set org.gnome.desktop.interface cursor-size 48
sudo -u user_access1 dbus-launch gsettings set org.gnome.desktop.wm.preferences theme 'HighContrast'
```

8. Configura el perfil de **user_admin1** (tema oscuro, extensiones de productividad):

```bash
sudo -u user_admin1 dbus-launch gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'
sudo -u user_admin1 dbus-launch gsettings set org.gnome.desktop.interface show-battery-percentage true
sudo -u user_admin1 dbus-launch gsettings set org.gnome.desktop.interface clock-show-seconds true
sudo -u user_admin1 dbus-launch gsettings set org.gnome.desktop.interface clock-format '24h'
sudo -u user_admin1 dbus-launch gsettings set org.gnome.desktop.wm.preferences button-layout 'appmenu:minimize,maximize,close'
```

9. Verifica las configuraciones aplicadas:

```bash
sudo -u user_corp1 dbus-launch gsettings get org.gnome.desktop.interface enable-animations
sudo -u user_access1 dbus-launch gsettings get org.gnome.desktop.interface text-scaling-factor
sudo -u user_admin1 dbus-launch gsettings get org.gnome.desktop.interface color-scheme
```

**Salida esperada:**

```
# user_corp1 - animaciones
false

# user_access1 - escala de texto
1.5

# user_admin1 - esquema de color
'prefer-dark'
```

**Verificación:** Cada usuario debe tener sus configuraciones específicas aplicadas. Las políticas globales de bloqueo de pantalla (idle-delay=600) deben estar activas para todos.

---

### Paso 7: Crear el script de configuración automatizada de perfiles

**Objetivo:** Desarrollar un script reutilizable que automatice la configuración de perfiles GNOME para nuevos usuarios corporativos.

**Instrucciones:**

1. Crea el directorio de scripts si no existe:

```bash
sudo mkdir -p /opt/scripts
```

2. Crea el script de configuración automatizada:

```bash
sudo tee /opt/scripts/setup_gnome_profile.sh << 'SCRIPT'
#!/bin/bash
#================================================================
# Script: setup_gnome_profile.sh
# Descripción: Configura perfiles GNOME para usuarios corporativos
# Uso: sudo ./setup_gnome_profile.sh <usuario> <perfil>
# Perfiles disponibles: standard, accessibility, admin
# Autor: sysadmin
# Fecha: $(date +%Y-%m-%d)
#================================================================

LOGFILE="/var/log/admin_scripts.log"

# Función de logging
log_msg() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') [GNOME-PROFILE] $1" | sudo tee -a "$LOGFILE"
}

# Validar argumentos
if [ $# -ne 2 ]; then
    echo "Uso: $0 <nombre_usuario> <perfil>"
    echo "Perfiles disponibles: standard, accessibility, admin"
    exit 1
fi

USERNAME="$1"
PROFILE="$2"

# Verificar que el usuario existe
if ! id "$USERNAME" &>/dev/null; then
    echo "ERROR: El usuario '$USERNAME' no existe."
    log_msg "ERROR: Intento de configurar usuario inexistente: $USERNAME"
    exit 2
fi

# Función para aplicar gsettings como el usuario objetivo
apply_setting() {
    sudo -u "$USERNAME" dbus-launch gsettings set "$1" "$2" 2>/dev/null
    if [ $? -eq 0 ]; then
        log_msg "OK: $USERNAME -> $1 = $2"
    else
        log_msg "WARN: No se pudo aplicar $1 para $USERNAME"
    fi
}

# Configuraciones comunes a todos los perfiles
apply_common() {
    log_msg "Aplicando configuración común para: $USERNAME"
    apply_setting org.gnome.desktop.interface clock-format "'24h'"
    apply_setting org.gnome.desktop.interface clock-show-seconds true
    apply_setting org.gnome.desktop.privacy remember-recent-files false
    apply_setting org.gnome.desktop.privacy remove-old-temp-files true
}

# Perfil estándar
apply_standard() {
    log_msg "Aplicando perfil STANDARD para: $USERNAME"
    apply_common
    apply_setting org.gnome.desktop.interface color-scheme "'default'"
    apply_setting org.gnome.desktop.interface enable-animations false
    apply_setting org.gnome.desktop.interface cursor-size 24
    apply_setting org.gnome.desktop.background picture-uri "''"
    apply_setting org.gnome.desktop.background primary-color "'#2c3e50'"
}

# Perfil accesibilidad
apply_accessibility() {
    log_msg "Aplicando perfil ACCESSIBILITY para: $USERNAME"
    apply_common
    apply_setting org.gnome.desktop.interface text-scaling-factor 1.5
    apply_setting org.gnome.desktop.interface cursor-size 48
    apply_setting org.gnome.desktop.a11y always-show-universal-access-status true
    apply_setting org.gnome.desktop.a11y.magnifier mag-factor 2.0
    apply_setting org.gnome.desktop.a11y.magnifier mouse-tracking "'proportional'"
    apply_setting org.gnome.desktop.wm.preferences theme "'HighContrast'"
    apply_setting org.gnome.desktop.interface color-scheme "'prefer-dark'"
}

# Perfil administrador
apply_admin() {
    log_msg "Aplicando perfil ADMIN para: $USERNAME"
    apply_common
    apply_setting org.gnome.desktop.interface color-scheme "'prefer-dark'"
    apply_setting org.gnome.desktop.interface enable-animations true
    apply_setting org.gnome.desktop.interface show-battery-percentage true
    apply_setting org.gnome.desktop.wm.preferences button-layout "'appmenu:minimize,maximize,close'"
    apply_setting org.gnome.desktop.interface cursor-size 24
}

# Aplicar perfil según argumento
case "$PROFILE" in
    standard)
        apply_standard
        ;;
    accessibility)
        apply_accessibility
        ;;
    admin)
        apply_admin
        ;;
    *)
        echo "ERROR: Perfil '$PROFILE' no reconocido."
        echo "Perfiles disponibles: standard, accessibility, admin"
        log_msg "ERROR: Perfil no reconocido: $PROFILE"
        exit 3
        ;;
esac

log_msg "Configuración completada exitosamente para: $USERNAME (perfil: $PROFILE)"
echo "✓ Perfil '$PROFILE' aplicado correctamente para el usuario '$USERNAME'"
exit 0
SCRIPT
```

3. Asigna permisos de ejecución:

```bash
sudo chmod 755 /opt/scripts/setup_gnome_profile.sh
```

4. Crea el archivo de log si no existe:

```bash
sudo touch /var/log/admin_scripts.log
sudo chmod 644 /var/log/admin_scripts.log
```

5. Prueba el script con un usuario de prueba:

```bash
sudo /opt/scripts/setup_gnome_profile.sh user_corp1 standard
```

6. Verifica el log:

```bash
tail -20 /var/log/admin_scripts.log
```

7. Prueba la validación de errores:

```bash
sudo /opt/scripts/setup_gnome_profile.sh usuario_inexistente standard
echo "Código de salida: $?"
```

**Salida esperada:**

```
# Ejecución exitosa
✓ Perfil 'standard' aplicado correctamente para el usuario 'user_corp1'

# Log
2024-XX-XX HH:MM:SS [GNOME-PROFILE] Aplicando perfil STANDARD para: user_corp1
2024-XX-XX HH:MM:SS [GNOME-PROFILE] OK: user_corp1 -> org.gnome.desktop.interface clock-format = '24h'
...
2024-XX-XX HH:MM:SS [GNOME-PROFILE] Configuración completada exitosamente para: user_corp1 (perfil: standard)

# Error con usuario inexistente
ERROR: El usuario 'usuario_inexistente' no existe.
Código de salida: 2
```

**Verificación:** El script debe ejecutarse sin errores para usuarios válidos, generar entradas en el log y retornar códigos de error apropiados para entradas inválidas.

---

### Paso 8: Verificar el protocolo de display activo e identificar diferencias X11/Wayland

**Objetivo:** Demostrar cómo identificar el protocolo gráfico en uso y documentar las diferencias observables entre X11 y Wayland.

**Instrucciones:**

1. Verifica qué tipo de sesión está ejecutando GDM3 actualmente:

```bash
sudo -u gdm dbus-launch gsettings get org.gnome.login-screen enable-wayland 2>/dev/null
# Alternativa:
grep -r "WaylandEnable" /etc/gdm3/custom.conf
```

2. Identifica los procesos gráficos activos:

```bash
ps aux | grep -E "Xorg|Xwayland|gnome-shell|gdm" | grep -v grep
```

3. Verifica el tipo de sesión desde la perspectiva del sistema:

```bash
loginctl list-sessions
loginctl show-session $(loginctl list-sessions | awk 'NR==2{print $1}') -p Type
```

4. Crea un script de diagnóstico para identificar el entorno gráfico:

```bash
sudo tee /opt/scripts/check_display_server.sh << 'EOF'
#!/bin/bash
#================================================================
# Script: check_display_server.sh
# Descripción: Identifica el servidor de display activo
#================================================================

echo "=== Diagnóstico del Sistema Gráfico ==="
echo ""

echo "1. Target de systemd activo:"
systemctl get-default
echo ""

echo "2. Estado de GDM3:"
systemctl is-active gdm3
echo ""

echo "3. Procesos gráficos:"
ps aux | grep -E "Xorg|Xwayland|gnome-shell|mutter" | grep -v grep | awk '{print $11, $12, $13}'
echo ""

echo "4. Sesiones activas (loginctl):"
loginctl list-sessions --no-legend
echo ""

echo "5. Configuración Wayland en GDM3:"
grep "WaylandEnable" /etc/gdm3/custom.conf 2>/dev/null || echo "Usando valor por defecto (habilitado)"
echo ""

echo "6. Sesiones de escritorio disponibles:"
echo "   X11:"
ls /usr/share/xsessions/ 2>/dev/null || echo "   Ninguna"
echo "   Wayland:"
ls /usr/share/wayland-sessions/ 2>/dev/null || echo "   Ninguna"
echo ""

echo "7. Sockets gráficos activos:"
ls /tmp/.X11-unix/ 2>/dev/null && echo "   → X11 socket presente"
ls /run/user/*/wayland-* 2>/dev/null && echo "   → Wayland socket presente"
echo ""

echo "8. Módulos DRM cargados:"
lsmod | grep drm | awk '{print "   "$1}'
echo ""

echo "=== Fin del diagnóstico ==="
EOF

sudo chmod 755 /opt/scripts/check_display_server.sh
```

5. Ejecuta el script de diagnóstico:

```bash
sudo /opt/scripts/check_display_server.sh
```

6. Documenta las diferencias observadas creando un archivo de referencia:

```bash
sudo tee /opt/sysreport/display_comparison.txt << 'EOF'
=== Comparación X11 vs Wayland en srv-linux-01 ===
Fecha: $(date)
Sistema: Ubuntu 22.04.4 LTS

CARACTERÍSTICA          | X11 (Xorg)              | Wayland
------------------------|-------------------------|---------------------------
Variable de sesión      | XDG_SESSION_TYPE=x11    | XDG_SESSION_TYPE=wayland
Variable display        | DISPLAY=:0              | WAYLAND_DISPLAY=wayland-0
Proceso principal       | /usr/lib/xorg/Xorg      | gnome-shell (compositor)
Compatibilidad legacy   | Nativa                  | Via XWayland
Captura de pantalla     | Cualquier app (xwd)     | Solo con permisos del compositor
Acceso remoto           | X11 forwarding nativo   | Requiere RDP/VNC externo
Rendimiento VirtualBox  | Estable                 | Puede tener artefactos
Archivo sesión          | /usr/share/xsessions/   | /usr/share/wayland-sessions/
EOF
```

7. Crea el directorio de reportes si no existe:

```bash
sudo mkdir -p /opt/sysreport
```

**Salida esperada:**

```
=== Diagnóstico del Sistema Gráfico ===

1. Target de systemd activo:
graphical.target

2. Estado de GDM3:
active

3. Procesos gráficos:
/usr/bin/gnome-shell
/usr/lib/xorg/Xorg :0 ...

4. Sesiones activas (loginctl):
...

5. Configuración Wayland en GDM3:
WaylandEnable=true
...
```

**Verificación:** El script debe mostrar `graphical.target` activo, GDM3 en estado `active`, y al menos un proceso gráfico (Xorg o gnome-shell) en ejecución.

---

### Paso 9: Configurar el inicio de sesión automático y opciones de GDM3

**Objetivo:** Configurar opciones avanzadas de GDM3 incluyendo la lista de usuarios, banner de bienvenida y opciones de sesión predeterminada.

**Instrucciones:**

1. Configura un banner corporativo en la pantalla de login de GDM3:

```bash
sudo tee /etc/gdm3/greeter.dconf-defaults << 'EOF'
[org/gnome/login-screen]
banner-message-enable=true
banner-message-text='Servidor srv-linux-01 - Acceso autorizado solamente. Uso no autorizado será perseguido legalmente.'
disable-user-list=false
enable-fingerprint-authentication=false
enable-smartcard-authentication=false
EOF
```

2. Actualiza la base de datos dconf para GDM3:

```bash
sudo dconf update
```

3. Configura la sesión X11 como predeterminada para entornos virtualizados (mayor estabilidad en VirtualBox):

```bash
# Crear archivo AccountsService para user_corp1 con sesión X11
sudo mkdir -p /var/lib/AccountsService/users/
sudo tee /var/lib/AccountsService/users/user_corp1 << 'EOF'
[User]
Session=ubuntu-xorg
SystemAccount=false
EOF

sudo tee /var/lib/AccountsService/users/user_access1 << 'EOF'
[User]
Session=ubuntu-xorg
SystemAccount=false
EOF

sudo tee /var/lib/AccountsService/users/user_admin1 << 'EOF'
[User]
Session=ubuntu-xorg
SystemAccount=false
EOF
```

4. Establece los permisos correctos:

```bash
sudo chmod 644 /var/lib/AccountsService/users/user_corp1
sudo chmod 644 /var/lib/AccountsService/users/user_access1
sudo chmod 644 /var/lib/AccountsService/users/user_admin1
```

5. Reinicia GDM3 para aplicar todos los cambios:

```bash
sudo systemctl restart gdm3
```

6. Verifica que el banner está configurado:

```bash
sudo -u gdm dbus-launch gsettings get org.gnome.login-screen banner-message-text 2>/dev/null
# Alternativa:
grep "banner-message" /etc/gdm3/greeter.dconf-defaults
```

**Salida esperada:**

```
# Verificación del banner
banner-message-text='Servidor srv-linux-01 - Acceso autorizado solamente...'

# Archivos AccountsService
# cat /var/lib/AccountsService/users/user_corp1
[User]
Session=ubuntu-xorg
SystemAccount=false
```

**Verificación:** El banner debe estar configurado en el archivo de defaults del greeter, y cada usuario debe tener su archivo en AccountsService con la sesión X11 asignada.

---

### Paso 10: Instalar VirtualBox Guest Additions para aceleración gráfica

**Objetivo:** Instalar las Guest Additions de VirtualBox para mejorar el rendimiento gráfico y habilitar funcionalidades como resolución dinámica.

**Instrucciones:**

1. Instala las dependencias necesarias para compilar las Guest Additions:

```bash
sudo apt install -y build-essential dkms linux-headers-$(uname -r)
```

2. Instala las Guest Additions desde los repositorios de Ubuntu (método más estable):

```bash
sudo apt install -y virtualbox-guest-x11 virtualbox-guest-utils virtualbox-guest-dkms
```

3. Verifica que los módulos de VirtualBox están cargados:

```bash
lsmod | grep vbox
```

4. Verifica que el servicio de Guest Additions está activo:

```bash
systemctl status vboxadd-service --no-pager 2>/dev/null || \
systemctl status virtualbox-guest-utils --no-pager 2>/dev/null
```

5. Agrega los usuarios al grupo `vboxsf` para acceso a carpetas compartidas (opcional):

```bash
sudo usermod -aG vboxsf user_corp1
sudo usermod -aG vboxsf user_access1
sudo usermod -aG vboxsf user_admin1
```

6. Reinicia el sistema para aplicar todos los cambios:

```bash
sudo reboot
```

7. Después del reinicio, verifica que el sistema arranca en modo gráfico:

```bash
# Reconectar por SSH después del reinicio
ssh sysadmin@192.168.100.10

# Verificar
systemctl get-default
systemctl is-active gdm3
loginctl list-sessions
```

**Salida esperada:**

```
# lsmod | grep vbox
vboxguest             ...
vboxvideo             ...

# Después del reinicio
graphical.target
active
SESSION  UID USER     SEAT  TTY
      1  125 gdm      seat0 tty1
```

**Verificación:** Los módulos `vboxguest` y `vboxvideo` deben estar cargados, y después del reinicio el sistema debe arrancar directamente en modo gráfico con GDM3 mostrando la pantalla de login.

---

## Validación y Pruebas

Ejecuta la siguiente secuencia de verificaciones para confirmar que todo el laboratorio se completó correctamente:

```bash
#!/bin/bash
echo "=== VALIDACIÓN COMPLETA DEL LAB 08-00-01 ==="
echo ""

# Test 1: Target gráfico
echo -n "1. Target graphical.target activo: "
[ "$(systemctl get-default)" = "graphical.target" ] && echo "✓ PASS" || echo "✗ FAIL"

# Test 2: GDM3 activo
echo -n "2. GDM3 en ejecución: "
systemctl is-active gdm3 &>/dev/null && echo "✓ PASS" || echo "✗ FAIL"

# Test 3: Usuarios creados
echo -n "3. user_corp1 existe: "
id user_corp1 &>/dev/null && echo "✓ PASS" || echo "✗ FAIL"

echo -n "4. user_access1 existe: "
id user_access1 &>/dev/null && echo "✓ PASS" || echo "✗ FAIL"

echo -n "5. user_admin1 existe (con sudo): "
groups user_admin1 | grep -q sudo && echo "✓ PASS" || echo "✗ FAIL"

# Test 4: Sesiones disponibles
echo -n "6. Sesión X11 disponible: "
[ -f /usr/share/xsessions/ubuntu-xorg.desktop ] && echo "✓ PASS" || echo "✗ FAIL"

echo -n "7. Sesión Wayland disponible: "
ls /usr/share/wayland-sessions/*.desktop &>/dev/null && echo "✓ PASS" || echo "✗ FAIL"

# Test 5: Script de configuración
echo -n "8. Script setup_gnome_profile.sh existe y es ejecutable: "
[ -x /opt/scripts/setup_gnome_profile.sh ] && echo "✓ PASS" || echo "✗ FAIL"

# Test 6: Script de diagnóstico
echo -n "9. Script check_display_server.sh existe y es ejecutable: "
[ -x /opt/scripts/check_display_server.sh ] && echo "✓ PASS" || echo "✗ FAIL"

# Test 7: Políticas dconf
echo -n "10. Base de datos dconf actualizada: "
[ -f /etc/dconf/db/local.d/00-corporate-policy ] && echo "✓ PASS" || echo "✗ FAIL"

# Test 8: Locks de dconf
echo -n "11. Locks corporativos configurados: "
[ -f /etc/dconf/db/local.d/locks/00-corporate-locks ] && echo "✓ PASS" || echo "✗ FAIL"

# Test 9: Banner GDM3
echo -n "12. Banner corporativo configurado: "
grep -q "banner-message-enable=true" /etc/gdm3/greeter.dconf-defaults 2>/dev/null && echo "✓ PASS" || echo "✗ FAIL"

# Test 10: Configuración de usuario
echo -n "13. Configuración gsettings user_access1 (text-scaling): "
SCALE=$(sudo -u user_access1 dbus-launch gsettings get org.gnome.desktop.interface text-scaling-factor 2>/dev/null)
[ "$SCALE" = "1.5" ] && echo "✓ PASS" || echo "✗ FAIL (valor: $SCALE)"

echo ""
echo "=== FIN DE VALIDACIÓN ==="
```

Guarda y ejecuta este script:

```bash
sudo tee /opt/scripts/validate_lab08.sh << 'HEREDOC'
# [Pegar el contenido anterior]
HEREDOC
sudo chmod +x /opt/scripts/validate_lab08.sh
sudo /opt/scripts/validate_lab08.sh
```

**Resultado esperado:** Todos los tests deben mostrar `✓ PASS`.

---

## Solución de Problemas

### Problema 1: GDM3 no inicia y el sistema queda en pantalla negra

**Síntomas:**
- Después de ejecutar `systemctl isolate graphical.target` o reiniciar, la pantalla queda negra o muestra solo un cursor parpadeante.
- El servicio GDM3 aparece en estado `failed` o en ciclo de reinicio.
- `journalctl -u gdm3` muestra errores relacionados con el driver gráfico.

**Causa:**
La aceleración gráfica de VirtualBox no está correctamente configurada, o el controlador gráfico seleccionado (VBoxVGA vs VMSVGA) es incompatible con la versión de Guest Additions instalada. En Ubuntu 22.04, VMSVGA es el controlador recomendado.

**Solución:**

```bash
# 1. Acceder por SSH (la sesión SSH sigue funcional)
ssh sysadmin@192.168.100.10

# 2. Verificar los logs de GDM3
journalctl -u gdm3 --no-pager -n 50

# 3. Verificar logs de Xorg
cat /var/log/Xorg.0.log | grep -E "EE|WW" | head -20

# 4. Volver temporalmente al target texto
sudo systemctl isolate multi-user.target

# 5. En VirtualBox: apagar la VM y verificar la configuración
# Settings → Display → Graphics Controller: VMSVGA
# Settings → Display → Video Memory: 128 MB
# Settings → Display → Enable 3D Acceleration: ✓

# 6. Reiniciar y verificar
sudo reboot

# 7. Si persiste, forzar Xorg en lugar de Wayland:
sudo sed -i 's/^#WaylandEnable=false/WaylandEnable=false/' /etc/gdm3/custom.conf
sudo systemctl restart gdm3
```

---

### Problema 2: gsettings falla con "No schemas installed" o "dbus-launch" no conecta

**Síntomas:**
- Al ejecutar `sudo -u user_corp1 dbus-launch gsettings set ...` aparece el error:
  ```
  No schemas installed
  ```
  o
  ```
  GLib-GIO-ERROR: Settings schema 'org.gnome.desktop.interface' is not installed
  ```
- También puede aparecer: `Failed to connect to bus: No such file or directory`

**Causa:**
Los esquemas de GNOME no están disponibles para el usuario porque la variable `XDG_DATA_DIRS` no incluye la ruta correcta, o el servicio dbus del usuario no está accesible desde la sesión sudo. Esto ocurre frecuentemente cuando se ejecutan comandos gsettings desde una sesión SSH sin entorno gráfico activo.

**Solución:**

```bash
# 1. Verificar que los esquemas están instalados
find / -name "org.gnome.desktop.interface.gschema.xml" 2>/dev/null

# 2. Recompilar los esquemas
sudo glib-compile-schemas /usr/share/glib-2.0/schemas/

# 3. Usar la variable XDG_DATA_DIRS explícitamente
sudo -u user_corp1 env XDG_DATA_DIRS=/usr/share/gnome:/usr/local/share:/usr/share \
    dbus-launch gsettings set org.gnome.desktop.interface clock-format '24h'

# 4. Alternativa: usar dconf directamente (no requiere esquemas compilados)
sudo -u user_corp1 dbus-launch dconf write /org/gnome/desktop/interface/clock-format "'24h'"

# 5. Si dbus-launch falla completamente, crear el archivo dconf manualmente:
sudo mkdir -p /home/user_corp1/.config/dconf
sudo -u user_corp1 dbus-launch dconf dump / > /dev/null 2>&1

# 6. Verificar que el paquete de esquemas está instalado
dpkg -l | grep gsettings-desktop-schemas
# Si no está:
sudo apt install -y gsettings-desktop-schemas
```

---

## Limpieza

Si necesitas revertir los cambios de este laboratorio (por ejemplo, para volver al modo servidor sin entorno gráfico):

```bash
# Volver al target multi-user (modo texto)
sudo systemctl set-default multi-user.target

# Detener GDM3
sudo systemctl stop gdm3
sudo systemctl disable gdm3

# OPCIONAL: Eliminar completamente el entorno gráfico (libera ~2 GB)
# PRECAUCIÓN: Solo ejecutar si se desea eliminar GNOME completamente
# sudo apt remove --purge -y ubuntu-desktop-minimal gdm3 gnome-shell
# sudo apt autoremove --purge -y

# Los usuarios creados NO se eliminan (serán utilizados en Lab 09)
# Los scripts en /opt/scripts/ se mantienen para uso futuro
```

> **Nota importante:** No elimines los usuarios `user_corp1`, `user_access1` ni `user_admin1`, ya que serán administrados en el Lab 09. Tampoco elimines los scripts creados en `/opt/scripts/`.

---

## Resumen

En este laboratorio has completado las siguientes tareas de administración de estaciones de trabajo Linux:

| Tarea | Resultado |
|-------|-----------|
| Instalación de GNOME 42.9 | `ubuntu-desktop-minimal` instalado con todas las dependencias |
| Configuración de systemd | Target cambiado a `graphical.target` |
| Administración de GDM3 | Habilitado, con banner corporativo y sesiones X11/Wayland |
| Comparación X11/Wayland | Ambas sesiones disponibles, X11 configurada por defecto para VirtualBox |
| Usuarios corporativos | 3 usuarios creados con perfiles diferenciados |
| Políticas dconf | Bloqueo de pantalla a 10 min, locks en configuraciones críticas |
| Automatización | Script `setup_gnome_profile.sh` funcional y documentado |
| Guest Additions | Instaladas para rendimiento gráfico óptimo |

### Conceptos clave reforzados

- La separación entre el sistema gráfico y el kernel Linux permite instalar/desinstalar entornos de escritorio sin afectar la funcionalidad del servidor.
- GDM3 actúa como puerta de entrada al entorno gráfico, controlando la autenticación y la selección de sesión.
- dconf/gsettings proporcionan un mecanismo centralizado para gestionar configuraciones de GNOME a nivel de usuario y de sistema.
- La elección entre X11 y Wayland depende del caso de uso: X11 para compatibilidad y acceso remoto, Wayland para seguridad y rendimiento.

### Recursos adicionales

- [Documentación de GNOME System Administration Guide](https://help.gnome.org/admin/system-admin-guide/stable/)
- [Manual de dconf (freedesktop.org)](https://wiki.gnome.org/Projects/dconf/SystemAdministrators)
- [Ubuntu Wiki: GNOME Shell](https://wiki.ubuntu.com/GNOME)
- [VirtualBox Guest Additions Documentation](https://www.virtualbox.org/manual/ch04.html)

---
