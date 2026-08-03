# Administración de Paquetes, Repositorios y Bibliotecas Compartidas en Linux

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 57 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio aplicarás el ciclo de vida completo de administración de paquetes en dos distribuciones Linux: Ubuntu 22.04 (APT/dpkg) y Rocky Linux 9.3 (DNF/rpm). Configurarás repositorios personalizados, resolverás dependencias rotas, gestionarás claves GPG y trabajarás con bibliotecas compartidas personalizadas usando `ldconfig` y `ldd`. Los paquetes instalados aquí (`curl`, `wget`, `jq`) serán utilizados en laboratorios posteriores de scripting.

## Objetivos de Aprendizaje

- [ ] Administrar el ciclo de vida completo de paquetes en Ubuntu usando APT y dpkg con resolución de dependencias
- [ ] Gestionar paquetes RPM en Rocky Linux 9.3 usando DNF y rpm para instalación, actualización y resolución de conflictos
- [ ] Configurar repositorios personalizados y gestionar claves GPG para garantizar la integridad de los paquetes
- [ ] Diagnosticar y resolver problemas de bibliotecas compartidas usando `ldconfig` y `ldd`

## Prerrequisitos

### Conocimientos Previos

- Familiaridad con la línea de comandos Linux (navegación, edición de archivos, permisos)
- Conceptos básicos de paquetes `.deb` y `.rpm`
- Comprensión del formato de repositorios APT (`sources.list`)

### Acceso Requerido

- VM `srv-linux-01` (Ubuntu 22.04.4 LTS) operativa con IP 192.168.100.10
- VM `srv-linux-03` (Rocky Linux 9.3) operativa con IP 192.168.100.30
- Acceso `sudo` con usuario `sysadmin` en ambas VMs
- Conectividad a internet en ambas VMs para descarga de paquetes

## Entorno del Laboratorio

### Máquinas Virtuales

| VM | SO | IP | Rol |
|----|----|----|-----|
| srv-linux-01 | Ubuntu 22.04.4 LTS | 192.168.100.10 | Gestión APT/dpkg, repositorio local, bibliotecas compartidas |
| srv-linux-03 | Rocky Linux 9.3 | 192.168.100.30 | Gestión DNF/rpm, repositorios YUM |

### Software Necesario

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| apt | 2.4.12 | Gestión de paquetes alto nivel (Ubuntu) |
| dpkg | 1.21.1 | Gestión de paquetes bajo nivel (Ubuntu) |
| DNF | 4.14.0 | Gestión de paquetes alto nivel (Rocky) |
| rpm | 4.16.1.3 | Gestión de paquetes bajo nivel (Rocky) |
| GCC | 11.4.0 | Compilación de biblioteca compartida |
| dpkg-dev | 1.21.1 | Creación de repositorio local |

### Preparación Inicial

Ejecutar en ambas VMs para verificar conectividad:

```bash
# En srv-linux-01 (Ubuntu)
ssh sysadmin@192.168.100.10
ping -c 2 archive.ubuntu.com

# En srv-linux-03 (Rocky Linux)
ssh sysadmin@192.168.100.30
ping -c 2 mirror.rockylinux.org
```

Crear los directorios de trabajo estándar si no existen:

```bash
# En ambas VMs
sudo mkdir -p /opt/sysreport /opt/scripts /opt/backups
```

---

## Paso 1: Gestión de Paquetes con APT y dpkg en Ubuntu

**Objetivo:** Instalar, consultar, actualizar y eliminar paquetes en srv-linux-01 usando las herramientas APT y dpkg, incluyendo los paquetes `curl`, `wget` y `jq` requeridos para laboratorios futuros.

### Instrucciones

**1.1** Conéctate a srv-linux-01 y actualiza la caché de repositorios:

```bash
ssh sysadmin@192.168.100.10
sudo apt update
```

**1.2** Instala los paquetes esenciales para laboratorios posteriores:

```bash
sudo apt install -y curl wget jq
```

**1.3** Verifica la instalación y versiones de cada paquete:

```bash
dpkg -s curl | grep -E "^(Package|Status|Version)"
dpkg -s wget | grep -E "^(Package|Status|Version)"
dpkg -s jq | grep -E "^(Package|Status|Version)"
```

**1.4** Consulta los archivos instalados por el paquete `jq`:

```bash
dpkg -L jq
```

**1.5** Identifica a qué paquete pertenece el binario `curl`:

```bash
dpkg -S $(which curl)
```

**1.6** Muestra las dependencias del paquete `curl`:

```bash
apt depends curl
```

**1.7** Muestra las dependencias inversas (qué paquetes dependen de `libcurl4`):

```bash
apt rdepends libcurl4 | head -20
```

**1.8** Simula la instalación de `nginx` sin ejecutarla (dry-run):

```bash
apt install --dry-run nginx
```

**1.9** Bloquea el paquete `curl` para prevenir actualizaciones accidentales:

```bash
sudo apt-mark hold curl
apt-mark showhold
```

**1.10** Genera un reporte de los paquetes instalados manualmente:

```bash
apt-mark showmanual | sort > /opt/sysreport/manual_packages_ubuntu.txt
wc -l /opt/sysreport/manual_packages_ubuntu.txt
```

### Salida Esperada

Para el paso 1.3, deberías ver algo similar a:

```
Package: curl
Status: install ok installed
Version: 7.81.0-1ubuntu1.16

Package: wget
Status: install ok installed
Version: 1.21.2-2ubuntu1.1

Package: jq
Status: install ok installed
Version: 1.6-2.1ubuntu3
```

Para el paso 1.9:

```
curl
```

### Verificación

```bash
# Confirmar que los tres paquetes están instalados y funcionales
curl --version | head -1
wget --version | head -1
jq --version

# Verificar que curl está en hold
dpkg -l curl | grep "^hi"
```

---

## Paso 2: Gestión de Paquetes con DNF y rpm en Rocky Linux

**Objetivo:** Realizar operaciones equivalentes de gestión de paquetes en srv-linux-03 usando DNF y rpm, incluyendo búsqueda, instalación, consulta e historial de transacciones.

### Instrucciones

**2.1** Conéctate a srv-linux-03 y verifica los repositorios habilitados:

```bash
ssh sysadmin@192.168.100.30
dnf repolist
```

**2.2** Actualiza la caché de metadatos:

```bash
sudo dnf makecache
```

**2.3** Busca el paquete `htop` en los repositorios:

```bash
dnf search htop
```

**2.4** Muestra información detallada antes de instalar:

```bash
dnf info htop
```

**2.5** Instala `htop`, `tree` y `tmux`:

```bash
sudo dnf install -y htop tree tmux
```

**2.6** Consulta los paquetes instalados con rpm:

```bash
rpm -qi htop
rpm -ql htop | head -10
```

**2.7** Identifica a qué paquete pertenece un archivo:

```bash
rpm -qf /usr/bin/htop
```

**2.8** Lista las dependencias de un paquete con rpm:

```bash
rpm -qR htop
```

**2.9** Consulta el historial de transacciones DNF:

```bash
sudo dnf history
sudo dnf history info last
```

**2.10** Bloquea el paquete `tmux` para evitar actualizaciones:

```bash
sudo dnf install -y python3-dnf-plugin-versionlock
sudo dnf versionlock add tmux
sudo dnf versionlock list
```

**2.11** Genera un reporte de paquetes instalados:

```bash
rpm -qa --queryformat '%{NAME}-%{VERSION}-%{RELEASE}.%{ARCH}\n' | sort > /opt/sysreport/installed_packages_rocky.txt
wc -l /opt/sysreport/installed_packages_rocky.txt
```

### Salida Esperada

Para el paso 2.1:

```
repo id                      repo name                              status
appstream                    Rocky Linux 9 - AppStream              enabled
baseos                       Rocky Linux 9 - BaseOS                 enabled
extras                       Rocky Linux 9 - Extras                 enabled
```

Para el paso 2.6 (parcial):

```
Name        : htop
Version     : 3.2.1
Release     : 3.el9
Architecture: x86_64
Install Date: [fecha actual]
...
```

### Verificación

```bash
# Confirmar instalación funcional
htop --version
tree --version
tmux -V

# Verificar versionlock
sudo dnf versionlock list | grep tmux
```

---

## Paso 3: Configurar un Repositorio Local APT en Ubuntu

**Objetivo:** Crear un repositorio local de paquetes `.deb` en srv-linux-01 usando `dpkg-scanpackages`, configurar APT para utilizarlo y verificar la instalación desde el repositorio local.

### Instrucciones

**3.1** Instala las herramientas necesarias para crear repositorios:

```bash
ssh sysadmin@192.168.100.10
sudo apt install -y dpkg-dev
```

**3.2** Crea la estructura de directorios del repositorio local:

```bash
sudo mkdir -p /opt/local-repo/packages
```

**3.3** Descarga un paquete `.deb` sin instalarlo para usarlo en el repositorio:

```bash
cd /tmp
apt download cowsay
sudo cp cowsay*.deb /opt/local-repo/packages/
```

**3.4** Descarga un segundo paquete para tener contenido adicional:

```bash
apt download sl
sudo cp sl*.deb /opt/local-repo/packages/
```

**3.5** Genera el archivo de índice `Packages` del repositorio:

```bash
cd /opt/local-repo
dpkg-scanpackages packages /dev/null | sudo tee packages/Packages
sudo gzip -k packages/Packages
```

**3.6** Verifica el contenido del archivo de índice:

```bash
cat /opt/local-repo/packages/Packages
```

**3.7** Configura APT para usar el repositorio local:

```bash
echo "deb [trusted=yes] file:///opt/local-repo packages/" | \
  sudo tee /etc/apt/sources.list.d/local-repo.list
```

**3.8** Actualiza la caché e instala desde el repositorio local:

```bash
sudo apt update
apt-cache policy cowsay
sudo apt install -y cowsay
```

**3.9** Verifica que el paquete proviene del repositorio local:

```bash
apt-cache policy cowsay | grep -A2 "Installed"
```

### Salida Esperada

Para el paso 3.6, el archivo `Packages` mostrará metadatos como:

```
Package: cowsay
Version: 3.03+dfsg2-8
Architecture: all
Filename: packages/cowsay_3.03+dfsg2-8_all.deb
Size: [tamaño]
...
```

Para el paso 3.9:

```
cowsay:
  Installed: 3.03+dfsg2-8
  Candidate: 3.03+dfsg2-8
  Version table:
 *** 3.03+dfsg2-8 100
        100 file:///opt/local-repo packages/ Packages
```

### Verificación

```bash
# Confirmar instalación funcional desde repositorio local
cowsay "Repositorio local funcionando"

# Verificar que el repositorio aparece en las fuentes
grep -r "local-repo" /etc/apt/sources.list.d/
```

---

## Paso 4: Configurar un Repositorio Personalizado en Rocky Linux

**Objetivo:** Crear y configurar un repositorio YUM/DNF personalizado en srv-linux-03, incluyendo la verificación de integridad con claves GPG.

### Instrucciones

**4.1** Conéctate a srv-linux-03 e instala las herramientas necesarias:

```bash
ssh sysadmin@192.168.100.30
sudo dnf install -y createrepo_c
```

**4.2** Crea la estructura del repositorio local:

```bash
sudo mkdir -p /opt/local-repo/packages
```

**4.3** Descarga un paquete RPM para incluirlo en el repositorio:

```bash
sudo dnf download --destdir=/opt/local-repo/packages/ cowsay
```

> **Nota:** Si `cowsay` no está disponible en los repositorios base, usa `fortune-mod` o descarga un RPM de EPEL:

```bash
sudo dnf install -y epel-release
sudo dnf download --destdir=/opt/local-repo/packages/ cowsay
```

**4.4** Genera los metadatos del repositorio:

```bash
sudo createrepo_c /opt/local-repo/
```

**4.5** Crea el archivo de configuración del repositorio:

```bash
sudo tee /etc/yum.repos.d/local-custom.repo << 'EOF'
[local-custom]
name=Repositorio Local Personalizado
baseurl=file:///opt/local-repo/
enabled=1
gpgcheck=0
priority=10
EOF
```

**4.6** Verifica que el repositorio es reconocido:

```bash
sudo dnf clean all
sudo dnf repolist | grep local-custom
```

**4.7** Instala un paquete desde el repositorio local:

```bash
sudo dnf install -y --repo=local-custom cowsay
```

**4.8** Verifica la procedencia del paquete:

```bash
dnf info cowsay | grep "Repository"
```

### Salida Esperada

Para el paso 4.4:

```
Directory walk started
Directory walk done - 1 packages
Temporary output repo path: /opt/local-repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```

Para el paso 4.6:

```
repo id              repo name                                    status
local-custom         Repositorio Local Personalizado              1
```

### Verificación

```bash
# Confirmar que el repositorio está activo y funcional
dnf repoinfo local-custom
rpm -qi cowsay | grep "Install Date"
```

---

## Paso 5: Simulación y Resolución de Dependencias Rotas en Ubuntu

**Objetivo:** Simular un escenario de dependencias rotas en srv-linux-01 y aplicar las técnicas de reparación con `dpkg --configure -a` y `apt install -f`.

### Instrucciones

**5.1** Conéctate a srv-linux-01:

```bash
ssh sysadmin@192.168.100.10
```

**5.2** Descarga un paquete que tiene dependencias sin descargarlas:

```bash
cd /tmp
apt download nginx-core
```

**5.3** Intenta instalar solo el paquete sin resolver dependencias (esto generará un error controlado):

```bash
sudo dpkg -i nginx-core*.deb 2>&1 | tee /opt/sysreport/dependency_error.txt
```

**5.4** Verifica el estado de paquetes con problemas:

```bash
dpkg -l | grep -E "^(iF|iU)" | tee /opt/sysreport/broken_packages.txt
```

**5.5** Intenta completar configuraciones pendientes:

```bash
sudo dpkg --configure -a
```

**5.6** Resuelve las dependencias insatisfechas con APT:

```bash
sudo apt install -f -y
```

**5.7** Verifica que no quedan paquetes en estado inconsistente:

```bash
dpkg -l | grep -E "^(iF|iU|iH)" | wc -l
```

**5.8** Limpia el paquete nginx-core instalado para no afectar el sistema:

```bash
sudo apt purge -y nginx-core nginx-common
sudo apt autoremove -y
```

**5.9** Consulta el historial de APT para documentar la resolución:

```bash
tail -30 /var/log/apt/history.log
```

### Salida Esperada

Para el paso 5.3, verás errores de dependencias:

```
dpkg: dependency problems prevent configuration of nginx-core:
 nginx-core depends on libnginx-mod-http-geoip2 (= ...); however:
  Package libnginx-mod-http-geoip2 is not installed.
 nginx-core depends on nginx-common (= ...); however:
  Package nginx-common is not installed.
...
Errors were encountered while processing:
 nginx-core
```

Para el paso 5.7, el resultado debe ser:

```
0
```

### Verificación

```bash
# Confirmar que el sistema está limpio
dpkg --audit
echo $?  # Debe retornar 0 si no hay problemas

# Verificar que el reporte se generó
cat /opt/sysreport/dependency_error.txt | head -5
```

---

## Paso 6: Resolución de Conflictos de Paquetes en Rocky Linux

**Objetivo:** Demostrar la gestión de conflictos de paquetes y el uso de historial de transacciones DNF para revertir cambios en srv-linux-03.

### Instrucciones

**6.1** Conéctate a srv-linux-03:

```bash
ssh sysadmin@192.168.100.30
```

**6.2** Instala un paquete y registra la transacción:

```bash
sudo dnf install -y httpd
sudo dnf history | head -5
```

**6.3** Muestra los detalles de la última transacción:

```bash
sudo dnf history info last
```

**6.4** Consulta las dependencias que se instalaron junto con `httpd`:

```bash
rpm -qR httpd | head -15
```

**6.5** Verifica archivos de configuración del paquete:

```bash
rpm -qc httpd
```

**6.6** Verifica la integridad del paquete instalado (archivos modificados):

```bash
sudo rpm -V httpd
```

**6.7** Revierte la transacción de instalación de httpd:

```bash
LAST_ID=$(sudo dnf history | grep "install" | head -1 | awk '{print $1}')
sudo dnf history undo $LAST_ID -y
```

**6.8** Confirma que httpd fue removido:

```bash
rpm -q httpd
```

### Salida Esperada

Para el paso 6.6, si no hay modificaciones:

```
(sin salida - el paquete está íntegro)
```

Para el paso 6.8:

```
package httpd is not installed
```

### Verificación

```bash
# Verificar que la reversión fue exitosa
dnf history | head -5
which httpd 2>&1  # Debe indicar que no se encuentra
```

---

## Paso 7: Creación y Gestión de Bibliotecas Compartidas

**Objetivo:** Crear una biblioteca compartida personalizada (`.so`), registrarla con `ldconfig` y diagnosticar vinculaciones con `ldd` en srv-linux-01.

### Instrucciones

**7.1** Conéctate a srv-linux-01 e instala GCC:

```bash
ssh sysadmin@192.168.100.10
sudo apt install -y gcc build-essential
```

**7.2** Crea el directorio de trabajo para la biblioteca:

```bash
mkdir -p /opt/scripts/sharedlib
cd /opt/scripts/sharedlib
```

**7.3** Crea el código fuente de la biblioteca compartida:

```bash
cat > sysinfo.c << 'EOF'
#include <stdio.h>
#include <sys/utsname.h>

void print_system_info(void) {
    struct utsname info;
    if (uname(&info) == 0) {
        printf("Sistema: %s\n", info.sysname);
        printf("Hostname: %s\n", info.nodename);
        printf("Kernel: %s\n", info.release);
        printf("Arquitectura: %s\n", info.machine);
    }
}

int get_cpu_count(void) {
    FILE *fp = fopen("/proc/cpuinfo", "r");
    int count = 0;
    char line[256];
    if (fp) {
        while (fgets(line, sizeof(line), fp)) {
            if (strncmp(line, "processor", 9) == 0)
                count++;
        }
        fclose(fp);
    }
    return count;
}
EOF
```

**7.4** Compila la biblioteca compartida:

```bash
gcc -shared -fPIC -o libsysinfo.so.1.0.0 sysinfo.c
```

**7.5** Crea los enlaces simbólicos estándar (soname):

```bash
ln -sf libsysinfo.so.1.0.0 libsysinfo.so.1
ln -sf libsysinfo.so.1 libsysinfo.so
```

**7.6** Copia la biblioteca al directorio estándar del sistema:

```bash
sudo cp libsysinfo.so.1.0.0 /usr/local/lib/
sudo ln -sf /usr/local/lib/libsysinfo.so.1.0.0 /usr/local/lib/libsysinfo.so.1
sudo ln -sf /usr/local/lib/libsysinfo.so.1 /usr/local/lib/libsysinfo.so
```

**7.7** Registra la biblioteca con `ldconfig`:

```bash
# Verificar que /usr/local/lib está en la configuración de ldconfig
cat /etc/ld.so.conf.d/*.conf | grep local

# Si no está, agregar la ruta
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/local-custom.conf

# Actualizar la caché de bibliotecas
sudo ldconfig
```

**7.8** Verifica que la biblioteca está registrada:

```bash
ldconfig -p | grep sysinfo
```

**7.9** Crea un programa que use la biblioteca:

```bash
cat > test_sysinfo.c << 'EOF'
#include <stdio.h>

extern void print_system_info(void);
extern int get_cpu_count(void);

int main() {
    printf("=== Información del Sistema ===\n");
    print_system_info();
    printf("CPUs detectadas: %d\n", get_cpu_count());
    return 0;
}
EOF
```

**7.10** Compila el programa enlazándolo con la biblioteca:

```bash
gcc -o test_sysinfo test_sysinfo.c -lsysinfo -L/usr/local/lib
```

**7.11** Ejecuta el programa y verifica:

```bash
./test_sysinfo
```

**7.12** Usa `ldd` para inspeccionar las dependencias del binario:

```bash
ldd ./test_sysinfo
```

**7.13** Diagnostica qué pasaría si la biblioteca no estuviera registrada:

```bash
# Simular eliminando temporalmente de la caché
sudo rm /usr/local/lib/libsysinfo.so.1
sudo ldconfig

# Intentar ejecutar (fallará)
./test_sysinfo 2>&1 | tee /opt/sysreport/ldd_error.txt

# Restaurar
sudo ln -sf /usr/local/lib/libsysinfo.so.1.0.0 /usr/local/lib/libsysinfo.so.1
sudo ldconfig

# Verificar que funciona de nuevo
./test_sysinfo
```

### Salida Esperada

Para el paso 7.8:

```
	libsysinfo.so.1 (libc6,x86-64) => /usr/local/lib/libsysinfo.so.1
```

Para el paso 7.11:

```
=== Información del Sistema ===
Sistema: Linux
Hostname: srv-linux-01
Kernel: 5.15.0-xxx-generic
Arquitectura: x86_64
CPUs detectadas: 4
```

Para el paso 7.12:

```
	linux-vdso.so.1 (0x00007fff...)
	libsysinfo.so.1 => /usr/local/lib/libsysinfo.so.1 (0x00007f...)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f...)
	/lib64/ld-linux-x86-64.so.2 (0x00007f...)
```

Para el paso 7.13 (error):

```
./test_sysinfo: error while loading shared libraries: libsysinfo.so.1: cannot open shared object file: No such file or directory
```

### Verificación

```bash
# Verificación final completa
ldd ./test_sysinfo | grep "not found"  # No debe mostrar nada
./test_sysinfo  # Debe ejecutarse sin errores
ldconfig -p | grep sysinfo  # Debe mostrar la biblioteca
```

---

## Validación y Pruebas Finales

Ejecuta los siguientes comandos para confirmar que todos los objetivos del laboratorio se cumplieron correctamente.

### En srv-linux-01 (Ubuntu):

```bash
echo "=== VALIDACIÓN srv-linux-01 ==="

# 1. Paquetes esenciales instalados
echo "[1] Verificando paquetes esenciales..."
for pkg in curl wget jq; do
    if dpkg -s $pkg &>/dev/null; then
        echo "  ✓ $pkg instalado: $(dpkg -s $pkg | grep Version | awk '{print $2}')"
    else
        echo "  ✗ $pkg NO instalado"
    fi
done

# 2. curl en hold
echo "[2] Verificando hold en curl..."
apt-mark showhold | grep -q curl && echo "  ✓ curl bloqueado" || echo "  ✗ curl NO bloqueado"

# 3. Repositorio local configurado
echo "[3] Verificando repositorio local..."
test -f /etc/apt/sources.list.d/local-repo.list && \
  echo "  ✓ Repositorio local configurado" || echo "  ✗ Repositorio local NO configurado"
test -f /opt/local-repo/packages/Packages && \
  echo "  ✓ Índice Packages generado" || echo "  ✗ Índice Packages NO encontrado"

# 4. Biblioteca compartida
echo "[4] Verificando biblioteca compartida..."
ldconfig -p | grep -q sysinfo && \
  echo "  ✓ libsysinfo registrada en ldconfig" || echo "  ✗ libsysinfo NO registrada"
test -x /opt/scripts/sharedlib/test_sysinfo && \
  echo "  ✓ Binario test_sysinfo compilado" || echo "  ✗ Binario NO encontrado"

# 5. Reportes generados
echo "[5] Verificando reportes..."
test -f /opt/sysreport/manual_packages_ubuntu.txt && \
  echo "  ✓ Reporte de paquetes manuales generado" || echo "  ✗ Reporte NO generado"
test -f /opt/sysreport/dependency_error.txt && \
  echo "  ✓ Reporte de error de dependencias generado" || echo "  ✗ Reporte NO generado"

echo "=== FIN VALIDACIÓN ==="
```

### En srv-linux-03 (Rocky Linux):

```bash
echo "=== VALIDACIÓN srv-linux-03 ==="

# 1. Paquetes instalados
echo "[1] Verificando paquetes..."
for pkg in htop tree tmux; do
    if rpm -q $pkg &>/dev/null; then
        echo "  ✓ $pkg instalado: $(rpm -q $pkg)"
    else
        echo "  ✗ $pkg NO instalado"
    fi
done

# 2. Versionlock activo
echo "[2] Verificando versionlock..."
sudo dnf versionlock list 2>/dev/null | grep -q tmux && \
  echo "  ✓ tmux bloqueado con versionlock" || echo "  ✗ tmux NO bloqueado"

# 3. Repositorio local
echo "[3] Verificando repositorio local..."
test -f /etc/yum.repos.d/local-custom.repo && \
  echo "  ✓ Repositorio local configurado" || echo "  ✗ Repositorio local NO configurado"
test -d /opt/local-repo/repodata && \
  echo "  ✓ Metadatos del repositorio generados" || echo "  ✗ Metadatos NO encontrados"

# 4. httpd removido (reversión exitosa)
echo "[4] Verificando reversión de httpd..."
rpm -q httpd &>/dev/null && \
  echo "  ✗ httpd sigue instalado" || echo "  ✓ httpd removido correctamente"

# 5. Reporte generado
echo "[5] Verificando reportes..."
test -f /opt/sysreport/installed_packages_rocky.txt && \
  echo "  ✓ Reporte de paquetes Rocky generado" || echo "  ✗ Reporte NO generado"

echo "=== FIN VALIDACIÓN ==="
```

---

## Solución de Problemas

### Problema 1: Error "dpkg was interrupted" al ejecutar apt

**Síntomas:**

```
E: dpkg was interrupted, you must manually run 'sudo dpkg --configure -a' to correct the problem.
```

Al intentar ejecutar `sudo apt install` o `sudo apt update`, APT se niega a continuar mostrando este mensaje.

**Causa:**

Una operación previa de `dpkg` fue interrumpida (por ejemplo, la VM se apagó durante una instalación o se cerró la terminal accidentalmente). La base de datos de `dpkg` quedó en un estado inconsistente con paquetes parcialmente configurados.

**Solución:**

```bash
# Paso 1: Completar configuraciones pendientes
sudo dpkg --configure -a

# Paso 2: Si persiste, forzar la resolución de dependencias
sudo apt install -f -y

# Paso 3: Si hay archivos de bloqueo huérfanos
sudo rm -f /var/lib/dpkg/lock-frontend
sudo rm -f /var/lib/dpkg/lock
sudo rm -f /var/cache/apt/archives/lock
sudo dpkg --configure -a

# Paso 4: Verificar estado limpio
dpkg --audit
```

---

### Problema 2: "error while loading shared libraries: libXXX.so: cannot open shared object file"

**Síntomas:**

Al ejecutar un binario compilado contra una biblioteca compartida personalizada:

```
./test_sysinfo: error while loading shared libraries: libsysinfo.so.1: cannot open shared object file: No such file or directory
```

El comando `ldd ./test_sysinfo` muestra:

```
libsysinfo.so.1 => not found
```

**Causa:**

La biblioteca compartida no está registrada en la caché de `ldconfig`. Esto ocurre cuando:
1. La biblioteca se copió a un directorio que no está en `/etc/ld.so.conf` ni en `/etc/ld.so.conf.d/`
2. Se olvidó ejecutar `sudo ldconfig` después de copiar la biblioteca
3. El enlace simbólico del soname está roto o ausente

**Solución:**

```bash
# Paso 1: Verificar que la biblioteca existe físicamente
find /usr/local/lib /usr/lib /lib -name "libsysinfo*" 2>/dev/null

# Paso 2: Verificar que la ruta está en la configuración de ldconfig
cat /etc/ld.so.conf
ls /etc/ld.so.conf.d/

# Paso 3: Si la ruta no está incluida, agregarla
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/local-custom.conf

# Paso 4: Verificar que los symlinks del soname existen
ls -la /usr/local/lib/libsysinfo*

# Paso 5: Recrear symlinks si faltan
sudo ln -sf /usr/local/lib/libsysinfo.so.1.0.0 /usr/local/lib/libsysinfo.so.1
sudo ln -sf /usr/local/lib/libsysinfo.so.1 /usr/local/lib/libsysinfo.so

# Paso 6: Actualizar la caché
sudo ldconfig

# Paso 7: Verificar registro
ldconfig -p | grep sysinfo

# Alternativa temporal (sin modificar ldconfig):
LD_LIBRARY_PATH=/usr/local/lib ./test_sysinfo
```

---

## Limpieza

Si deseas revertir los cambios realizados en este laboratorio (opcional, ya que algunos paquetes se necesitan en labs futuros):

### En srv-linux-01 (Ubuntu) — Limpieza parcial:

```bash
# NO eliminar curl, wget, jq (necesarios para lab 07)

# Desbloquear curl si ya no se necesita el hold
sudo apt-mark unhold curl

# Eliminar paquete de prueba cowsay
sudo apt purge -y cowsay
sudo apt autoremove -y

# Eliminar repositorio local (opcional)
sudo rm /etc/apt/sources.list.d/local-repo.list
sudo rm -rf /opt/local-repo
sudo apt update

# Mantener la biblioteca compartida como referencia educativa
# Para eliminarla completamente:
# sudo rm /usr/local/lib/libsysinfo*
# sudo rm /etc/ld.so.conf.d/local-custom.conf
# sudo ldconfig
```

### En srv-linux-03 (Rocky Linux) — Limpieza parcial:

```bash
# Eliminar versionlock de tmux
sudo dnf versionlock delete tmux

# Eliminar paquetes de prueba (mantener htop, tree, tmux si son útiles)
sudo dnf remove -y cowsay

# Eliminar repositorio local (opcional)
sudo rm /etc/yum.repos.d/local-custom.repo
sudo rm -rf /opt/local-repo
sudo dnf clean all
```

---

## Resumen

En este laboratorio se aplicaron las siguientes competencias de administración de paquetes:

| Competencia | Ubuntu (srv-linux-01) | Rocky Linux (srv-linux-03) |
|-------------|----------------------|---------------------------|
| Instalación de paquetes | `apt install`, `dpkg -i` | `dnf install`, `rpm -i` |
| Consulta de información | `dpkg -s`, `apt show` | `rpm -qi`, `dnf info` |
| Búsqueda de archivos | `dpkg -L`, `dpkg -S` | `rpm -ql`, `rpm -qf` |
| Bloqueo de versiones | `apt-mark hold` | `dnf versionlock` |
| Repositorios locales | `dpkg-scanpackages` | `createrepo_c` |
| Resolución de dependencias | `apt install -f`, `dpkg --configure -a` | `dnf history undo` |
| Bibliotecas compartidas | `ldconfig`, `ldd` | — |

**Paquetes clave instalados para laboratorios futuros:**
- `curl` 7.81.0 — transferencia de datos HTTP/HTTPS
- `wget` 1.21.2 — descarga de archivos desde la web
- `jq` 1.6 — procesamiento de JSON en línea de comandos

### Recursos Adicionales

- [Documentación oficial de APT — Debian Wiki](https://wiki.debian.org/Apt)
- [DNF Command Reference — Fedora Docs](https://dnf.readthedocs.io/)
- [Shared Libraries HOWTO — Linux Documentation Project](https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)
- `man dpkg`, `man apt`, `man dnf`, `man rpm`, `man ldconfig`, `man ldd`
