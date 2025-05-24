- Tags: #Tproot
---
[Maquina Tproot](https://mega.nz/file/ORUEzLia#WQgvveTv3kAnXBs6UyRShd1JomGNg6Sk7DSa_fJwD7k) ==> Enlace de descarga al laboratorio
```bash
7z x tproot.zip # Descomoprimimos el archivo
 sudo bash auto_deploy.sh tproot.tar # Desplegamos el laboratorio
```
---

#### Fase Reconocimiento:
- **( 172.17.0.2 )** ==> Direccion IP del **target**
```bash
 ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para ver si tenemos conectiviadad con el target

wichSystem.py 172.17.0.2 # Detectamos que estamos ante un sistema Linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos.
extractPorts allPorts # Parseamos la informacion mas relevante de el primer escaneo
 nmap -sC -sV -p21,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhauxtivo para determianr el servicio y la version que coren detras de estos puertos.
 nmap -sC -sV --script ftp-anon.nse -p21 172.17.0.2 # Para el usuari anonymous obtuvimos un codigo de estado 500.
nmap -p80 --script http-enum 172.17.0.2 # No reporta nada critico
```
---
- **( 21/tcp open  ftp     vsftpd 2.3.4 )** ==> Tenemos un servicio **FTP** desactualizada, ya que la version mas reciente, es: **( 3.69.1 )**
- **( ftp-anon: got code 500 "OOPS: cannot change directory:/var/ftp". )** ==>Tendremos que verificar si existe el usuario **Anonymous** activado para este servicio.
- **( 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) )** ==> Tenemos el servicio apache desplegado en el puerto 80
- **( Apache httpd 2.4.58 )**==> Por el **codeName** en **launchPad** determinamos que estamos ante un **ubuntuNoble**

#### Fase Reconocimiento Web:
- **( http://172.17.0.2/ )** ==> Accedemos al servicio pero solo tenemos el incio de la pagina de apache: **( index.html )**
```bash
whatweb http://172.17.0.2 # Realizamos reconocimiento de tecnologias, asi mismo wappalyzer que usa codigo php.
http://172.17.0.2 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], Title[Apache2 Ubuntu Default Page: It works]
```
---
- **Nota** ==> De la parte web, Por reconocimiento de rutas, directorios o subdominios, no encontramos nada que nos pueda dar informacion sobre un posible vector de ataque.

#### Fase Reconocimiento Servicio FTP:
- **Nota** ==> Despues de enumerar este servicio, Tendremos que usar un exploit para poder ganar acceso al **target**, Para este caso usamos **Metasploit** para ganar acceso al target, 
- Buscamos un exploit basado en la version del servicio: **( vsftpd 2.3.4 )** configuramos el payload y lo ejecutamos con **run**
- Obtendremos una consola como **root**
---