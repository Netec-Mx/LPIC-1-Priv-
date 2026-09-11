# Restaurar la operación de un servidor Linux que presenta fallas de arranque.

## 1. Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 72 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Máquina(s)** | srv-linux-01 (Ubuntu Server 22.04.4 LTS o superior) |
| **IP** | 192.168.10.100 |
| **Password de root en Azure** | Raiz1234 |

## 2. Descripción General

En este laboratorio simularás tres escenarios de falla de arranque en un servidor Linux de producción y los resolverás aplicando técnicas profesionales de recuperación. Trabajarás con modificaciones incorrectas en GRUB2, recuperación de contraseña de root mediante `rd.break`, y reinstalación completa del gestor de arranque desde un entorno de rescate. Cada escenario representa una situación real que un administrador de sistemas enfrenta en entornos empresariales.

Nota importante:  Este laboratorio no es realizable en Azure, solo podra hacerse en ambientes de hardware real o virtualizacion local de Linux usando Virtualbox o VMWare Workstation.

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Diagnosticar y resolver fallas de arranque causadas por parámetros incorrectos del kernel en GRUB2
- [ ] Recuperar el acceso a un sistema con contraseña de root perdida utilizando el modo single-user y `rd.break`
- [ ] Reparar una configuración de GRUB2 dañada reinstalando y regenerando el gestor de arranque
- [ ] Administrar targets de systemd para controlar el nivel de ejecución durante procedimientos de recuperación
- [ ] Documentar procedimientos de recuperación de forma estructurada para referencia futura

## 4. Prerrequisitos

### Conocimiento previo

- Comprensión de las seis fases de la secuencia de arranque de Linux (firmware, cargador de primera etapa, GRUB2, kernel, initramfs, systemd)
- Familiaridad con la edición de archivos en terminal (nano o vi)
- Conocimiento básico de systemd y systemctl
- Lab 01 completado con reporte inicial generado

### Acceso requerido

- VM srv-linux-01 operativa con Ubuntu Server 22.04.4 LTS o superior
- Acceso a la consola de VirtualBox (no SSH — se trabajará sin red en varios escenarios)
- Imagen ISO de Ubuntu Server 22.04.4 LTS o superior disponible para el modo rescate
- Usuario `root` con contraseña `Linux@Admin2024!`

## 5. Entorno del Laboratorio

### Configuración de hardware virtual

| Recurso | Configuración |
|---------|---------------|
| VM | srv-linux-01 |
| CPU | 2 vCPU mínimo |
| RAM | 2048 MB |
| Disco | /dev/sda 40 GB (sistema) |
| Red | Adaptador 1: Red interna `syslab-network` / Adaptador 2: NAT |
| ISO | Ubuntu Server 22.04.4 LTS o superior o superior (montada en unidad óptica virtual) |

### Preparación inicial (Solo en Virtualbox si se usa en los laboratorios)

**CRÍTICO: Antes de iniciar cualquier escenario, crea un snapshot de la VM.**

```bash
# Desde el HOST (no dentro de la VM) — en la terminal del hipervisor
VBoxManage snapshot "srv-linux-01" take "pre-lab02-clean" \
  --description "Estado limpio antes del lab 02 de recuperación de arranque"
```

Verifica que el snapshot se creó correctamente:

```bash
VBoxManage snapshot "srv-linux-01" list
```

**Salida esperada:**
```
Name: pre-lab02-clean (UUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
   This is the current snapshot
```

Ahora inicia sesión en srv-linux-01 y prepara el directorio de documentación:

```bash
# Iniciar sesión como root
sudo mkdir -p /opt/sysreport
sudo chown root:root /opt/sysreport
```

Registra el estado actual del sistema como línea base:

```bash
# Guardar información de referencia del arranque actual
systemd-analyze > /opt/sysreport/boot_baseline.txt
systemd-analyze blame | head -20 >> /opt/sysreport/boot_baseline.txt
cat /etc/default/grub >> /opt/sysreport/boot_baseline.txt
echo "--- Fecha de referencia: $(date) ---" >> /opt/sysreport/boot_baseline.txt
```

---

## 6. Procedimiento Paso a Paso

---

### Escenario 1: Parámetros incorrectos del kernel en GRUB2 (20 minutos)

---

#### Paso 1.1: Simular la falla — Modificar parámetros del kernel

**Objetivo:** Introducir un parámetro inválido en la configuración de GRUB2 que impida el arranque normal del sistema.

**Instrucciones:**

1. Inicia sesión en srv-linux-01 como `root`:

```bash
ssh root@192.168.10.100
# O accede directamente por consola de VirtualBox
```

2. Realiza una copia de seguridad del archivo de configuración de GRUB:

```bash
sudo cp /etc/default/grub /etc/default/grub.backup
sudo cp /boot/grub/grub.cfg /boot/grub/grub.cfg.backup
```

3. Modifica el archivo `/etc/default/grub` para introducir un parámetro de root inválido:

```bash
sudo nano /etc/default/grub
```

4. Localiza la línea `GRUB_CMDLINE_LINUX_DEFAULT` y modifícala:

```
# ANTES (valor original):
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

# DESPUÉS (valor con error - raíz inexistente):
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash root=/dev/sda99"
```

5. Adicionalmente, modifica `GRUB_CMDLINE_LINUX` para agregar un parámetro inválido:

```
# ANTES:
GRUB_CMDLINE_LINUX=""

# DESPUÉS:
GRUB_CMDLINE_LINUX="init=/bin/falso"
```

6. Regenera la configuración de GRUB con los parámetros incorrectos:

```bash
sudo update-grub
```

**Salida esperada:**
```
Sourcing file `/etc/default/grub'
Sourcing file `/etc/default/grub.d/init-select.cfg'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.15.0-xxx-generic
Found initrd image: /boot/initrd.img-5.15.0-xxx-generic
done
```

7. Reinicia el sistema para que la falla se manifieste:

```bash
sudo reboot
```

**Verificación:** El sistema NO arrancará correctamente. Verás un mensaje de pánico del kernel o quedará en una pantalla de `initramfs` con un prompt de emergencia similar a:

```
(initramfs) _
```

O bien un mensaje como:
```
Kernel panic - not syncing: No working init found.
```

---

#### Paso 1.2: Diagnosticar la falla desde el menú de GRUB

**Objetivo:** Identificar los parámetros incorrectos accediendo al editor de GRUB durante el arranque.

**Instrucciones:**

1. Reinicia la VM (si está colgada, usa el menú de VirtualBox: Máquina → Reiniciar).

2. Durante el arranque, **mantén presionada la tecla `Shift`** (en sistemas BIOS) o **presiona `Esc`** repetidamente para acceder al menú de GRUB.

> **Nota:** En Ubuntu con UEFI, presiona `Esc` inmediatamente después del logo del firmware.

3. Una vez visible el menú de GRUB, selecciona la entrada principal de Ubuntu y presiona **`e`** para editar los parámetros de arranque.

4. Localiza la línea que comienza con `linux` (la línea del kernel). Observarás algo similar a:

```
linux   /boot/vmlinuz-5.15.0-xxx-generic root=/dev/sda99 ro quiet splash init=/bin/falso
```

5. **Identifica los errores:**
   - `root=/dev/sda99` — dispositivo inexistente (debería ser algo como `root=UUID=xxxx` o `root=/dev/sda2`)
   - `init=/bin/falso` — ejecutable de init inexistente

**Verificación:** Has identificado visualmente los dos parámetros que causan la falla de arranque.

---

#### Paso 1.3: Resolver la falla editando GRUB temporalmente

**Objetivo:** Corregir los parámetros del kernel en tiempo de arranque para restaurar el acceso al sistema.

**Instrucciones:**

1. Estando en la pantalla de edición de GRUB (tras presionar `e`), navega con las flechas hasta la línea `linux`.

2. Elimina el parámetro `root=/dev/sda99` y `init=/bin/falso` de la línea. La línea corregida debe verse similar a:

```
linux   /boot/vmlinuz-5.15.0-xxx-generic root=UUID=<tu-uuid-real> ro quiet splash
```

> **Tip:** Si no recuerdas el UUID correcto, puedes eliminar solo los parámetros incorrectos y dejar que GRUB use el root definido en la configuración original. Alternativamente, busca la línea que contenga `set root=` más arriba en la configuración.

3. Si no conoces el UUID, simplemente elimina `root=/dev/sda99` e `init=/bin/falso`, dejando la línea como:

```
linux   /boot/vmlinuz-5.15.0-xxx-generic ro quiet splash
```

4. Presiona **`Ctrl+X`** o **`F10`** para arrancar con los parámetros corregidos.

**Salida esperada:** El sistema arranca normalmente y presenta el prompt de inicio de sesión:

```
srv-linux-01 login: _
```

5. Inicia sesión como `root` y verifica el arranque:

```bash
systemd-analyze
```

**Salida esperada:**
```
Startup finished in XXXms (kernel) + XXXs (initrd) + XXXs (userspace) = XXXs
multi-user.target reached after XXXs in userspace
```

---

#### Paso 1.4: Corregir la falla de forma permanente

**Objetivo:** Restaurar la configuración correcta de GRUB2 para que el sistema arranque sin intervención manual.

**Instrucciones:**

1. Restaura el archivo de configuración original:

```bash
sudo cp /etc/default/grub.backup /etc/default/grub
```

2. Verifica el contenido restaurado:

```bash
cat /etc/default/grub | grep CMDLINE
```

**Salida esperada:**
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX=""
```

3. Regenera la configuración de GRUB:

```bash
sudo update-grub
```

**Salida esperada:**
```
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.15.0-xxx-generic
Found initrd image: /boot/initrd.img-5.15.0-xxx-generic
done
```

4. Verifica que `grub.cfg` ya no contiene los parámetros erróneos:

```bash
grep -c "sda99\|falso" /boot/grub/grub.cfg
```

**Salida esperada:**
```
0
```

5. Reinicia para confirmar que el arranque es limpio:

```bash
sudo reboot
```

**Verificación:** El sistema arranca sin intervención manual y llega al prompt de login.

---

### Escenario 2: Recuperación de contraseña de root (22 minutos)

---

#### Paso 2.1: Simular la pérdida de contraseña de root

**Objetivo:** Configurar un escenario donde la contraseña de root es desconocida y se requiere restablecerla.

**Instrucciones:**

1. Inicia sesión como `root` en srv-linux-01.

2. Cambia la contraseña de root a un valor "desconocido" (simula que alguien la cambió sin documentar):

```bash
# Cambiar a una contraseña que "olvidaremos"
echo "root:ContraseñaOlvidada123!" | sudo chpasswd
```

3. Para hacer el escenario más realista, bloquea también la cuenta de root temporalmente (simula que no puedes hacer sudo):

```bash
# IMPORTANTE: Esto te dejará sin acceso administrativo
sudo passwd -l root
```

4. Cierra la sesión:

```bash
exit
```

5. Intenta iniciar sesión como root con la contraseña documentada (`R00t@Linux2024!`):

```
srv-linux-01 login: root
Password: R00t@Linux2024!
```

**Salida esperada:**
```
Login incorrect
```

6. Intenta iniciar sesión como root:

```
srv-linux-01 login: root
Password: Linux@Admin2024!
```

**Salida esperada:**
```
Authentication failure
```

**Verificación:** No es posible acceder al sistema con las credenciales conocidas. Se requiere un procedimiento de recuperación.

---

#### Paso 2.2: Acceder al modo single-user mediante edición de GRUB

**Objetivo:** Utilizar parámetros de arranque del kernel para acceder al sistema sin contraseña.

**Instrucciones:**

1. Reinicia la VM (desde la consola de VirtualBox: Máquina → Reiniciar, o presiona `Ctrl+Alt+Del` en la consola).

2. Mantén presionada la tecla **`Shift`** durante el arranque para acceder al menú de GRUB.

3. Selecciona la entrada principal de Ubuntu y presiona **`e`** para editar.

4. Localiza la línea que comienza con `linux` y realiza las siguientes modificaciones:

   a. Elimina los parámetros `quiet` y `splash` (para ver mensajes de arranque).
   
   b. Añade `rd.break` al final de la línea para interrumpir el arranque antes de montar el sistema de archivos raíz:

```
linux   /boot/vmlinuz-5.15.0-xxx-generic root=UUID=xxxx ro rd.break
```

> **Nota:** En Ubuntu Server 22.04, si `rd.break` no funciona directamente, usa el método alternativo con `init=/bin/bash` que se describe a continuación.

5. **Método alternativo (más confiable en Ubuntu):** En lugar de `rd.break`, reemplaza `ro` por `rw` y añade `init=/bin/bash` al final:

```
linux   /boot/vmlinuz-5.15.0-xxx-generic root=UUID=xxxx rw init=/bin/bash
```

6. Presiona **`Ctrl+X`** o **`F10`** para arrancar.

**Salida esperada con `init=/bin/bash`:**
```
root@(none):/# _
```

El sistema arranca directamente a un shell de root sin solicitar contraseña.

**Salida esperada con `rd.break`:**
```
switch_root:/# _
```

---

#### Paso 2.3: Restablecer la contraseña de root

**Objetivo:** Cambiar la contraseña de root y restaurar el acceso administrativo al sistema.

**Instrucciones:**

**Si usaste `init=/bin/bash`:**

1. Verifica que el sistema de archivos está montado con permisos de escritura:

```bash
mount | grep "on / "
```

**Salida esperada (debe incluir `rw`):**
```
/dev/sda2 on / type ext4 (rw,relatime)
```

2. Si el sistema de archivos está en modo solo lectura (`ro`), remóntalo:

```bash
mount -o remount,rw /
```

3. Restablece la contraseña de root:

```bash
passwd root
```

Introduce la nueva contraseña: `R00t@Linux2024!`

**Salida esperada:**
```
New password: 
Retype new password: 
passwd: password updated successfully
```

4. Desbloquea la cuenta de root:

```bash
passwd -u root
```

**Salida esperada:**
```
passwd: password expiry information changed.
```

5. Asegúrate de que SELinux (si aplica) relabele los archivos. En Ubuntu no es necesario, pero es buena práctica sincronizar:

```bash
sync
```

6. Reinicia el sistema de forma segura:

```bash
# Con init=/bin/bash no hay systemd, usa:
exec /sbin/init
```

O si lo anterior no funciona:

```bash
# Forzar reinicio (último recurso)
echo b > /proc/sysrq-trigger
```

**Si usaste `rd.break`:**

1. Monta el sistema de archivos raíz real:

```bash
mount -o remount,rw /sysroot
```

2. Cambia el entorno raíz con `chroot`:

```bash
chroot /sysroot
```

3. Restablece la contraseña de root:

```bash
passwd root
# Introduce: R00t@Linux2024!
```

4. Desbloquea root:

```bash
passwd -u root
```

5. En sistemas con SELinux (Rocky Linux), crea el archivo de relabeling:

```bash
touch /.autorelabel
```

6. Sal del chroot y reinicia:

```bash
exit
exit
# O: reboot -f
```

---

#### Paso 2.4: Verificar la recuperación de acceso

**Objetivo:** Confirmar que las credenciales restablecidas funcionan correctamente.

**Instrucciones:**

1. Una vez reiniciado el sistema, inicia sesión como root:

```
srv-linux-01 login: root
Password: R00t@Linux2024!
```

**Salida esperada:**
```
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-xxx-generic x86_64)
root@srv-linux-01:~#
```

2. Verifica que root puede hacer login y sudo:

```bash
su - root
sudo whoami
```

**Salida esperada:**
```
[sudo] password for root: 
root
```

3. Verifica el estado general del sistema:

```bash
systemctl is-system-running
```

**Salida esperada:**
```
running
```

4. Revisa los logs para confirmar que no hay errores residuales:

```bash
journalctl -b --priority=err --no-pager | head -20
```

**Verificación:** La cuenta root es accesible con su contraseña documentada.

---

### Escenario 3: Reparación de GRUB2 dañado desde entorno de rescate (25 minutos)

---

#### Paso 3.1: Simular la corrupción de GRUB2

**Objetivo:** Dañar intencionalmente el archivo `grub.cfg` y el sector de arranque para simular una corrupción real.

**Instrucciones:**

1. Inicia sesión como `root` en srv-linux-01.

2. Primero, verifica la ubicación actual de GRUB:

```bash
sudo grub-install --version
sudo fdisk -l /dev/sda | head -15
```

3. Corrompe el archivo de configuración de GRUB:

```bash
# Crear backup adicional antes de destruir
sudo cp /boot/grub/grub.cfg /opt/sysreport/grub.cfg.pre-corruption

# Corromper el archivo grub.cfg
sudo bash -c 'echo "ARCHIVO CORRUPTO - SIMULACION DE FALLA" > /boot/grub/grub.cfg'
```

4. Verifica la corrupción:

```bash
cat /boot/grub/grub.cfg
```

**Salida esperada:**
```
ARCHIVO CORRUPTO - SIMULACION DE FALLA
```

5. Para hacer el escenario más desafiante, borra también los módulos auxiliares de GRUB:

```bash
sudo rm -rf /boot/grub/i386-pc/
# O en UEFI: sudo rm -rf /boot/grub/x86_64-efi/
```

6. Reinicia el sistema:

```bash
sudo reboot
```

**Salida esperada:** El sistema NO arrancará. Verás uno de estos mensajes:

```
error: file '/boot/grub/i386-pc/normal.mod' not found.
grub rescue> _
```

O directamente:

```
GRUB  error: no such device: xxxx
grub rescue> _
```

**Verificación:** El sistema queda en el prompt `grub rescue>` o `grub>`, indicando que GRUB no puede cargar su configuración.

---

#### Paso 3.2: Intentar arranque manual desde GRUB rescue

**Objetivo:** Demostrar las capacidades limitadas del modo rescue de GRUB y los comandos básicos disponibles.

**Instrucciones:**

1. Desde el prompt `grub rescue>`, lista los dispositivos disponibles:

```
grub rescue> ls
```

**Salida esperada:**
```
(hd0) (hd0,msdos2) (hd0,msdos1)
```

O en sistemas GPT/UEFI:
```
(hd0) (hd0,gpt3) (hd0,gpt2) (hd0,gpt1)
```

2. Intenta identificar la partición que contiene `/boot`:

```
grub rescue> ls (hd0,msdos1)/
grub rescue> ls (hd0,msdos2)/boot/
```

3. Si encuentras la partición con `/boot`, intenta cargar los módulos manualmente:

```
grub rescue> set prefix=(hd0,msdos2)/boot/grub
grub rescue> insmod normal
```

**Salida esperada (con módulos borrados):**
```
error: file '/boot/grub/i386-pc/normal.mod' not found.
```

> **Conclusión:** Sin los módulos de GRUB, no es posible arrancar desde el modo rescue. Se requiere un medio externo de rescate.

---

#### Paso 3.3: Arrancar desde ISO de Ubuntu (modo rescate)

**Objetivo:** Utilizar la ISO de instalación de Ubuntu como medio de rescate para acceder al sistema.

**Instrucciones:**

1. En VirtualBox, accede a la configuración de la VM:
   - Configuración → Almacenamiento → Controlador IDE
   - Añade la ISO de Ubuntu Server 22.04.4 LTS o superior como disco óptico
   - Configuración → Sistema → Orden de arranque: coloca "Óptico" antes de "Disco duro"

2. Inicia la VM. Debería arrancar desde la ISO.

3. En el menú de instalación de Ubuntu, selecciona el idioma y luego busca la opción **"Rescue a broken system"** o **"Boot and Install"**.

> **Nota para Ubuntu Server 22.04:** La ISO moderna usa Subiquity y no tiene un modo rescate clásico. Usa el siguiente método alternativo:

**Método alternativo — Shell desde el instalador:**

4. Cuando aparezca el instalador de Ubuntu Server (Subiquity), presiona **`Ctrl+Alt+F2`** para cambiar a una consola TTY alternativa.

5. Obtendrás un shell con acceso al sistema:

```bash
# Verificar que estás en el entorno live
whoami
```

**Salida esperada:**
```
root
```

**Método alternativo 2 — Usar opción "Try or Install Ubuntu" con shell:**

Si usas la ISO Desktop, selecciona "Try Ubuntu" y abre una terminal.

---

#### Paso 3.4: Montar el sistema y reinstalar GRUB2

**Objetivo:** Montar el sistema de archivos raíz del disco duro y reinstalar GRUB2 completamente.

**Instrucciones:**

1. Identifica las particiones del disco duro:

```bash
lsblk
fdisk -l /dev/sda
```

**Salida esperada (ejemplo):**
```
Device     Boot   Start      End  Sectors  Size Id Type
/dev/sda1  *       2048  2099199  2097152    1G 83 Linux
/dev/sda2       2099200 83886079 81786880   39G 83 Linux
```

2. Identifica la partición raíz (la más grande, generalmente `/dev/sda2`):

```bash
blkid /dev/sda2
```

**Salida esperada:**
```
/dev/sda2: UUID="xxxx-xxxx" TYPE="ext4"
```

3. Monta el sistema de archivos raíz:

```bash
mount /dev/sda2 /mnt
```

4. Verifica que el montaje es correcto:

```bash
ls /mnt/etc/hostname
cat /mnt/etc/hostname
```

**Salida esperada:**
```
srv-linux-01
```

5. Monta la partición de boot si es separada:

```bash
# Solo si /boot está en partición separada
mount /dev/sda1 /mnt/boot
```

6. Monta los sistemas de archivos virtuales necesarios para chroot:

```bash
mount --bind /dev /mnt/dev
mount --bind /dev/pts /mnt/dev/pts
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
mount --bind /run /mnt/run
```

7. Entra al entorno chroot:

```bash
chroot /mnt
```

**Salida esperada:**
```
root@(none):/# 
```

8. Verifica que estás dentro del sistema correcto:

```bash
cat /etc/os-release | head -3
```

**Salida esperada:**
```
PRETTY_NAME="Ubuntu 22.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
```

---

#### Paso 3.5: Reinstalar GRUB2 y regenerar configuración

**Objetivo:** Reinstalar el gestor de arranque en el disco y regenerar `grub.cfg`.

**Instrucciones:**

1. Reinstala los paquetes de GRUB (esto restaura los módulos borrados):

```bash
apt-get install --reinstall grub-pc grub-common grub2-common
```

Cuando pregunte en qué disco instalar GRUB, selecciona `/dev/sda`.

**Salida esperada:**
```
Setting up grub-pc (2.06-2ubuntu7) ...
Installing for i386-pc platform.
Installation finished. No error reported.
```

2. Si el paso anterior no solicita el disco, instala GRUB manualmente:

```bash
grub-install /dev/sda
```

**Salida esperada:**
```
Installing for i386-pc platform.
Installation finished. No error reported.
```

3. Para sistemas UEFI, el comando sería:

```bash
# Solo para UEFI (montar la ESP primero):
# mount /dev/sda1 /boot/efi
# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
```

4. Regenera el archivo de configuración `grub.cfg`:

```bash
update-grub
```

**Salida esperada:**
```
Sourcing file `/etc/default/grub'
Sourcing file `/etc/default/grub.d/init-select.cfg'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.15.0-xxx-generic
Found initrd image: /boot/initrd.img-5.15.0-xxx-generic
done
```

5. Verifica que el archivo `grub.cfg` se generó correctamente:

```bash
head -20 /boot/grub/grub.cfg
```

**Salida esperada (primeras líneas):**
```
#
# DO NOT EDIT THIS FILE
#
# It is automatically generated by grub-mkconfig using templates
# from /etc/grub.d and settings from /etc/default/grub
#
```

6. Verifica que los módulos de GRUB están restaurados:

```bash
ls /boot/grub/i386-pc/ | wc -l
```

**Salida esperada (debe ser un número mayor a 200):**
```
267
```

7. Sal del chroot y desmonta todo:

```bash
exit
umount /mnt/dev/pts
umount /mnt/dev
umount /mnt/proc
umount /mnt/sys
umount /mnt/run
umount /mnt/boot  # Solo si se montó separadamente
umount /mnt
```

8. Retira la ISO del orden de arranque:
   - En VirtualBox: Dispositivos → Unidades ópticas → Quitar disco de la unidad virtual
   - O cambia el orden de arranque: Sistema → Disco duro primero

9. Reinicia:

```bash
reboot
```

**Verificación:** El sistema arranca normalmente mostrando el menú de GRUB y luego el prompt de login.

---

#### Paso 3.6: Verificar la reparación y explorar targets de systemd

**Objetivo:** Confirmar el arranque exitoso y explorar los targets de systemd como parte del procedimiento de recuperación.

**Instrucciones:**

1. Inicia sesión como `root`.

2. Verifica el target actual del sistema:

```bash
systemctl get-default
```

**Salida esperada:**
```
multi-user.target
```

3. Lista todos los targets disponibles:

```bash
systemctl list-units --type=target --all
```

**Salida esperada (parcial):**
```
UNIT                   LOAD   ACTIVE   SUB    DESCRIPTION
basic.target           loaded active   active Basic System
emergency.target       loaded inactive dead   Emergency Mode
graphical.target       loaded inactive dead   Graphical Interface
multi-user.target      loaded active   active Multi-User System
rescue.target          loaded inactive dead   Rescue Mode
```

4. Verifica la equivalencia entre targets y niveles de ejecución:

```bash
# Rescue mode (equivalente a runlevel 1 / single-user)
ls -la /lib/systemd/system/runlevel1.target

# Multi-user (equivalente a runlevel 3)
ls -la /lib/systemd/system/runlevel3.target

# Graphical (equivalente a runlevel 5)
ls -la /lib/systemd/system/runlevel5.target
```

**Salida esperada:**
```
lrwxrwxrwx 1 root root 13 ... /lib/systemd/system/runlevel1.target -> rescue.target
lrwxrwxrwx 1 root root 17 ... /lib/systemd/system/runlevel3.target -> multi-user.target
lrwxrwxrwx 1 root root 18 ... /lib/systemd/system/runlevel5.target -> graphical.target
```

5. Practica el cambio de target en caliente (sin reiniciar):

```bash
# Cambiar a rescue target (pedirá contraseña de root)
sudo systemctl isolate rescue.target
```

**Salida esperada:** La pantalla se limpia y muestra:

```
You are in rescue mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" to try again
to boot into default mode.
Give root password for maintenance
(or press Control-D to continue): 
```

6. Introduce la contraseña de root (`R00t@Linux2024!`) y explora el entorno de rescate:

```bash
# Verificar servicios activos en rescue mode
systemctl list-units --type=service --state=running
```

7. Regresa al modo multi-usuario:

```bash
systemctl default
# O equivalente:
# systemctl isolate multi-user.target
```

8. Practica el cambio al emergency target:

```bash
sudo systemctl isolate emergency.target
```

> **Nota:** El emergency target monta el sistema de archivos raíz en modo solo lectura. Es más restrictivo que rescue.

9. Desde emergency, remonta el sistema de archivos en modo escritura y regresa:

```bash
mount -o remount,rw /
systemctl default
```

---

### Paso Final: Documentar los procedimientos de recuperación

**Objetivo:** Crear documentación profesional de los procedimientos ejecutados para referencia futura del equipo.

**Instrucciones:**

1. Inicia sesión como `root` y crea el documento de procedimientos:

```bash
cat > /opt/sysreport/recovery_procedures.txt << 'EOF'
============================================================
PROCEDIMIENTOS DE RECUPERACIÓN DE ARRANQUE - srv-linux-01
============================================================
Fecha: $(date)
Administrador: root
Sistema: Ubuntu Server 22.04.4 LTS o superior
============================================================

ESCENARIO 1: PARÁMETROS INCORRECTOS EN GRUB2
---------------------------------------------
Síntoma: Sistema no arranca, kernel panic o prompt initramfs
Causa: Parámetros inválidos en GRUB_CMDLINE_LINUX_DEFAULT

Procedimiento de recuperación temporal:
1. Reiniciar y presionar Shift para acceder al menú GRUB
2. Presionar 'e' para editar la entrada de arranque
3. Localizar la línea 'linux' y eliminar parámetros incorrectos
4. Presionar Ctrl+X para arrancar con parámetros corregidos

Procedimiento de corrección permanente:
1. Editar /etc/default/grub
2. Corregir las líneas GRUB_CMDLINE_LINUX_DEFAULT y GRUB_CMDLINE_LINUX
3. Ejecutar: sudo update-grub
4. Reiniciar y verificar

Prevención:
- Siempre crear backup antes de modificar /etc/default/grub
- Comando: sudo cp /etc/default/grub /etc/default/grub.backup.$(date +%Y%m%d)
- Verificar sintaxis antes de reiniciar

ESCENARIO 2: CONTRASEÑA DE ROOT PERDIDA
-----------------------------------------
Síntoma: Imposibilidad de acceder al sistema con credenciales conocidas
Causa: Contraseña cambiada o cuenta bloqueada

Procedimiento de recuperación:
1. Reiniciar y acceder al menú GRUB (Shift durante arranque)
2. Editar entrada de arranque (tecla 'e')
3. En línea 'linux': cambiar 'ro' por 'rw', añadir 'init=/bin/bash'
4. Arrancar con Ctrl+X
5. En el shell root: ejecutar 'passwd root' para nueva contraseña
6. Desbloquear cuentas: passwd -u <usuario>
7. Sincronizar: sync
8. Reiniciar: exec /sbin/init (o echo b > /proc/sysrq-trigger)

Método alternativo (rd.break):
1. Añadir 'rd.break' al final de la línea linux
2. mount -o remount,rw /sysroot
3. chroot /sysroot
4. passwd root
5. touch /.autorelabel (solo en sistemas con SELinux)
6. exit && reboot -f

Prevención:
- Documentar contraseñas en gestor de secretos corporativo
- Configurar acceso SSH con llaves para evitar dependencia de contraseñas

ESCENARIO 3: GRUB2 DAÑADO/CORRUPTO
------------------------------------
Síntoma: Prompt 'grub rescue>' o error de módulos no encontrados
Causa: Archivo grub.cfg corrupto o módulos de GRUB eliminados

Procedimiento de recuperación:
1. Arrancar desde ISO de Ubuntu Server (configurar orden de arranque en VM/BIOS)
2. Acceder a shell (Ctrl+Alt+F2 en Subiquity)
3. Identificar particiones: lsblk && fdisk -l /dev/sda
4. Montar sistema raíz: mount /dev/sdaX /mnt
5. Montar /boot si es separado: mount /dev/sdaY /mnt/boot
6. Montar sistemas virtuales:
   mount --bind /dev /mnt/dev
   mount --bind /dev/pts /mnt/dev/pts
   mount --bind /proc /mnt/proc
   mount --bind /sys /mnt/sys
   mount --bind /run /mnt/run
7. Entrar en chroot: chroot /mnt
8. Reinstalar GRUB: grub-install /dev/sda
9. Regenerar configuración: update-grub
10. Salir y desmontar: exit && umount -R /mnt
11. Retirar ISO y reiniciar

Prevención:
- Mantener backup de /boot/grub/grub.cfg
- Crear snapshots antes de modificaciones del sistema
- Comando de backup: sudo cp /boot/grub/grub.cfg /opt/backups/grub.cfg.$(date +%Y%m%d)

TARGETS DE SYSTEMD PARA RECUPERACIÓN
--------------------------------------
| Target              | Uso                                    | Equivalente |
|---------------------|----------------------------------------|-------------|
| emergency.target    | Sistema mínimo, root en solo lectura   | N/A         |
| rescue.target       | Single-user con servicios básicos      | Runlevel 1  |
| multi-user.target   | Operación normal sin GUI               | Runlevel 3  |
| graphical.target    | Operación normal con GUI               | Runlevel 5  |

Comandos útiles:
- Ver target actual: systemctl get-default
- Cambiar target en caliente: systemctl isolate <target>
- Cambiar target por defecto: systemctl set-default <target>
- Arrancar en rescue desde GRUB: añadir 'systemd.unit=rescue.target' a línea linux

============================================================
FIN DEL DOCUMENTO
============================================================
EOF
```

2. Reemplaza la fecha dinámica:

```bash
sed -i "s/\$(date)/$(date)/" /opt/sysreport/recovery_procedures.txt
```

3. Verifica el documento:

```bash
wc -l /opt/sysreport/recovery_procedures.txt
head -5 /opt/sysreport/recovery_procedures.txt
```

**Salida esperada:**
```
95 /opt/sysreport/recovery_procedures.txt
============================================================
PROCEDIMIENTOS DE RECUPERACIÓN DE ARRANQUE - srv-linux-01
============================================================
Fecha: Mon Mar XX HH:MM:SS UTC 2024
Administrador: root
```

---

## 7. Validación y Pruebas

Ejecuta los siguientes comandos para confirmar que todos los escenarios se completaron exitosamente:

```bash
echo "=== VALIDACIÓN FINAL DEL LAB 02 ==="
echo ""

# 1. Verificar que el sistema arranca correctamente
echo "1. Estado del sistema:"
systemctl is-system-running
echo ""

# 2. Verificar que GRUB está correctamente configurado
echo "2. Configuración de GRUB (sin errores):"
grep -c "sda99\|falso\|CORRUPTO" /boot/grub/grub.cfg && echo "FALLO: Aún hay errores en grub.cfg" || echo "OK: grub.cfg limpio"
echo ""

# 3. Verificar acceso de root
echo "3. Verificación de acceso root:"
sudo -n true 2>/dev/null && echo "OK: sudo funciona" || echo "NOTA: sudo requiere contraseña (normal)"
echo ""

# 4. Verificar que el documento de procedimientos existe
echo "4. Documento de recuperación:"
if [ -f /opt/sysreport/recovery_procedures.txt ]; then
    echo "OK: Archivo existe ($(wc -l < /opt/sysreport/recovery_procedures.txt) líneas)"
else
    echo "FALLO: Archivo no encontrado"
fi
echo ""

# 5. Verificar target actual
echo "5. Target de systemd:"
systemctl get-default
echo ""

# 6. Verificar tiempo de arranque
echo "6. Tiempo de arranque:"
systemd-analyze
echo ""

# 7. Verificar que no hay errores críticos en el journal
echo "7. Errores críticos en el arranque actual:"
ERRORES=$(journalctl -b --priority=crit --no-pager 2>/dev/null | grep -v "^--" | wc -l)
echo "Errores críticos encontrados: $ERRORES"
echo ""

echo "=== VALIDACIÓN COMPLETADA ==="
```

**Criterios de éxito:**

| Criterio | Resultado esperado |
|----------|-------------------|
| Sistema operativo | `running` |
| grub.cfg sin errores | `OK: grub.cfg limpio` |
| Documento de procedimientos | Existe con >80 líneas |
| Target por defecto | `multi-user.target` |
| Tiempo de arranque | Menor a 60 segundos |
| Errores críticos | 0 o mínimos |

---

## 8. Solución de Problemas

### Problema 1: El sistema no muestra el menú de GRUB al presionar Shift

**Síntomas:** Al reiniciar y presionar Shift (o Esc), el sistema arranca directamente sin mostrar el menú de GRUB. No hay oportunidad de editar parámetros de arranque.

**Causa:** Ubuntu 22.04 tiene configurado `GRUB_TIMEOUT=0` y `GRUB_TIMEOUT_STYLE=hidden` por defecto, lo que oculta el menú. En máquinas virtuales, la temporización de la tecla Shift puede no ser detectada correctamente por el firmware virtual.

**Solución:**

1. Si puedes acceder al sistema normalmente, modifica la configuración:

```bash
sudo nano /etc/default/grub
# Cambiar:
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10
# Guardar y ejecutar:
sudo update-grub
```

2. Si NO puedes acceder al sistema, usa VirtualBox para forzar el menú:
   - Apaga la VM completamente
   - Configuración → Sistema → Placa base → Habilita "EFI" si no está habilitado
   - O bien: inicia la VM y presiona **F12** inmediatamente para el menú de arranque del firmware, luego selecciona el disco y presiona Esc rápidamente

3. Como alternativa definitiva, arranca desde la ISO y ejecuta desde chroot:

```bash
chroot /mnt
sed -i 's/GRUB_TIMEOUT_STYLE=hidden/GRUB_TIMEOUT_STYLE=menu/' /etc/default/grub
sed -i 's/GRUB_TIMEOUT=0/GRUB_TIMEOUT=10/' /etc/default/grub
update-grub
exit
reboot
```

---

### Problema 2: Después de chroot, `grub-install` falla con "cannot find a device for /boot/grub"

**Síntomas:** Al ejecutar `grub-install /dev/sda` dentro del chroot, aparece un error:

```
grub-install: error: cannot find a device for /boot/grub (is /dev mounted?).
```

**Causa:** Los sistemas de archivos virtuales (`/dev`, `/proc`, `/sys`) no se montaron correctamente dentro del chroot, o se omitió el montaje de `/dev/pts` y `/run`. GRUB necesita acceso a los dispositivos de bloque a través de `/dev` para escribir en el MBR.

**Solución:**

1. Sal del chroot:

```bash
exit
```

2. Verifica y monta todos los sistemas virtuales necesarios:

```bash
# Verificar qué está montado
mount | grep /mnt

# Montar todo lo necesario (en orden)
mount --bind /dev /mnt/dev
mount --bind /dev/pts /mnt/dev/pts
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
mount --bind /run /mnt/run
```

3. Vuelve a entrar en el chroot:

```bash
chroot /mnt /bin/bash
```

4. Verifica que `/dev/sda` es accesible:

```bash
ls -la /dev/sda
```

**Salida esperada:**
```
brw-rw---- 1 root disk 8, 0 ... /dev/sda
```

5. Ahora ejecuta `grub-install` nuevamente:

```bash
grub-install --target=i386-pc /dev/sda
```

6. Si persiste el error, intenta con la opción `--force`:

```bash
grub-install --target=i386-pc --force /dev/sda
```

---

## 9. Limpieza

Después de completar exitosamente todos los escenarios, realiza la limpieza:

```bash
# Eliminar archivos de backup temporales (mantener los de /opt/sysreport)
sudo rm -f /etc/default/grub.backup
sudo rm -f /boot/grub/grub.cfg.backup

# Verificar que la configuración de GRUB es correcta
sudo grep -E "^GRUB_CMDLINE" /etc/default/grub

# Asegurar que el timeout de GRUB permite acceso al menú para futuros labs
sudo sed -i 's/GRUB_TIMEOUT_STYLE=hidden/GRUB_TIMEOUT_STYLE=menu/' /etc/default/grub 2>/dev/null
sudo sed -i 's/GRUB_TIMEOUT=0/GRUB_TIMEOUT=5/' /etc/default/grub 2>/dev/null
sudo update-grub

# Crear un snapshot post-lab para tener un punto de restauración limpio
# (Ejecutar desde el HOST)
# VBoxManage snapshot "srv-linux-01" take "post-lab02-complete" \
#   --description "Lab 02 completado - GRUB y recuperación configurados"
```

> **Importante:** NO elimines el snapshot `pre-lab02-clean` todavía. Puede ser necesario para labs posteriores si algo sale mal.

---

## 10. Resumen

### Competencias demostradas

En este laboratorio has aplicado exitosamente las siguientes habilidades de administración de sistemas:

| Competencia | Escenario | Herramientas utilizadas |
|-------------|-----------|------------------------|
| Diagnóstico de fallas de arranque | Escenario 1 | Edición interactiva de GRUB, `/etc/default/grub`, `update-grub` |
| Recuperación de acceso sin credenciales | Escenario 2 | `init=/bin/bash`, `rd.break`, `chroot`, `passwd` |
| Reinstalación de gestor de arranque | Escenario 3 | ISO de rescate, `mount --bind`, `chroot`, `grub-install`, `update-grub` |
| Gestión de targets de systemd | Todos | `systemctl isolate`, `systemctl get-default`, targets rescue/emergency |

### Relación con la secuencia de arranque

Cada escenario corresponde a una fase específica de la secuencia de arranque estudiada en la lección 2.1:

- **Escenario 1** → Fase 3 (GRUB2): parámetros incorrectos pasados al kernel
- **Escenario 2** → Fase 4-5 (Kernel/initramfs): interrupción controlada del arranque con `rd.break` o `init=/bin/bash`
- **Escenario 3** → Fase 2-3 (Cargador de primera etapa/GRUB2): corrupción del gestor de arranque

### Recursos adicionales

- `man grub-install` — Documentación del comando de instalación de GRUB
- `man update-grub` — Documentación de regeneración de configuración
- `info grub` — Manual completo de GNU GRUB2
- [Ubuntu Recovery Mode](https://wiki.ubuntu.com/RecoveryMode) — Wiki oficial de Ubuntu
- `man systemd.special` — Documentación de targets especiales de systemd
- `/usr/share/doc/grub2-common/` — Documentación local instalada con GRUB

---
