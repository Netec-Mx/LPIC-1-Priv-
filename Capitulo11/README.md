# Práctica 11 — Identificar y resolver fallas de comunicación, DNS, rutas y acceso remoto utilizando herramientas de diagnóstico y administración de redes Linux

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 135 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio configurarás manualmente interfaces de red con Netplan en dos máquinas virtuales Ubuntu 22.04.4 LTS, diagnosticarás fallas de conectividad inducidas intencionalmente usando herramientas como `ping`, `traceroute`, `mtr`, `ss` y `tcpdump`, resolverás problemas de DNS con `dig`/`nslookup`/`host`, y establecerás acceso remoto seguro mediante SSH con autenticación basada en llaves RSA-4096. Finalmente, transferirás archivos de forma segura con SCP y SFTP verificando su integridad.

## Objetivos de Aprendizaje

- [ ] Configurar interfaces de red IPv4 e IPv6 estáticas usando Netplan y verificar con `ip addr`
- [ ] Diagnosticar fallas de conectividad identificando la capa OSI afectada mediante `ping`, `traceroute`, `ss` y `tcpdump`
- [ ] Resolver problemas de DNS interpretando registros A, AAAA, MX, CNAME y PTR con `dig` y `nslookup`
- [ ] Configurar acceso SSH con autenticación basada en llaves RSA-4096 y gestionar passphrases con `ssh-agent`
- [ ] Transferir archivos entre hosts con SCP y SFTP verificando integridad mediante checksums SHA-256

## Prerrequisitos

### Conocimientos previos

- Modelo OSI/TCP-IP: función de las capas 2, 3 y 4
- Direccionamiento IPv4: notación CIDR, máscaras de subred, cálculo de rangos
- Conceptos básicos de IPv6: link-local, global unicast
- Navegación en terminal Linux: `cd`, `ls`, `cat`, `nano`/`vim`, redirección
- Concepto de DNS y tipos de registros básicos

### Acceso requerido

- Dos VMs Ubuntu 22.04.4 LTS instaladas y operativas en VirtualBox 7.0.14
- Usuario `labuser` con privilegios `sudo` en ambas VMs
- Adaptador de red interna configurado en VirtualBox (nombre: `labnet`)
- Acceso a consola de cada VM (ventana VirtualBox o SSH desde host si ya está configurado)

## Entorno de Laboratorio

### Topología de red

```
┌─────────────────────┐         Red Interna: labnet         ┌─────────────────────┐
│     srv-linux        │        192.168.100.0/24             │     cli-linux        │
│  192.168.100.10/24   │◄──────────────────────────────────►│  192.168.100.20/24   │
│  IPv6: fd00::10/64   │         enp0s3 ↔ enp0s3            │  IPv6: fd00::20/64   │
│  Gateway: .1 (sim)   │                                     │  Gateway: .10        │
└─────────────────────┘                                     └─────────────────────┘
```

### Máquinas virtuales

| Nombre VM | SO | IP (enp0s3) | IPv6 | Rol |
|---|---|---|---|---|
| `srv-linux` | Ubuntu 22.04.4 LTS | 192.168.100.10/24 | fd00::10/64 | Servidor SSH, DNS simulado |
| `cli-linux` | Ubuntu 22.04.4 LTS | 192.168.100.20/24 | fd00::20/64 | Cliente SSH, diagnóstico |

### Software necesario

| Paquete | Versión | Propósito |
|---|---|---|
| `iproute2` | 5.15.0 | Comandos `ip`, `ss` |
| `openssh-server` / `openssh-client` | 8.9p1 | Servicio y cliente SSH |
| `tcpdump` | 4.99.1 | Captura de paquetes |
| `traceroute` | 2.1.0 | Diagnóstico de rutas |
| `mtr` | 0.95 | Traceroute interactivo |
| `dnsutils` | 9.18.x | `dig`, `nslookup`, `host` |
| `nmap` | 7.91 | Escaneo de puertos |
| `net-tools` | 1.60 | `netstat`, `ifconfig` (legacy) |

---

## Paso 1 — Preparación del Entorno y Configuración de VirtualBox

### Objetivo

Crear la red interna en VirtualBox y verificar que ambas VMs tienen el adaptador correcto asignado.

### Instrucciones

1. Apaga ambas VMs si están encendidas:

```bash
# Desde el host (PowerShell/Terminal)
VBoxManage controlvm srv-linux poweroff 2>/dev/null
VBoxManage controlvm cli-linux poweroff 2>/dev/null
```

2. Configura el adaptador 1 de **srv-linux** como red interna `labnet`:

```bash
VBoxManage modifyvm srv-linux --nic1 intnet --intnet1 labnet
```

3. Configura el adaptador 1 de **cli-linux** como red interna `labnet`:

```bash
VBoxManage modifyvm cli-linux --nic1 intnet --intnet1 labnet
```

4. (Opcional) Añade un adaptador 2 NAT a **srv-linux** para acceso a internet:

```bash
VBoxManage modifyvm srv-linux --nic2 nat
```

5. Inicia ambas VMs:

```bash
VBoxManage startvm srv-linux --type headless
VBoxManage startvm cli-linux --type headless
```

6. Accede a la consola de cada VM e inicia sesión como `labuser`.

### Verificación

```bash
# En cada VM, verifica que la interfaz de red existe
ip link show enp0s3
```

**Salida esperada:** La interfaz `enp0s3` aparece listada (puede estar en estado `DOWN` si aún no se ha configurado IP).

---

## Paso 2 — Configuración de Red con Netplan (IPv4 e IPv6)

### Objetivo

Asignar direcciones IPv4 e IPv6 estáticas a ambas VMs usando Netplan y verificar conectividad L3.

### Instrucciones

#### En srv-linux (192.168.100.10):

1. Respalda la configuración actual de Netplan:

```bash
sudo cp /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.yaml.bak
```

2. Edita el archivo de configuración:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

3. Reemplaza el contenido con:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses:
        - 192.168.100.10/24
        - fd00::10/64
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
        search:
          - lab.local
```

4. Aplica la configuración:

```bash
sudo netplan apply
```

5. Verifica la asignación:

```bash
ip addr show enp0s3
ip -6 addr show enp0s3
```

#### En cli-linux (192.168.100.20):

6. Repite los pasos 1-2 y usa el siguiente contenido:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses:
        - 192.168.100.20/24
        - fd00::20/64
      routes:
        - to: default
          via: 192.168.100.10
      nameservers:
        addresses:
          - 192.168.100.10
          - 8.8.8.8
        search:
          - lab.local
```

7. Aplica la configuración:

```bash
sudo netplan apply
```

### Salida esperada

```
# En srv-linux:
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.100.10/24 brd 192.168.100.255 scope global enp0s3
    inet6 fd00::10/64 scope global
    inet6 fe80::..../64 scope link
```

### Verificación

```bash
# Desde srv-linux, verificar conectividad IPv4
ping -c 3 192.168.100.20

# Desde srv-linux, verificar conectividad IPv6
ping6 -c 3 fd00::20

# Desde cli-linux, verificar conectividad IPv4
ping -c 3 192.168.100.10

# Desde cli-linux, verificar conectividad IPv6
ping6 -c 3 fd00::10
```

Todas las pruebas deben mostrar `0% packet loss`.

---

## Paso 3 — Instalación de Herramientas de Diagnóstico

### Objetivo

Instalar todos los paquetes necesarios para las actividades de diagnóstico en ambas VMs.

### Instrucciones

1. En **ambas VMs** (si hay acceso a internet vía NAT en srv-linux, ejecuta primero allí y luego transfiere paquetes o configura proxy; si no hay internet, verifica que los paquetes ya estén instalados):

```bash
sudo apt update
sudo apt install -y openssh-server openssh-client tcpdump traceroute \
  mtr-tiny dnsutils nmap net-tools iputils-ping
```

2. Verifica la instalación:

```bash
which tcpdump traceroute mtr dig nmap ss
```

3. Habilita y arranca el servicio SSH en **srv-linux**:

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

### Salida esperada

```
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; ...)
     Active: active (running) since ...
```

### Verificación

```bash
ss -tlnp | grep :22
```

Debe mostrar `sshd` escuchando en el puerto 22.

---

## Paso 4 — Diagnóstico de Conectividad de Red (Capas 2-4)

### Objetivo

Inducir fallas de red controladas y diagnosticarlas sistemáticamente identificando la capa OSI afectada.

### Instrucciones

#### 4.1 — Falla de Capa 2: Interfaz caída

1. En **srv-linux**, desactiva la interfaz:

```bash
sudo ip link set enp0s3 down
```

2. Desde **cli-linux**, intenta hacer ping:

```bash
ping -c 3 -W 2 192.168.100.10
```

3. Observa el resultado: `Destination Host Unreachable` o `100% packet loss`.

4. Diagnóstico — verifica el estado del enlace desde srv-linux:

```bash
ip link show enp0s3
```

**Indicador de falla:** Estado `DOWN` en la interfaz → problema en **Capa 2 (Enlace)**.

5. Restaura la interfaz:

```bash
sudo ip link set enp0s3 up
# Espera unos segundos y verifica
ping -c 2 192.168.100.20
```

#### 4.2 — Falla de Capa 3: Ruta incorrecta

6. En **cli-linux**, agrega una ruta estática incorrecta hacia un destino simulado:

```bash
# Simula que para llegar a 10.0.0.0/8 debe ir por un gateway inexistente
sudo ip route add 10.0.0.0/8 via 192.168.100.99
```

7. Intenta alcanzar una IP en ese rango:

```bash
ping -c 3 -W 2 10.0.0.1
```

8. Diagnostica con `ip route`:

```bash
ip route get 10.0.0.1
```

**Indicador de falla:** El gateway `192.168.100.99` no existe → problema en **Capa 3 (Red/Enrutamiento)**.

9. Corrige eliminando la ruta incorrecta:

```bash
sudo ip route del 10.0.0.0/8 via 192.168.100.99
```

#### 4.3 — Falla de Capa 4: Puerto cerrado/filtrado

10. Desde **cli-linux**, intenta conectar al puerto 8080 de srv-linux (no hay servicio):

```bash
# Timeout rápido con nmap
nmap -p 8080 192.168.100.10
```

11. Verifica con `ss` en srv-linux que no hay servicio en ese puerto:

```bash
ss -tlnp | grep 8080
```

**Indicador de falla:** Puerto `closed` o sin proceso escuchando → problema en **Capa 4 (Transporte/Servicio)**.

#### 4.4 — Captura con tcpdump

12. En **srv-linux**, inicia una captura en segundo plano:

```bash
sudo tcpdump -i enp0s3 -c 10 -w /tmp/captura_icmp.pcap icmp &
```

13. Desde **cli-linux**, genera tráfico ICMP:

```bash
ping -c 5 192.168.100.10
```

14. En **srv-linux**, espera que la captura termine y analiza:

```bash
sudo tcpdump -r /tmp/captura_icmp.pcap -nn
```

### Salida esperada (tcpdump)

```
reading from file /tmp/captura_icmp.pcap, link-type EN10MB (Ethernet)
12:00:01.000000 IP 192.168.100.20 > 192.168.100.10: ICMP echo request, id 1234, seq 1, length 64
12:00:01.000100 IP 192.168.100.10 > 192.168.100.20: ICMP echo reply, id 1234, seq 1, length 64
...
```

### Verificación

```bash
# Confirma que la conectividad está restaurada completamente
ping -c 2 192.168.100.10   # desde cli-linux
ping -c 2 192.168.100.20   # desde srv-linux
```

---

## Paso 5 — Diagnóstico de Rutas con traceroute y mtr

### Objetivo

Utilizar herramientas de trazado de rutas para visualizar el camino de los paquetes y diagnosticar problemas de enrutamiento.

### Instrucciones

1. Desde **cli-linux**, ejecuta `traceroute` hacia srv-linux:

```bash
traceroute 192.168.100.10
```

2. Ejecuta `mtr` en modo reporte (10 paquetes):

```bash
mtr -r -c 10 192.168.100.10
```

3. Agrega una ruta estática en **srv-linux** para simular un salto adicional y observa el efecto:

```bash
# En srv-linux, crea una ruta estática persistente para la red 172.16.0.0/16
sudo ip route add 172.16.0.0/16 via 192.168.100.20
```

4. Desde **srv-linux**, traza la ruta a 172.16.0.1:

```bash
traceroute -n 172.16.0.1
```

5. Observa que el primer salto es 192.168.100.20 (aunque no puede continuar más allá).

6. Limpia la ruta de prueba:

```bash
sudo ip route del 172.16.0.0/16 via 192.168.100.20
```

### Salida esperada (mtr)

```
HOST: cli-linux              Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 192.168.100.10        0.0%    10    0.5   0.6   0.4   1.2   0.3
```

### Verificación

```bash
ip route show
# Confirma que no quedan rutas residuales de prueba
```

---

## Paso 6 — Resolución de Problemas DNS

### Objetivo

Diagnosticar y resolver problemas de resolución de nombres usando `dig`, `nslookup` y `host`, interpretando diferentes tipos de registros.

### Instrucciones

#### 6.1 — Consultas DNS básicas (requiere acceso a internet en srv-linux)

1. Desde **srv-linux** (con NAT habilitado en adaptador 2), consulta registros DNS:

```bash
# Registro A (IPv4)
dig A google.com +short

# Registro AAAA (IPv6)
dig AAAA google.com +short

# Registro MX (correo)
dig MX google.com +short

# Registro CNAME
dig CNAME www.google.com +short

# Registro PTR (resolución inversa)
dig -x 8.8.8.8 +short
```

2. Usa `nslookup` para verificar:

```bash
nslookup google.com
nslookup -type=MX google.com
```

3. Usa `host` para una consulta rápida:

```bash
host google.com
host -t AAAA google.com
```

#### 6.2 — Simular falla DNS

4. En **cli-linux**, configura un servidor DNS inválido:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Cambia los nameservers a una IP inexistente:

```yaml
      nameservers:
        addresses:
          - 192.168.100.99
```

5. Aplica:

```bash
sudo netplan apply
```

6. Intenta resolver un nombre:

```bash
dig google.com +time=3
nslookup google.com
```

**Resultado esperado:** `connection timed out; no servers could be reached`

7. Diagnostica verificando el archivo de resolución:

```bash
cat /etc/resolv.conf
# Observa que apunta a 192.168.100.99 (inalcanzable)
```

8. Verifica que el problema NO es de conectividad L3:

```bash
ping -c 2 192.168.100.10
# Funciona → el problema es específicamente DNS (Capa 7/Aplicación)
```

#### 6.3 — Restaurar DNS

9. Corrige la configuración de Netplan:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Restaura:

```yaml
      nameservers:
        addresses:
          - 192.168.100.10
          - 8.8.8.8
```

10. Aplica y verifica:

```bash
sudo netplan apply
dig google.com +short
```

#### 6.4 — Configurar resolución local con /etc/hosts

11. En **ambas VMs**, agrega entradas de hosts locales:

```bash
# En srv-linux
echo "192.168.100.20 cli-linux cli-linux.lab.local" | sudo tee -a /etc/hosts

# En cli-linux
echo "192.168.100.10 srv-linux srv-linux.lab.local" | sudo tee -a /etc/hosts
```

### Verificación

```bash
# Desde cli-linux
ping -c 2 srv-linux
# Debe resolver a 192.168.100.10

# Desde srv-linux
ping -c 2 cli-linux
# Debe resolver a 192.168.100.20
```

---

## Paso 7 — Configuración de SSH con Autenticación Basada en Llaves RSA-4096

### Objetivo

Generar un par de llaves RSA de 4096 bits, distribuir la llave pública al servidor y configurar `ssh-agent` para gestión de passphrase.

### Instrucciones

#### 7.1 — Generación de llaves en cli-linux (cliente)

1. En **cli-linux**, genera el par de llaves:

```bash
ssh-keygen -t rsa -b 4096 -C "labuser@cli-linux" -f /home/labuser/.ssh/id_rsa
```

Cuando solicite passphrase, ingresa: `Lab@SSH2024!`

2. Verifica los archivos generados:

```bash
ls -la /home/labuser/.ssh/
```

**Salida esperada:**

```
-rw------- 1 labuser labuser 3401 ... id_rsa
-rw-r--r-- 1 labuser labuser  745 ... id_rsa.pub
```

3. Inspecciona la llave pública:

```bash
cat /home/labuser/.ssh/id_rsa.pub
```

#### 7.2 — Distribución de la llave pública a srv-linux

4. Copia la llave pública al servidor:

```bash
ssh-copy-id labuser@192.168.100.10
```

Ingresa la contraseña de `labuser` en srv-linux cuando se solicite.

5. Verifica que la llave se copió correctamente — en **srv-linux**:

```bash
cat /home/labuser/.ssh/authorized_keys
```

Debe contener la llave pública de cli-linux.

#### 7.3 — Conexión SSH con llave

6. Desde **cli-linux**, conéctate sin contraseña (pedirá passphrase de la llave):

```bash
ssh labuser@192.168.100.10
```

Ingresa la passphrase `Lab@SSH2024!` → acceso exitoso.

7. Sal de la sesión:

```bash
exit
```

#### 7.4 — Configuración de ssh-agent

8. Inicia `ssh-agent` y agrega la llave:

```bash
eval $(ssh-agent -s)
ssh-add /home/labuser/.ssh/id_rsa
```

Ingresa la passphrase una sola vez.

9. Verifica que la llave está cargada:

```bash
ssh-add -l
```

**Salida esperada:**

```
4096 SHA256:xxxxx... labuser@cli-linux (RSA)
```

10. Conéctate nuevamente — ahora sin solicitar passphrase:

```bash
ssh labuser@192.168.100.10 "hostname && uptime"
```

#### 7.5 — Hardening básico de sshd_config en srv-linux

11. En **srv-linux**, edita la configuración del servidor SSH:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
```

12. Modifica/agrega las siguientes directivas:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
Protocol 2
```

> **Nota:** Deshabilitar `PasswordAuthentication` solo después de confirmar que la autenticación por llave funciona correctamente.

13. Valida la configuración y reinicia:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

14. Prueba la conexión desde **cli-linux**:

```bash
ssh labuser@192.168.100.10 "echo 'SSH con llave funcionando correctamente'"
```

### Verificación

```bash
# Desde cli-linux, intenta conectar con contraseña (debe fallar)
ssh -o PubkeyAuthentication=no labuser@192.168.100.10
# Resultado esperado: Permission denied (publickey).
```

---

## Paso 8 — Transferencia Segura de Archivos con SCP y SFTP

### Objetivo

Transferir archivos entre hosts de forma segura y verificar la integridad mediante checksums SHA-256.

### Instrucciones

#### 8.1 — Preparación de archivos de prueba

1. En **srv-linux**, crea un directorio y archivos de prueba:

```bash
mkdir -p /home/labuser/transfer_test
dd if=/dev/urandom of=/home/labuser/transfer_test/datos_prueba.bin bs=1M count=5
echo "Archivo de configuración de ejemplo" > /home/labuser/transfer_test/config.txt
```

2. Genera checksums:

```bash
sha256sum /home/labuser/transfer_test/* > /home/labuser/transfer_test/checksums.sha256
cat /home/labuser/transfer_test/checksums.sha256
```

#### 8.2 — Transferencia con SCP

3. Desde **cli-linux**, descarga archivos de srv-linux:

```bash
mkdir -p /home/labuser/recibidos

# Copiar un archivo individual
scp labuser@192.168.100.10:/home/labuser/transfer_test/config.txt /home/labuser/recibidos/

# Copiar directorio completo recursivamente
scp -r labuser@192.168.100.10:/home/labuser/transfer_test/ /home/labuser/recibidos/
```

4. Sube un archivo desde cli-linux a srv-linux:

```bash
echo "Reporte desde cliente $(date)" > /home/labuser/reporte_cliente.txt
scp /home/labuser/reporte_cliente.txt labuser@192.168.100.10:/home/labuser/
```

#### 8.3 — Transferencia con SFTP

5. Desde **cli-linux**, inicia sesión SFTP interactiva:

```bash
sftp labuser@192.168.100.10
```

6. Dentro de la sesión SFTP, ejecuta:

```sftp
ls
cd transfer_test
ls
get datos_prueba.bin /home/labuser/recibidos/datos_sftp.bin
put /home/labuser/reporte_cliente.txt /home/labuser/reporte_sftp.txt
bye
```

#### 8.4 — Verificación de integridad

7. En **cli-linux**, verifica los checksums:

```bash
cd /home/labuser/recibidos/transfer_test
sha256sum -c checksums.sha256
```

**Salida esperada:**

```
datos_prueba.bin: OK
config.txt: OK
checksums.sha256: OK
```

8. Verifica el archivo descargado por SFTP:

```bash
sha256sum /home/labuser/recibidos/datos_sftp.bin
# Compara manualmente con el checksum original del archivo datos_prueba.bin
```

### Verificación

```bash
# Confirma que todos los archivos existen
ls -la /home/labuser/recibidos/transfer_test/
ls -la /home/labuser/recibidos/datos_sftp.bin

# En srv-linux, confirma la recepción del archivo subido
ssh labuser@192.168.100.10 "ls -la /home/labuser/reporte_cliente.txt /home/labuser/reporte_sftp.txt"
```

---

## Paso 9 — Configuración de Rutas Estáticas Persistentes

### Objetivo

Configurar rutas IP estáticas que persistan entre reinicios y diagnosticar problemas de enrutamiento.

### Instrucciones

1. En **srv-linux**, agrega una ruta estática persistente via Netplan para una red simulada:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Agrega bajo la sección `routes:`:

```yaml
    enp0s3:
      addresses:
        - 192.168.100.10/24
        - fd00::10/64
      routes:
        - to: default
          via: 192.168.100.1
        - to: 172.16.50.0/24
          via: 192.168.100.20
          metric: 100
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
        search:
          - lab.local
```

2. Aplica:

```bash
sudo netplan apply
```

3. Verifica la tabla de rutas:

```bash
ip route show
```

**Salida esperada (parcial):**

```
172.16.50.0/24 via 192.168.100.20 dev enp0s3 metric 100
192.168.100.0/24 dev enp0s3 proto kernel scope link src 192.168.100.10
```

4. Verifica con `ip route get`:

```bash
ip route get 172.16.50.1
```

**Salida esperada:**

```
172.16.50.1 via 192.168.100.20 dev enp0s3 src 192.168.100.10
```

5. Verifica persistencia reiniciando el servicio de red:

```bash
sudo systemctl restart systemd-networkd
ip route show | grep 172.16
```

### Verificación

La ruta `172.16.50.0/24 via 192.168.100.20` debe aparecer después del reinicio del servicio.

---

## Paso 10 — Escaneo de Puertos y Servicios con nmap

### Objetivo

Verificar qué puertos y servicios están accesibles en el servidor usando `nmap` y correlacionar con `ss`.

### Instrucciones

1. Desde **cli-linux**, escanea los puertos más comunes de srv-linux:

```bash
nmap -sT -p 22,80,443,3306,8080 192.168.100.10
```

2. Realiza un escaneo de detección de servicios:

```bash
nmap -sV -p 22 192.168.100.10
```

3. Compara con la vista del servidor — en **srv-linux**:

```bash
ss -tlnp
```

4. Escanea puertos UDP (requiere privilegios):

```bash
sudo nmap -sU -p 53,123,161 192.168.100.10
```

5. Documenta los resultados:

```bash
# Desde cli-linux
nmap -sT -p 1-1024 192.168.100.10 -oN /home/labuser/scan_srv-linux.txt
cat /home/labuser/scan_srv-linux.txt
```

### Salida esperada

```
PORT   STATE  SERVICE
22/tcp open   ssh
80/tcp closed http
443/tcp closed https
...
```

### Verificación

El puerto 22 debe aparecer como `open` y el servicio identificado como `OpenSSH 8.9p1`.

---

## Validación Final y Pruebas

Ejecuta la siguiente secuencia de validación completa desde **cli-linux**:

```bash
echo "=== VALIDACIÓN DEL LABORATORIO 11 ==="
echo ""

echo "1. Verificar conectividad IPv4:"
ping -c 1 -W 2 192.168.100.10 > /dev/null 2>&1 && echo "   [OK] IPv4 a srv-linux" || echo "   [FALLO] IPv4 a srv-linux"

echo "2. Verificar conectividad IPv6:"
ping6 -c 1 -W 2 fd00::10 > /dev/null 2>&1 && echo "   [OK] IPv6 a srv-linux" || echo "   [FALLO] IPv6 a srv-linux"

echo "3. Verificar resolución de nombres local:"
ping -c 1 -W 2 srv-linux > /dev/null 2>&1 && echo "   [OK] Resolución srv-linux" || echo "   [FALLO] Resolución srv-linux"

echo "4. Verificar SSH con llave:"
ssh -o BatchMode=yes -o ConnectTimeout=5 labuser@192.168.100.10 "echo OK" 2>/dev/null && echo "   [OK] SSH con llave" || echo "   [FALLO] SSH con llave"

echo "5. Verificar archivos transferidos:"
[ -f /home/labuser/recibidos/transfer_test/config.txt ] && echo "   [OK] SCP transfer" || echo "   [FALLO] SCP transfer"

echo "6. Verificar integridad (checksums):"
cd /home/labuser/recibidos/transfer_test && sha256sum -c checksums.sha256 > /dev/null 2>&1 && echo "   [OK] Integridad verificada" || echo "   [FALLO] Integridad"

echo "7. Verificar ruta estática en srv-linux:"
ssh labuser@192.168.100.10 "ip route show | grep 172.16.50.0" > /dev/null 2>&1 && echo "   [OK] Ruta estática persistente" || echo "   [FALLO] Ruta estática"

echo ""
echo "=== FIN DE VALIDACIÓN ==="
```

**Resultado esperado:** Todos los puntos deben mostrar `[OK]`.

---

## Solución de Problemas

### Problema 1: SSH rechaza la conexión con "Permission denied (publickey)"

**Síntomas:** Al intentar conectar desde cli-linux a srv-linux con `ssh labuser@192.168.100.10`, se recibe el error `Permission denied (publickey).` incluso después de haber copiado la llave.

**Causa:** Los permisos del directorio `/home/labuser/.ssh/` o del archivo `authorized_keys` en srv-linux son demasiado permisivos. OpenSSH rechaza llaves si el directorio tiene permisos de escritura para grupo u otros.

**Solución:**

```bash
# En srv-linux, corrige permisos
chmod 700 /home/labuser/.ssh
chmod 600 /home/labuser/.ssh/authorized_keys
chown -R labuser:labuser /home/labuser/.ssh

# Verifica en los logs del servidor
sudo journalctl -u ssh -n 20 | grep -i "auth"

# Reinicia SSH
sudo systemctl restart ssh
```

---

### Problema 2: Netplan apply falla con error YAML

**Síntomas:** Al ejecutar `sudo netplan apply` se recibe un error como `Error in network definition: expected mapping` o `Invalid YAML`.

**Causa:** Indentación incorrecta en el archivo YAML. Netplan es estricto con la indentación (debe ser exactamente 2 espacios por nivel, sin tabulaciones).

**Solución:**

```bash
# Valida la sintaxis antes de aplicar
sudo netplan generate

# Si hay error, verifica indentación
cat -A /etc/netplan/00-installer-config.yaml | head -20
# Los tabuladores aparecen como ^I — deben reemplazarse por espacios

# Corrige reemplazando tabs por 2 espacios
sudo sed -i 's/\t/  /g' /etc/netplan/00-installer-config.yaml

# Alternativa: restaura el backup y reescribe
sudo cp /etc/netplan/00-installer-config.yaml.bak /etc/netplan/00-installer-config.yaml
sudo nano /etc/netplan/00-installer-config.yaml

# Aplica nuevamente
sudo netplan apply
```

---

## Limpieza

Ejecuta los siguientes comandos para dejar el entorno en un estado limpio pero conservando las configuraciones que serán utilizadas en el Laboratorio 12:

### Elementos a CONSERVAR (necesarios para Lab 12):
- Configuración de red Netplan en ambas VMs
- Llaves SSH en `/home/labuser/.ssh/`
- Archivo `authorized_keys` en srv-linux
- Configuración hardened de `/etc/ssh/sshd_config` en srv-linux
- Entradas en `/etc/hosts`

### Elementos a limpiar:

```bash
# En srv-linux
rm -f /tmp/captura_icmp.pcap
rm -rf /home/labuser/transfer_test
rm -f /home/labuser/reporte_cliente.txt
rm -f /home/labuser/reporte_sftp.txt

# En cli-linux
rm -rf /home/labuser/recibidos
rm -f /home/labuser/reporte_cliente.txt
rm -f /home/labuser/scan_srv-linux.txt

# Detener ssh-agent (se reiniciará en la próxima sesión si se necesita)
ssh-agent -k 2>/dev/null

# Eliminar la ruta estática de prueba (172.16.50.0/24) si no se necesita
# NOTA: Dejar si se usará en Lab 12, eliminar si no:
# sudo sed -i '/172.16.50.0/d' /etc/netplan/00-installer-config.yaml
# sudo netplan apply
```

---

## Resumen

En este laboratorio se completaron las siguientes actividades fundamentales:

| Actividad | Herramientas utilizadas | Resultado |
|---|---|---|
| Configuración de red estática IPv4/IPv6 | Netplan, `ip addr` | Interfaces configuradas y comunicándose |
| Diagnóstico por capas OSI | `ping`, `ip link`, `ip route`, `ss`, `nmap` | Identificación de fallas en capas 2, 3 y 4 |
| Captura y análisis de tráfico | `tcpdump` | Paquetes ICMP capturados y analizados |
| Trazado de rutas | `traceroute`, `mtr` | Visualización del camino de paquetes |
| Diagnóstico DNS | `dig`, `nslookup`, `host` | Registros A, AAAA, MX, CNAME, PTR interpretados |
| SSH con llaves RSA-4096 | `ssh-keygen`, `ssh-copy-id`, `ssh-agent` | Acceso sin contraseña configurado |
| Hardening SSH | `sshd_config` | Root login deshabilitado, solo autenticación por llave |
| Transferencia segura | `scp`, `sftp`, `sha256sum` | Archivos transferidos con integridad verificada |
| Rutas estáticas persistentes | Netplan `routes:` | Rutas sobreviven reinicio de servicio |

### Conceptos clave reforzados

- El diagnóstico sistemático de red sigue un enfoque **de abajo hacia arriba** (capa 2 → 3 → 4 → 7)
- `tcpdump` permite verificar qué tráfico realmente llega/sale de una interfaz
- SSH con llaves RSA-4096 + passphrase + `ssh-agent` proporciona seguridad y conveniencia
- La verificación de integridad con SHA-256 es esencial al transferir archivos críticos
- Netplan es la herramienta declarativa estándar para configuración de red persistente en Ubuntu 22.04

### Próximo laboratorio

El **Laboratorio 12** utilizará la infraestructura SSH y de red configurada aquí para implementar controles de acceso avanzados, hardening adicional del servidor y políticas de seguridad.

---

### Recursos adicionales

- [Netplan Reference Documentation](https://netplan.readthedocs.io/)
- [OpenSSH Manual Pages](https://www.openssh.com/manual.html)
- [tcpdump Tutorial and Primer](https://danielmiessler.com/study/tcpdump/)
- [Linux ip command cheat sheet - Red Hat](https://access.redhat.com/articles/ip-command-cheat-sheet)
- `man 5 netplan`, `man ssh_config`, `man sshd_config`, `man tcpdump`
