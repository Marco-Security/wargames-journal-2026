# Yellow RAT — CyberDefenders

## Información del lab

| Campo | Valor |
|---|---|
| Plataforma | CyberDefenders |
| Track | SOC Analyst Tier 1 — Level 1 |
| Categoría | Threat Intelligence |
| Dificultad | Easy |
| Herramienta | VirusTotal |
| Tiempo estimado | 30 minutos |

---

## Escenario

GlobalTech Industries detecta tráfico de red anormal durante una verificación de seguridad rutinaria. El análisis inicial apunta a redirecciones de consultas de búsqueda e intentos de acceso no autorizado. La investigación se centra en identificar la causa raíz analizando artefactos maliciosos con herramientas de Threat Intelligence.

---

## Tácticas MITRE ATT&CK

| Táctica | Técnica | Descripción |
|---|---|---|
| Execution | T1059 | Ejecución de comandos en memoria |
| Persistence | T1547 | Mecanismos de persistencia |
| Command & Control | T1071 | Comunicación con servidor C2 |
| Exfiltration | T1041 | Exfiltración vía canal C2 |
| Defense Evasion | T1055 | Ejecución en memoria (fileless) |

---

## Análisis pregunta por pregunta

### Q1 — Familia de malware

**Pregunta:** ¿Cuál es la familia de malware que causó el tráfico anormal?

**Respuesta:** `Yellow Cockatoo`

Es un **RAT (Remote Access Trojan)** basado en .NET diseñado para:
- Ejecutar comandos directamente en memoria (fileless) → evita detección basada en disco
- Establecer conexiones con servidores C2
- Descargar payloads secundarios
- Recopilar información del host (OS, configuración de red, credenciales)

**¿Cómo se identificó?** Comparando el hash SHA256/MD5 de la muestra en VirusTotal, que lo correlacionó con la firma conocida de Yellow Cockatoo.

---

### Q2 — Nombre de archivo común del malware

**Pregunta:** ¿Cuál es el nombre de archivo asociado con el malware?

**Respuesta:** `111bc461-1ca8-43c6-97ed-911e0e69fdf8.dll`

Este archivo DLL fue marcado como malicioso por **59 de 72 proveedores** en VirusTotal. Las etiquetas incluyen:
- `trojan.msil/polazert`
- `Win32:MalwareX-gen`

Indica que funciona como un **dropper/loader** — descarga y ejecuta payloads adicionales.

**Uso como IoC:** este nombre de archivo puede usarse para escanear otras estaciones de trabajo con herramientas EDR o antivirus.

---

### Q3 — Timestamp de compilación

**Pregunta:** ¿Cuál es la fecha de compilación del malware?

**Respuesta:** `2020-09-24 18:26:47 UTC`

La fecha de compilación indica cuándo se construyó el ejecutable malicioso. Combinada con otras fechas:

| Evento | Fecha |
|---|---|
| Compilación | 2020-09-24 18:26:47 UTC |
| Primer envío a VirusTotal | 2020-10-15 02:47:37 UTC |
| Primera detección en la naturaleza | 2021-04-12 10:51:04 UTC |

El gap entre compilación y primera detección sugiere que el malware pasó tiempo en fase de prueba o distribución limitada antes de su despliegue masivo.

---

### Q4 — Primera vez en VirusTotal

**Pregunta:** ¿Cuándo fue enviado por primera vez a VirusTotal?

**Respuesta:** `2020-10-15 02:47:37 UTC`

Esta fecha representa el conocimiento público más temprano de la muestra. Permite a los analistas:
- Correlacionar con logs históricos de la red
- Identificar si el malware estuvo activo antes de ser detectado
- Estimar el tiempo de permanencia (dwell time) en el entorno

---

### Q5 — Archivo .dat en AppData

**Pregunta:** ¿Cuál es el nombre del archivo .dat que el malware colocó en AppData?

**Respuesta:** `solarmarker.dat`

**Ruta:** `%USERPROFILE%\AppData\Roaming\solarmarker.dat`

Este archivo puede usarse para:
- Almacenar datos de configuración del malware
- Registrar actividad del sistema
- Mantener persistencia

**¿Por qué AppData\Roaming?**
Es una carpeta accesible sin privilegios de administrador y usada por aplicaciones legítimas, lo que ayuda al malware a pasar desapercibido.

**Acciones de remediación:**
1. Escanear todos los endpoints buscando `solarmarker.dat`
2. Eliminar el archivo y entradas de registro relacionadas
3. Buscar tareas programadas que referencien este archivo
4. Aislar sistemas comprometidos y resetear credenciales

---

### Q6 — Servidor C2

**Pregunta:** ¿Cuál es el servidor C2 con el que se comunica el malware?

**Respuesta:** `gogohid.com` / subred `45.146.165.X`

El dominio `gogohid.com` actúa como centro de comando y control — recibe datos exfiltrados y envía instrucciones al malware. La subred `45.146.165.X` forma parte de la infraestructura del atacante.

**Acciones inmediatas:**
- Bloquear `gogohid.com` en el firewall y proxy
- Bloquear la subred `45.146.165.0/24` en el IPS
- Crear reglas de alerta en el SIEM para detectar conexiones a estos endpoints

---

## Línea de tiempo del ataque

```
2020-09-24  → Malware compilado
2020-10-15  → Primera muestra enviada a VirusTotal
2021-04-12  → Primera detección en la naturaleza
   ↓
Infección en GlobalTech Industries:
   └── DLL maliciosa descargada: 111bc461-...dll
   └── Ejecución en memoria (fileless)
   └── Archivo solarmarker.dat depositado en AppData\Roaming
   └── Conexión C2 a gogohid.com
   └── Exfiltración de datos del host
```

---

## Indicadores de Compromiso (IoCs)

| Tipo | Valor |
|---|---|
| Familia de malware | Yellow Cockatoo |
| Archivo DLL | 111bc461-1ca8-43c6-97ed-911e0e69fdf8.dll |
| Archivo DAT | solarmarker.dat |
| Ruta AppData | %USERPROFILE%\AppData\Roaming\ |
| Dominio C2 | gogohid.com |
| Subred C2 | 45.146.165.0/24 |
| Fecha compilación | 2020-09-24 18:26:47 UTC |

---

## Lecciones aprendidas para el Blue Team

1. **Monitorear AppData\Roaming** — ruta frecuente de malware por no requerir privilegios de administrador
2. **Bloquear dominios C2 conocidos** en firewall y proxy DNS
3. **Usar hashes como IoCs** — el hash SHA256 del DLL permite bloquear la ejecución con herramientas EDR
4. **Detectar ejecución en memoria (fileless)** — el malware que no escribe archivos ejecutables en disco es más difícil de detectar; usar EDR con capacidades de análisis de memoria
5. **Correlacionar timestamps** — la brecha entre compilación y primera detección indica posible dwell time prolongado

---

## Herramientas utilizadas

- **VirusTotal** — plataforma de análisis de malware que agrega resultados de 70+ motores antivirus. Permite analizar archivos, URLs, IPs y dominios sospechosos.
- **MITRE ATT&CK** — framework para mapear comportamiento del malware a técnicas conocidas

---

## Conceptos clave

- **RAT (Remote Access Trojan)**: malware que permite al atacante controlar remotamente el sistema infectado — ejecutar comandos, exfiltrar datos, descargar payloads adicionales.
- **Fileless malware**: malware que ejecuta código directamente en memoria sin escribir archivos en disco. Evade antivirus basados en firma de archivos.
- **Dropper/Loader**: malware cuya función principal es descargar e instalar otros componentes maliciosos.
- **Dwell time**: tiempo que el malware permanece activo en un entorno antes de ser detectado. El gap entre compilación y detección puede indicar dwell time prolongado.
- **IoC (Indicator of Compromise)**: evidencia de actividad maliciosa — hashes, IPs, dominios, nombres de archivo — usados para detectar y responder a amenazas.
- **VirusTotal**: herramienta fundamental de Threat Intelligence para verificar si un archivo, hash, IP o dominio es conocido como malicioso.
