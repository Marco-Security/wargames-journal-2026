# Amadey APT-C-36 — CyberDefenders

## Información del lab

| Campo | Valor |
|---|---|
| Plataforma | CyberDefenders |
| Track | SOC Analyst Tier 1 — Level 1 |
| Categoría | Endpoint Forensics |
| Dificultad | Easy |
| Herramienta | Volatility3 |
| Tiempo estimado | 30 minutos |

---

## Escenario

Una alerta fuera de horario del sistema EDR detecta actividad sospechosa en una estación de trabajo Windows. Se proporciona un volcado de memoria (memory dump) para análisis forense. El objetivo es descubrir la actividad del **Amadey Trojan Stealer** — un malware modular especializado en reconocimiento, recolección de datos, robo de credenciales y comunicación con servidores C2.

---

## Tácticas MITRE ATT&CK

| Táctica | Técnica | Descripción |
|---|---|---|
| Defense Evasion | T1036 | Masquerading — proceso lssass.exe imita lsass.exe |
| Execution | T1218.011 | rundll32.exe para ejecutar DLLs maliciosas |
| Command & Control | T1071.001 | Comunicación HTTP con servidor C2 |
| Persistence | T1053.005 | Tarea programada en Windows Task Scheduler |
| Collection | T1555 | Robo de credenciales (cred64.dll) |

---

## Herramienta principal: Volatility3

**Volatility3** es un framework de análisis forense de memoria. Permite examinar volcados de RAM para identificar procesos activos, conexiones de red, archivos en memoria y artefactos de malware — incluso cuando el malware opera en memoria sin escribir archivos en disco.

### Plugins usados en este lab

| Plugin | Función |
|---|---|
| `windows.pstree` | Listar procesos en árbol (PID, PPID, relaciones) |
| `windows.cmdline` | Ver argumentos de línea de comandos por PID |
| `windows.netscan` | Escanear conexiones de red activas y cerradas |
| `windows.memmap` | Extraer regiones de memoria de un proceso |
| `windows.filescan` | Escanear archivos en memoria |

---

## Análisis pregunta por pregunta

### Q1 — Proceso principal malicioso

**Pregunta:** ¿Cuál es el proceso principal que desencadenó el comportamiento malicioso?

**Respuesta:** `lssass.exe` (PID 2748)

```bash
vol.py -f Windows\ 7\ x64-Snapshot4.vmem windows.pstree.PsTree
```

Al listar los procesos se observan dos entradas similares:
- `lsass.exe` (PID 508) → proceso legítimo de Windows (Local Security Authority)
- `lssass.exe` (PID 2748) → proceso malicioso con una `s` extra

**Técnica: Masquerading (T1036)**
El malware nombra su proceso casi igual al proceso legítimo del sistema para pasar desapercibido. `lsass.exe` es crítico para la autenticación en Windows — el atacante lo imita precisamente porque los analistas menos experimentados pueden ignorarlo.

`lssass.exe` además tiene un proceso hijo: `rundll32.exe` (PID 3064) — señal adicional de actividad maliciosa.

---

### Q2 — Ubicación del proceso malicioso

**Pregunta:** ¿Dónde está alojado el proceso malicioso en la estación de trabajo?

**Respuesta:** `C:\Users\0XSH3R~1\AppData\Local\Temp\925e7e99C5\lssass.exe`

```bash
vol.py -f Windows\ 7\ x64-Snapshot4.vmem windows.cmdline --pid 2748
```

**Señales de alerta:**
- Ubicado en `AppData\Local\Temp` — carpeta temporal accesible sin privilegios admin, favorita del malware
- Nombre de carpeta con hash aleatorio (`925e7e99C5`) — generación dinámica para evadir detección
- El `0XSH3R~1` es notación corta 8.3 de Windows para nombres largos

---

### Q3 — Servidor C2

**Pregunta:** ¿Cuál es la IP del servidor de Comando y Control?

**Respuesta:** `41.75.84.12` (puerto 80)

```bash
vol.py -f Windows\ 7\ x64-Snapshot4.vmem windows.netscan | grep 2748
```

El proceso `lssass.exe` (PID 2748) tiene conexiones cerradas hacia `41.75.84.12:80`. Usar el puerto 80 (HTTP) disfraza el tráfico malicioso como tráfico web normal para evadir firewalls.

---

### Q4 — Archivos descargados del C2

**Pregunta:** ¿Cuántos archivos distintos intenta descargar el malware?

**Respuesta:** `2 archivos`

```bash
vol.py -f Windows\ 7\ x64-Snapshot4.vmem windows.memmap --pid 2748 --dump
strings pid.2748.dmp | grep "GET /"
```

Las solicitudes HTTP GET encontradas en memoria:
```
GET /rock/Plugins/cred64.dll HTTP/1.1
GET /rock/Plugins/clip64.dll HTTP/1.1
```

| Archivo | Función probable |
|---|---|
| `cred64.dll` | Robo de credenciales |
| `clip64.dll` | Monitoreo del portapapeles (clipboard) |

Diseño modular — el malware descarga plugins según los objetivos del atacante.

---

### Q5 — Ruta de almacenamiento de los archivos descargados

**Pregunta:** ¿Cuál es la ruta completa del archivo descargado?

**Respuesta:** `\Users\0xSh3rl0ck\AppData\Roaming\116711e5a2ab05\clip64.dll`

```bash
vol.py -f Windows\ 7\ x64-Snapshot4.vmem windows.filescan | grep -E "cred64|clip64"
```

**Señales de alerta:**
- `AppData\Roaming` — escribible sin privilegios admin, frecuentemente excluido de antivirus
- Nombre de carpeta aleatorio (`116711e5a2ab05`) — generación dinámica para dificultar limpieza

---

### Q6 — Proceso hijo para ejecutar las DLLs

**Pregunta:** ¿Qué proceso hijo usa el malware para ejecutar los archivos DLL?

**Respuesta:** `rundll32.exe` (PID 3064)

`rundll32.exe` es una herramienta legítima de Windows para cargar y ejecutar DLLs. Los atacantes la abusan porque:
- Es un binario de confianza del sistema (Living off the Land)
- Su actividad puede eludir herramientas básicas de monitoreo
- Permite ejecutar DLLs maliciosas sin necesitar un ejecutable independiente

**Técnica: T1218.011 — Signed Binary Proxy Execution: Rundll32**

---

### Q7 — Mecanismo de persistencia adicional

**Pregunta:** ¿Dónde más garantiza el malware su presencia constante?

**Respuesta:** `\Windows\System32\Tasks\lssass.exe`

```bash
vol.py -f Windows\ 7\ x64-Snapshot4.vmem windows.filescan | grep -E "lssass"
```

Se encuentran dos ubicaciones:
1. `\Users\0XSH3R~1\AppData\Local\Temp\925e7e99C5\lssass.exe` → punto de entrega inicial
2. `\Windows\System32\Tasks\lssass.exe` → **persistencia via Task Scheduler**

El malware se registra como tarea programada en Windows Task Scheduler para ejecutarse automáticamente al reiniciar el sistema — técnica sigilosa porque abusa de un componente legítimo de Windows.

**Técnica: T1053.005 — Scheduled Task**

---

## Flujo completo del ataque

```
1. Entrega
   └── lssass.exe ejecutado desde AppData\Local\Temp\925e7e99C5\

2. Masquerading
   └── Nombre similar a lsass.exe (proceso legítimo del sistema)

3. Command & Control
   └── Conexiones HTTP a 41.75.84.12:80
   └── Descarga de plugins: cred64.dll y clip64.dll

4. Ejecución de plugins
   └── rundll32.exe carga las DLLs maliciosas
   └── cred64.dll → robo de credenciales
   └── clip64.dll → monitoreo del portapapeles

5. Persistencia
   └── Tarea programada en System32\Tasks\lssass.exe
   └── Sobrevive reinicios del sistema
```

---

## Indicadores de Compromiso (IoCs)

| Tipo | Valor |
|---|---|
| Proceso malicioso | lssass.exe |
| PID | 2748 |
| Ruta del ejecutable | C:\Users\0XSH3R~1\AppData\Local\Temp\925e7e99C5\lssass.exe |
| IP C2 | 41.75.84.12 |
| Puerto C2 | 80 (HTTP) |
| Plugin 1 | cred64.dll |
| Plugin 2 | clip64.dll |
| Ruta plugins | AppData\Roaming\116711e5a2ab05\ |
| Persistencia | \Windows\System32\Tasks\lssass.exe |

---

## Lecciones aprendidas para el Blue Team

1. **Revisar procesos con nombres similares a los del sistema** — una sola letra diferente puede ser masquerading
2. **Monitorear AppData\Temp y AppData\Roaming** — ubicaciones favoritas del malware
3. **Alertar sobre rundll32.exe ejecutado por procesos no autorizados** — abuso de Living off the Land
4. **Auditar tareas programadas regularmente** — `schtasks /query` o revisar `System32\Tasks`
5. **Bloquear conexiones HTTP salientes no autorizadas** — especialmente a IPs no categorizadas

---

## Herramientas utilizadas

- **Volatility3** — framework de análisis forense de memoria
- **strings** — extracción de texto legible de volcados de memoria
- **grep** — filtrado de resultados por patrones

---

## Conceptos clave

- **Memory Dump (volcado de memoria)**: captura del contenido de la RAM en un momento específico. Permite analizar procesos activos, conexiones y artefactos de malware en tiempo de ejecución.
- **Masquerading (T1036)**: técnica donde el malware imita nombres de procesos legítimos del sistema para pasar desapercibido.
- **Living off the Land (LotL)**: abuso de herramientas legítimas del sistema operativo (como rundll32.exe) para ejecutar código malicioso y evadir detección.
- **Amadey Trojan Stealer**: malware modular especializado en robo de credenciales, reconocimiento y comunicación C2. Diseño basado en plugins descargables.
- **Task Scheduler**: servicio de Windows para automatizar tareas. Frecuentemente abusado por malware para garantizar persistencia tras reinicios.
- **Volatility3**: framework open-source de análisis forense de memoria. Soporta múltiples sistemas operativos y proporciona plugins modulares para análisis profundo.
