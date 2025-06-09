
<h1> 🧠 Resolución de Máquinas DockerLabs | Ethical Hacking Journey</h1>

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

Este repositorio contiene mis resoluciones personales y anotaciones detalladas sobre las máquinas prácticas de la plataforma **DockerLabs**. Cada máquina está documentada con comandos utilizados, fases de reconocimiento, escalada de privilegios y explotación.
*Este proyecto es parte de mi proceso de aprendizaje en ciberseguridad.*

Repositorio con walkthroughs detallados de máquinas DockerLabs para práctica de ciberseguridad. Cada solución incluye:
- 🔍 Reconocimiento
- 🔍 Análisis de vulnerabilidades
- ⚡ Explotación paso a paso
- 🚀 Técnicas de escalado de privilegios
- 📌 Notas de aprendizaje

## 📜 Aviso Legal  
Este repositorio contiene técnicas y procedimientos de seguridad informática que deben usarse **exclusivamente en entornos controlados y con autorización explícita**. El autor no se hace responsable del uso incorrecto o ilegal de esta información.
**El autor no se responsabiliza por el uso de las tecnicas en entornos no éticos o ilegales**

---
## 🗂 Estructura
**Nota** Disponible un **( [Writeup Template](/00-Template.md) )** Este template incluye todas las secciones esenciales para un writeup completo y sigue el formato que yo uso para mis reportes, manteniendo la estructura de código, hallazgos y fases de ataque organizadas segun mi metodologia.

```txt
# Instrucciones de Uso:
1. Reemplazar los contenidos entre `[corchetes]` con la información específica de la máquina
2. Ajustar las secciones según las características del objetivo
3. Eliminar secciones no utilizadas
4. Agregar capturas de pantalla o outputs importantes donde sea necesario
5. Personalizar las lecciones aprendidas y recomendaciones
```

### Maquinas DockerLabs

| Dificultad: Muy Facil                                              | Dificultad: Facil V1                                                    | Dificultad: Facil V2                                               | Dificultad: Media V1                                       |
| ------------------------------------------------------------------ | ----------------------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------- |
| [01 Injection](01-DockerLabs/01-MyFacil/01-Injection.md)           | [01 Psycho](01-DockerLabs/02-Facil/01-Psycho.md)                        | [26 Balulero](01-DockerLabs/02-Facil/26-Balulero.md)               | [01 Dance-Samba](01-DockerLabs/03-Media/01-Dance-Samba.md) |
| [02 Trust](01-DockerLabs/01-MyFacil/02-Trust.md)                   | [02 PequenasMentirosas](01-DockerLabs/02-Facil/02-PequenasMentirosa.md) | [27 ChocolateLovers](01-DockerLabs/02-Facil/27-ChocolateLovers.md) | [02 Veneno](01-DockerLabs/03-Media/02-Veneno.md)           |
| [03 BreakMySSH](01-DockerLabs/01-MyFacil/03-BreakMySSH.md)         | [03 WhereIsMyWebShell](01-DockerLabs/02-Facil/03-WhereIsMyWebShell.md)  | [28 Redirection](01-DockerLabs/02-Facil/28-Redirection.md)         | [03-Apolo](01-DockerLabs/03-Media/03-Apolo.md)             |
| [04 FirstHacking](01-DockerLabs/01-MyFacil/04-FirstHacking.md)     | [04 Library](01-DockerLabs/02-Facil/04-Library.md)                      | [29 Pn](01-DockerLabs/02-Facil/29-Pn.md)                           | [04-Report](01-DockerLabs/03-Media/04-Report.md)           |
| [05 HedgeHog](01-DockerLabs/01-MyFacil/05-HedgeHog.md)             | [05 NodeClimb](01-DockerLabs/02-Facil/05-NodeClimb.md)                  | [30-Escolares](01-DockerLabs/02-Facil/30-Escolares.md)             | [05-Swiss](01-DockerLabs/03-Media/05-Swiss.md)             |
| [06 BorazuwarahCTF](01-DockerLabs/01-MyFacil/06-BorazuwarahCTF.md) | [06 ConsoleLog](01-DockerLabs/02-Facil/06-ConsoleLog.md)                | [31 AnonymousPingu](01-DockerLabs/02-Facil/31-AnonymousPingu.md)   | [06-Inclusion](01-DockerLabs/03-Media/06-Inclusion.md)     |
| [07 Vacaciones](01-DockerLabs/01-MyFacil/07-Vacaciones.md)         | [07 Candy](01-DockerLabs/02-Facil/07-Candy.md)                          | [32 File](01-DockerLabs/02-Facil/32-File.md)                       | [07 Collections](01-DockerLabs/03-Media/07-Collections.md) |
| [08 Tproot](01-DockerLabs/01-MyFacil/08-Tproot.md)                 | [08 Verdejo](01-DockerLabs/02-Facil/08-Verdejo.md)                      | [33 FindYourStyle](01-DockerLabs/02-Facil/33-FindYourStyle.md)     |                                                            |
| [09 Obsession](01-DockerLabs/01-MyFacil/09-Obsession.md)           | [09 Upload](01-DockerLabs/02-Facil/09-Upload.md)                        | [34 Whoiam](01-DockerLabs/02-Facil/34-Whoiam.md)                   |                                                            |
|                                                                    | [10 Move](01-DockerLabs/02-Facil/10-Move.md)                            | [35 Jenkhack](01-DockerLabs/02-Facil/35-Jenkhack.md)               |                                                            |
|                                                                    | [11 SecretJenkins](01-DockerLabs/02-Facil/11-SecretJenkins.md)          | [36 ByPassme](01-DockerLabs/02-Facil/36-ByPassme.md)               |                                                            |
|                                                                    | [12 Backend](01-DockerLabs/02-Facil/12-Backend.md)                      | [37 Amor](01-DockerLabs/02-Facil/37-Amor.md)                       |                                                            |
|                                                                    | [13 VulnVault](01-DockerLabs/02-Facil/13-VulnVault.md)                  | [38 AguaDeMayo](01-DockerLabs/02-Facil/38-AguaDeMayo.md)           |                                                            |
|                                                                    | [14 Allien](01-DockerLabs/02-Facil/14-Allien.md)                        | [39 Pressenter](01-DockerLabs/02-Facil/39-Pressenter.md)           |                                                            |
|                                                                    | [15 Reflection](01-DockerLabs/02-Facil/15-Reflection.md)                | [40 Los40Ladrones](01-DockerLabs/02-Facil/40-Los40Ladrones.md)     |                                                            |
|                                                                    | [16 Extraviado](01-DockerLabs/02-Facil/16-Extraviado.md)                | [41 Picadilly](01-DockerLabs/02-Facil/41-Picadilly.md)             |                                                            |
|                                                                    | [17 InternShip](01-DockerLabs/02-Facil/17-InternShip.md)                | [42 Winterfell](01-DockerLabs/02-Facil/42-Winterfell.md)           |                                                            |
|                                                                    | [18 AppiBAse](01-DockerLabs/02-Facil/18-AppiBAse.md)                    | [43 Pntopntobarra](01-DockerLabs/02-Facil/43-Pntopntobarra.md)     |                                                            |
|                                                                    | [19 Bicho](01-DockerLabs/02-Facil/19-Bicho.md)                          | [44 Paradise](01-DockerLabs/02-Facil/44-Paradise.md)               |                                                            |
|                                                                    | [20 BaluFood](01-DockerLabs/02-Facil/20-BaluFood.md)                    | [45 StellaJWT](01-DockerLabs/02-Facil/45-StellaJWT.md)             |                                                            |
|                                                                    | [21 Galeria](01-DockerLabs/02-Facil/21-Galeria.md)                      | [46 Dockerlabs](01-DockerLabs/02-Facil/46-Dockerlabs.md)           |                                                            |
|                                                                    | [22 WalkingCMS](01-DockerLabs/02-Facil/22-WalkingCMS.md)                | [47 Elevator](01-DockerLabs/02-Facil/47-Elevator.md)               |                                                            |
|                                                                    | [23 Mirame](01-DockerLabs/02-Facil/23-Mirame.md)                        | [48 PatriaQuerida](01-DockerLabs/02-Facil/48-PatriaQuerida.md)     |                                                            |
|                                                                    | [24 ShowTime](01-DockerLabs/02-Facil/24-ShowTime.md)                    | [49 WalkingDead](01-DockerLabs/02-Facil/49-WalkingDead.md)         |                                                            |
|                                                                    | [25 Pinguinazo](01-DockerLabs/02-Facil/25-Pinguinazo.md)                | [50 PkgPoison](01-DockerLabs/02-Facil/50-PkgPoison.md)             |                                                            |
