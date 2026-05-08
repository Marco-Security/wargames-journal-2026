# WebStrike — CyberDefenders

## Información del lab

| Campo | Valor |
|---|---|
| Plataforma | CyberDefenders |
| Track | SOC Analyst Tier 1 — Level 1 |
| Categoría | Network Forensics |
| Dificultad | Easy |
| Herramienta | Wireshark |
| Tiempo estimado | 30 minutos |

---

## Escenario

> Un archivo sospechoso fue identificado en un servidor web de una empresa, levantando alarmas en la intranet. El equipo de desarrollo detectó la anomalía, sospechando actividad maliciosa. El equipo de red capturó el tráfico crítico y preparó un archivo PCAP para revisión.
>
> Tu tarea es analizar el PCAP para descubrir cómo apareció el archivo y determinar el alcance de la actividad no autorizada.

---

## Tácticas MITRE ATT&CK involucradas

| Táctica | ID | Descripción |
|---|---|---|
| Initial Access | T1190 | Exploit de aplicación web (upload de webshell) |
| Execution | T1059 | Ejecución de comandos vía webshell |
| Persistence | T1505.003 | Web shell como mecanismo de persistencia |
| Command & Control | T1571 | Puerto no estándar para C2 |
| Exfiltration | T1041 | Exfiltración vía canal C2 |

---

## Análisis del ataque

### Pregunta 1 — Geolocalización del atacante

Analizando el tráfico HTTP en Wireshark, se identifica la IP de origen de los requests maliciosos. Usando una herramienta de geolocalización de IPs se determina el país de origen del atacante.

**Técnica:** filtrar por tráfico HTTP y analizar la IP source de los requests sospechosos.

```
Filtro Wireshark: http.request
```

---

### Pregunta 2 — User-Agent del atacante

El atacante modificó su **User-Agent** para hacerse pasar por un navegador Firefox legítimo, intentando mezclarse con tráfico normal y evadir detección basada en firmas de herramientas conocidas (como curl, sqlmap, etc.).

**Técnica de evasión:** spoofing de User-Agent.

```
Filtro Wireshark: http.user_agent
```

**Lección:** los User-Agents pueden ser falsificados fácilmente. No son una fuente confiable de identificación de herramientas. Un SOC analyst debe correlacionar con otros indicadores.

---

### Pregunta 3 — Upload de la reverse shell

Filtrando los requests **POST** se identifican dos intentos de subida de un webshell:

**Intento 1 — Fallido:**
```
Archivo: image.php
Resultado: Bloqueado por el servidor (filtro de extensiones)
```

**Intento 2 — Exitoso:**
```
Archivo: image.jpg.php
Resultado: Subida exitosa
```

El servidor web filtraba extensiones `.php` directamente, pero no validaba extensiones dobles. Al agregar `.jpg` antes de `.php`, el archivo pasó el filtro pero el servidor lo interpretó como PHP ejecutable.

```
Filtro Wireshark: http.request.method == "POST"
```

**Técnica:** Double Extension Bypass — técnica clásica para evadir filtros de upload que validan solo la primera o última extensión del nombre de archivo.

---

### Pregunta 4 — Puerto de Command & Control

Una vez ejecutado el webshell, el atacante estableció una **reverse shell** — una conexión desde el servidor comprometido hacia la máquina del atacante. Analizando el tráfico de salida se identifica el puerto utilizado para esta comunicación.

**Reverse shell:** a diferencia de un bind shell (el atacante conecta al servidor), en una reverse shell el servidor inicia la conexión hacia el atacante. Esto evita firewalls que bloquean conexiones entrantes.

```
Filtro Wireshark: tcp.flags.syn == 1 && ip.src == [IP_servidor]
```

---

### Pregunta 5 — Archivo exfiltrado

A través del webshell y la reverse shell, el atacante ejecutó comandos para leer archivos sensibles del sistema. El archivo exfiltrado fue:

```
/etc/passwd
```

Este archivo contiene la lista de usuarios del sistema Linux — nombres de usuario, UIDs, GIDs, shells asignados y directorios home. Es uno de los primeros archivos que un atacante lee tras comprometer un sistema Linux para hacer reconocimiento de usuarios.

---

## Flujo completo del ataque

```
1. Reconocimiento
   └── Atacante escanea el servidor web
   └── Usa User-Agent falso (Firefox) para evadir detección

2. Initial Access
   └── Descubre formulario de upload de archivos
   └── Intento 1: sube image.php → BLOQUEADO
   └── Intento 2: sube image.jpg.php → EXITOSO

3. Execution
   └── Accede al webshell vía HTTP
   └── Ejecuta comandos en el servidor

4. Command & Control
   └── Lanza reverse shell al puerto del atacante
   └── Obtiene shell interactiva en el servidor

5. Exfiltration
   └── Lee /etc/passwd
   └── Exfiltra datos de usuarios del sistema
```

---

## Indicadores de Compromiso (IoCs)

| Tipo | Valor |
|---|---|
| IP atacante | (identificada en el PCAP) |
| User-Agent | Mozilla/Firefox (falso) |
| Archivo malicioso | image.jpg.php |
| Archivo exfiltrado | /etc/passwd |
| Técnica de bypass | Double Extension |

---

## Lecciones aprendidas

**Para el Blue Team:**

1. **Validar extensiones correctamente** — no solo la última extensión sino todas las partes del nombre de archivo. Usar whitelist de extensiones permitidas, no blacklist.
2. **Monitorear requests POST** — especialmente a endpoints de upload. Un POST seguido de un GET al mismo archivo es señal de webshell.
3. **User-Agent no es confiable** — correlacionar con patrones de comportamiento (frecuencia, payloads, timing).
4. **Detectar conexiones salientes inusuales** — una reverse shell genera una conexión TCP saliente desde el servidor hacia una IP externa en un puerto no estándar.
5. **Alertar sobre acceso a `/etc/passwd`** — cualquier proceso web que lea este archivo es sospechoso.

---

## Herramientas utilizadas

- **Wireshark** — análisis de tráfico de red (PCAP)
- **Filtros de display** — `http.request`, `http.request.method == "POST"`, `tcp.flags`
- **Geolocalización de IP** — identificar origen del atacante

---

## Conceptos clave

- **PCAP (Packet Capture)**: archivo que contiene el tráfico de red capturado. Permite reconstruir comunicaciones completas entre hosts.
- **Webshell**: script (PHP, ASP, JSP) subido a un servidor web que permite ejecutar comandos del sistema vía HTTP.
- **Reverse Shell**: conexión iniciada desde el sistema comprometido hacia el atacante. Evita firewalls que bloquean conexiones entrantes.
- **Double Extension Bypass**: técnica para evadir filtros de upload usando nombres como `archivo.jpg.php`. El servidor procesa la extensión final como PHP pero el filtro solo revisa `.jpg`.
- **User-Agent Spoofing**: modificar el campo User-Agent para simular un navegador legítimo y evadir detección.
- **/etc/passwd**: archivo Linux con información de usuarios del sistema. Primer objetivo de reconocimiento post-compromiso.
- **MITRE ATT&CK**: framework que categoriza las tácticas y técnicas usadas por atacantes en incidentes reales.
