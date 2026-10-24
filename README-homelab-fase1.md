# Homelab — Fase 1: Ubuntu Server con acceso remoto

> Laboratorio de sistemas montado en casa para practicar administración de servidores
> Linux y Windows en un entorno propio, en paralelo al ciclo de ASIR.

**Fecha:** 24/10/2026
**Autor:** Martín De Prada — Soporte IT Deskside

---

## Objetivo

Montar un entorno de laboratorio donde poder administrar sistemas sin las limitaciones de
permisos de un puesto de soporte de nivel 1, y que además sirviera de apoyo práctico al
ciclo de ASIR.

El requisito que condicionó todo el diseño fue **el acceso remoto**: paso varios días a la
semana fuera de casa, así que un laboratorio al que solo pudiera entrar sentado delante del
PC habría quedado sin usar. Por eso la fase 1 no termina con el servidor instalado, sino con
el servidor accesible desde cualquier sitio.

---

## Arquitectura

```mermaid
graph TB
    INT(("Internet"))

    subgraph CASA["Casa · Yepes — LAN 192.168.1.0/24"]
        R["Router<br/>192.168.1.1<br/>DHCP + Gateway"]
        subgraph HOST["PC anfitrión — Windows 11 Pro"]
            HV["Hyper-V<br/>hipervisor tipo 1"]
            VSW["Conmutador virtual externo<br/>(puente sobre NIC Realtek)"]
            VM["ubuntusrv<br/>Ubuntu Server 26.04 LTS<br/>eth0 · 192.168.1.240/24<br/>OpenSSH :22"]
        end
        R --- HOST
        HV --- VSW
        VSW --- VM
    end

    subgraph FUERA["Fuera de casa"]
        MOV["Pixel 10a<br/>Termius · datos móviles"]
    end

    R --- INT
    MOV --- INT
    VM <-. "Tailscale · WireGuard<br/>100.83.169.115" .-> MOV
```

Las líneas continuas son red física; la discontinua es el túnel cifrado de Tailscale. El
acceso desde fuera no pasa por el router: sale por el túnel, y por eso no hay ningún puerto
abierto hacia internet.

| Componente | Detalle |
|---|---|
| Anfitrión | Windows 11 Pro · Ryzen 7 5700X · 16 GB RAM · Gigabyte B550M K (BIOS F7) |
| Hipervisor | Hyper-V (tipo 1) |
| Máquina virtual | Generación 2 (UEFI) · 4 vCPU · 4 GB RAM fijos · disco VHDX de 60 GB |
| Sistema invitado | Ubuntu Server 26.04.1 LTS |
| Red virtual | Conmutador externo en puente sobre la NIC Realtek PCIe GbE |
| Direccionamiento | 192.168.1.240/24 · gw 192.168.1.1 · DNS 192.168.1.1 y 1.1.1.1 |
| Acceso local | OpenSSH, puerto 22 |
| Acceso remoto | Tailscale (WireGuard) · 100.83.169.115 |

---

## Implantación

### 1. Preparación del anfitrión

Activación de la característica de Hyper-V desde PowerShell con privilegios de
administrador:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

Hyper-V es un hipervisor de **tipo 1**: no se ejecuta sobre Windows como una aplicación más,
sino que se instala por debajo y toma el control directo del hardware. El propio Windows
pasa a ejecutarse como una partición privilegiada sobre él. Esa diferencia respecto a los
hipervisores de tipo 2, como VirtualBox, explica dos cosas que aparecen más adelante: que la
capa se cargue únicamente durante el arranque, y que los cambios requieran un apagado
completo y no un simple reinicio.

### 2. Red virtual

Creación de un conmutador virtual **externo** asociado al adaptador físico Realtek PCIe GbE,
con la opción de compartir el adaptador con el sistema anfitrión activada.

Hyper-V ofrece tres tipos de conmutador:

| Tipo | Alcance | Uso previsto |
|---|---|---|
| Externo | VM, anfitrión y red física | Esta fase: la VM necesita ser un equipo más de la LAN |
| Interno | VM y anfitrión | Laboratorios sin salida a internet |
| Privado | Solo entre VM | Fase 2: simular una red de empresa aislada |

Con el conmutador externo, la máquina virtual recibe su propia MAC (del rango `00:15:5D`,
reservado por Microsoft para Hyper-V) y el router la trata como un equipo físico más,
asignándole dirección por DHCP.

### 3. Creación e instalación de la máquina virtual

Máquina de **generación 2**, que emula firmware UEFI en lugar de una BIOS heredada. Esto
obliga a un ajuste que, de no hacerse, impide el arranque: por defecto el arranque seguro
solo confía en cargadores firmados por la autoridad certificadora de Windows, mientras que
el cargador de Ubuntu está firmado por la **Entidad de certificación UEFI de Microsoft**. Hay
que seleccionar esa plantilla en la configuración de seguridad de la VM.

Instalación de Ubuntu Server 26.04 LTS con las siguientes decisiones:

- **Idioma del sistema en inglés**, que es el estándar en entornos de servidor y el idioma en
  que están escritos los mensajes de error y la documentación.
- **Distribución de teclado en inglés de Estados Unidos**. Decisión forzada por un problema
  práctico documentado más abajo.
- **Instalación de OpenSSH server** durante el proceso, para disponer de acceso remoto desde
  el primer arranque.
- Sin paquetes snap adicionales.

### 4. Configuración de red estática

La dirección asignada por DHCP cambió en cada reinstalación y reinicio (`.40`, `.46`, `.33`),
lo que rompía el acceso por SSH. Se fijó mediante **netplan**, el sistema de configuración de
red declarativa de Ubuntu, que lee los ficheros `.yaml` de `/etc/netplan/` y genera a partir
de ellos la configuración efectiva.

Fichero `/etc/netplan/01-static.yaml`:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.1.240/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1, 1.1.1.1]
```

```bash
sudo chmod 600 /etc/netplan/01-static.yaml
sudo netplan apply
```

Dos decisiones de diseño:

- **Se eligió una dirección alta (`.240`)** porque los routers domésticos reparten por DHCP un
  rango de direcciones bajas. Asignar manualmente una dirección dentro de ese rango puede
  provocar que el router se la entregue también a otro equipo, generando un conflicto de
  direcciones difícil de diagnosticar.
- **Permisos `600`** porque netplan advierte si los ficheros son legibles por otros usuarios:
  pueden contener credenciales de redes inalámbricas.

Verificación posterior siguiendo el orden estándar de diagnóstico de red:

```bash
ip a                  # ¿tengo dirección?
ping -c 4 1.1.1.1     # ¿tengo salida por IP? (valida la puerta de enlace)
ping -c 4 google.com  # ¿resuelvo nombres? (valida el DNS)
```

### 5. Acceso remoto con Tailscale

La dirección `192.168.1.240` es privada y solo existe dentro de la LAN. Desde el exterior
únicamente es visible la IP pública del router, que además descarta por defecto las
conexiones entrantes no solicitadas.

La alternativa habitual sería abrir el puerto 22 en el router y redirigirlo a la máquina,
pero eso expone un servicio SSH a internet y a los intentos de acceso automatizados que
recibe cualquier IP pública. En su lugar se optó por **Tailscale**, una VPN de malla basada
en WireGuard:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4
```

Cada dispositivo autenticado contra la misma cuenta recibe una dirección del rango `100.x`,
válida únicamente dentro de esa red privada. Las conexiones se establecen **salientes** desde
cada extremo hacia los servidores de coordinación, de modo que no hace falta abrir ningún
puerto ni exponer nada.

Se desactivó además la **caducidad de clave** del nodo. Por defecto las claves expiran a los
seis meses, lo cual tiene sentido en un portátil pero no en un servidor: el equipo quedaría
desconectado sin aviso.

Validación final: conexión por SSH desde un Pixel 10a con Termius, **con el WiFi desactivado
y saliendo por datos móviles**, para confirmar que el acceso funcionaba realmente a través
del túnel y no por estar en la misma red local.

---

## Incidencias resueltas

### 1. La máquina virtual no arranca: «el hipervisor no está en ejecución»

**Síntoma.** Al iniciar la VM, Hyper-V devuelve el error *La máquina virtual no se pudo
iniciar porque el hipervisor no está en ejecución*, con sugerencias sobre la virtualización
asistida por hardware.

**Diagnóstico.** Se descartó primero la configuración del gestor de arranque:

```powershell
bcdedit /enum {current} | findstr -i hypervisor
```

El comando no devolvió ninguna línea, lo que indicaba que `hypervisorlaunchtype` no estaba
definido y por tanto conservaba su valor por defecto. Descartada esa vía, se comprobó el
estado real del procesador en el Administrador de tareas:

```
Virtualización:            Deshabilitado
Compatibilidad con Hyper-V: Sí
```

**Causa.** La virtualización asistida por hardware (**SVM Mode** en terminología AMD,
equivalente a VT-x en Intel) estaba deshabilitada en la BIOS. Sin esas extensiones del
procesador, el hipervisor no puede cargarse.

**Solución.** Activación de `SVM Mode` en la BIOS, en *Tweaker → Advanced CPU Settings*, y
**apagado completo del equipo**, no reinicio. El hipervisor se carga únicamente durante el
arranque, y un reinicio caliente conserva parte del estado anterior.

**Aprendizaje.** La virtualización no es solo software: depende de un juego de instrucciones
del procesador que permite ejecutar sistemas invitados a velocidad prácticamente nativa en
lugar de emularlos. Y la distinción entre reiniciar y apagar deja de ser una formalidad
cuando se toca una capa que solo se inicializa en el arranque.

---

### 2. «Permission denied» al conectar por SSH con la contraseña correcta

**Síntoma.** Rechazo reiterado de la autenticación pese a introducir la contraseña correcta.
En la salida del cliente aparecía:

```
martin_@192.168.1.40's password:
Permission denied, please try again.
```

**Causa.** Error de sintaxis en el comando. Se había escrito `ssh @martin 192.168.1.40` en
lugar de `ssh martin@192.168.1.40`. Al no encontrar un usuario válido antes de la arroba, el
cliente SSH toma por defecto **el nombre del usuario local del sistema**, en este caso el
usuario de Windows `Martin_`, que no existe en el servidor Ubuntu.

**Solución.** Sintaxis correcta `usuario@host`, o alternativamente el parámetro explícito:

```bash
ssh -l martin 192.168.1.240
```

**Aprendizaje.** El mensaje de error contenía la respuesta desde el primer intento: el prompt
mostraba `martin_@`, con un guion bajo que no correspondía a ningún usuario del servidor.
Leer con atención lo que devuelve la herramienta ahorra más tiempo que repetir el intento.

---

### 3. El servicio SSH figura como «inactive (dead)» pero funciona

**Síntoma.** La comprobación del servicio devolvía un estado aparentemente erróneo:

```
ssh.service - OpenBSD Secure Shell server
   Loaded: loaded (/usr/lib/systemd/system/ssh.service; disabled; preset: enabled)
   Active: inactive (dead)
TriggeredBy: ● ssh.socket
```

**Causa.** No era un fallo. Las versiones recientes de Ubuntu emplean **activación por
socket**: systemd mantiene un socket a la escucha en el puerto 22 y arranca el servicio solo
cuando llega una conexión entrante. El servicio permanece detenido mientras nadie se conecta.

**Solución.** Ninguna. La clave estaba en la línea `TriggeredBy: ● ssh.socket`, con el punto
en verde indicando que el socket sí estaba activo. La comprobación definitiva es verificar
qué proceso escucha en el puerto:

```bash
ss -tln | grep :22
```

**Aprendizaje.** Un servicio detenido no siempre significa un servicio caído. La activación
por socket ahorra recursos en servicios de uso intermitente, y comprobar el puerto es más
fiable que comprobar el estado del servicio.

---

### 4. Fichero bloqueado por un proceso huérfano de nano

**Síntoma.** Al reabrir un fichero en edición tras una desconexión:

```
File /etc/netplan/01-static.yaml is being edited by root
(with nano 8.7.1, PID 10214); open anyway?
```

**Causa.** La sesión SSH anterior se interrumpió sin cerrar nano correctamente. El editor
había creado un fichero oculto de bloqueo (`.01-static.yaml.swp`) para impedir ediciones
simultáneas, y el proceso quedó huérfano al perder su terminal.

**Solución.**

```bash
ps aux | grep nano          # localizar el proceso
sudo kill 10214             # terminarlo
sudo rm -f /etc/netplan/.01-static.yaml.swp
```

**Aprendizaje.** Primera aproximación práctica a la gestión de procesos en Linux: cada
programa en ejecución es un proceso identificado por un PID, `kill` envía una señal de
terminación, y los ficheros cuyo nombre empieza por punto están ocultos. También queda claro
por qué los editores crean ficheros de bloqueo y qué ocurre cuando una sesión remota se corta
de forma abrupta.

---

### 5. El portapapeles de Hyper-V escribe caracteres incorrectos

**Síntoma.** Al utilizar *Portapapeles → Escribir texto del portapapeles* en la consola de la
máquina virtual, el comando `ss -tln | grep :22` llegaba al servidor convertido en
`DXDDDmm`.

**Causa.** La función de portapapeles de Hyper-V no transfiere texto: **simula pulsaciones de
teclado**. Si la distribución configurada en el sistema invitado no coincide con la que asume
el anfitrión al enviarlas, los caracteres se descolocan. El problema afecta sobre todo a
símbolos como `|`, `@` o `\`.

**Solución.** Reinstalación del sistema invitado con distribución de teclado **inglés de
Estados Unidos**. Es además la configuración habitual en servidores de entorno profesional.
De forma complementaria, el problema deja de ser relevante una vez se administra por SSH: el
cliente de terminal gestiona el portapapeles de forma nativa.

**Aprendizaje.** La consola del hipervisor no es un terminal: es un teclado y una pantalla
virtuales. Esa diferencia explica por qué copiar y pegar se comporta de forma distinta según
se trabaje por consola o por SSH, y refuerza el criterio general de administrar los
servidores en remoto y reservar la consola para emergencias de red.

---

### 6. Pérdida de estado al revertir un punto de control automático

**Síntoma.** Pulsar por error *Revertir* en el Administrador de Hyper-V devolvió la máquina a
un estado anterior, perdiendo la configuración aplicada.

**Causa.** Hyper-V crea **puntos de control automáticos** al iniciar cada máquina virtual, y
la acción de revertir restaura el último sin solicitar confirmación adicional.

**Solución.** Desactivación de los puntos de control automáticos en *Configuración →
Puntos de control*, de modo que solo existan los creados de forma deliberada.

**Aprendizaje.** Los puntos de control son una de las herramientas más útiles de un
laboratorio: permiten crear un estado conocido antes de una modificación arriesgada y volver
a él en segundos. Pero conviene que sean explícitos. Un punto de control que no se sabe que
existe es un riesgo; uno creado a propósito antes de tocar algo es una red de seguridad.

---

## Comandos de referencia

```bash
# Diagnóstico de red, en este orden
ip a                              # ¿tengo dirección?
ping -c 4 1.1.1.1                 # ¿tengo salida por IP?
ping -c 4 google.com              # ¿resuelvo nombres?

# Servicios
systemctl status ssh
systemctl status ssh.socket
ss -tln | grep :22                # qué escucha en el puerto 22

# Configuración de red
sudo nano /etc/netplan/01-static.yaml
sudo chmod 600 /etc/netplan/01-static.yaml
sudo netplan apply

# Procesos
ps aux | grep <proceso>
sudo kill <PID>

# Actualización del sistema
sudo apt update && sudo apt upgrade -y

# Tailscale
sudo tailscale up
tailscale ip -4
tailscale status
```

```powershell
# Anfitrión Windows
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
bcdedit /enum {current} | findstr -i hypervisor
ssh martin@192.168.1.240
```

---

## Relación con el ciclo de ASIR

| Contenido aplicado en el laboratorio | Unidad |
|---|---|
| Máquinas virtuales y arquitectura cliente-servidor | ISO · U1 |
| Gestores de arranque, UEFI y arranque seguro | ISO · U1 |
| Instalación de sistemas operativos y particionado | ISO · U1 |
| Usuarios, grupos locales y elevación de privilegios | ISO · U2 |
| Configuración del protocolo TCP/IP en un cliente de red | ISO · U2 |
| Resolución de nombres | ISO · U2 |
| Administración remota | ISO · U9 |
| Sistema de ficheros, usuarios y procesos en Linux | ISO · U10 |

---

## Siguientes fases

**Fase 1b — Consolidación del servidor Linux**
Autenticación por clave SSH en lugar de contraseña, cortafuegos con UFW, servidor web Nginx,
tareas programadas con cron y copias de seguridad.

**Fase 2 — Windows Server**
Controlador de dominio con Active Directory, DNS, DHCP, unidades organizativas, directivas
de grupo y un cliente Windows unido al dominio.

**Fase 3 — Integración**
Unión del servidor Ubuntu al dominio de Active Directory, de forma que autentique contra los
mismos usuarios que los equipos Windows.
