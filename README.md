# Infra-Hardening-IAM
Implementación de red Zero Trust (SDN), fortificación de acceso SSH (Ed25519) y ofuscación de credenciales en scripts de automatización mediante el principio de mínimo privilegio.
# Infra-Hardening & IAM: Zero Trust, SSH Ed25519 y Ocultación de Credenciales

**Rol:** System Administrator / SOC Analyst L1
**Tiempo de despliegue:** 2 Horas
**Tecnologías:** Linux (Termux/Ubuntu), Tailscale (SDN), OpenSSH, Samba, Bash Scripting.

## 🚨 El Problema / Vector de Riesgo
Despliegue de un nodo ARM en el *edge* (Termux) que requería acceso de administración remota y compartición de volúmenes en red local. La configuración por defecto suponía tres riesgos críticos de seguridad: 
1. Exposición de puertos estándar en la LAN.
2. Autenticación basada en contraseñas susceptibles a ataques de fuerza bruta.
3. Presencia de credenciales *hardcodeadas* (en texto plano) en scripts de montaje automáticos (`.bashrc`), lo que permitiría escalada de privilegios en caso de brecha local.

## 🛠️ Triaje y Solución (Hardening por Capas)

### Fase 1: Reducción de Superficie de Ataque (Zero Trust)
* En lugar de abrir puertos en el router o exponer el servicio a la LAN física (`0.0.0.0`), se desplegó **Tailscale (Mesh VPN)**. 
* El servicio SSH se configuró para escuchar exclusivamente en un puerto no estándar (ej. `5220`) y únicamente a través de la interfaz de red virtual generada por Tailscale, haciendo el nodo invisible para los escaneos de red tradicionales (Nmap) en la red local.

### Fase 2: Fortificación de Identidad (SSH IAM)
* Se abandonó el algoritmo RSA en favor de **Ed25519** (curva elíptica) por su mayor rendimiento y resistencia criptográfica.
* Tras la inyección de la clave pública en el nodo (`authorized_keys`), se modificó el archivo `$PREFIX/etc/ssh/sshd_config` para denegar categóricamente los accesos inseguros:

> PubkeyAuthentication yes
> PasswordAuthentication no
> PermitRootLogin no

### Fase 3: Securización de Credenciales en Scripts (Samba)
* Para automatizar el montaje del recurso SMB desde el servidor principal sin exponer contraseñas en el archivo `~/.bashrc`, se aplicó el principio de mínimo privilegio.
* Se creó un archivo de credenciales oculto (`.smb_credentials`) restringido a nivel de sistema de archivos. El usuario estándar puede leer/escribir; el resto del sistema tiene acceso denegado (0):

> chmod 600 ~/.smb_credentials

* Se refactorizó el alias de conexión para utilizar este archivo blindado en lugar del texto plano:

> alias conectar_nodo="sudo mount -t cifs //100.x.x.x/Storage /mnt/nodo -o credentials=/home/$USER/.smb_credentials,port=4445,uid=1000,gid=1000,vers=3.0"

## SOC
La presencia de credenciales en texto plano en scripts (`.bash_aliases`, `cron`, scripts en Python) es uno de los hallazgos más comunes en auditorías y un vector primario para el movimiento lateral de un atacante.

Esta arquitectura demuestra que la automatización operativa (SysAdmin) no tiene por qué estar reñida con los estándares de seguridad (SecOps). Al combinar un perímetro definido por software (SDN), autenticación criptográfica y restricción de lectura por DAC (Discretionary Access Control) con `chmod 600`, se sella la infraestructura frente a atacantes tanto externos como internos.
