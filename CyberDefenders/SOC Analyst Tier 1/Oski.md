# Oski — CyberDefenders

## Información del lab

| Campo | Valor |
|---|---|
| Plataforma | CyberDefenders |
| Track | SOC Analyst Tier 1 — Level 1 |
| Categoría | Threat Intelligence |
| Dificultad | Easy |
| Herramienta | Any.Run (sandbox) |
| Tiempo estimado | 30 minutos |

---

## Escenario

Análisis de un reporte de sandbox de **Any.Run** para investigar el comportamiento del malware **Oski Stealer** — un infostealer que roba credenciales de navegadores, usa cifrado RC4, y se autoeliminma después de exfiltrar los datos.

---

## Tácticas MITRE ATT&CK involucradas

| Táctica | Técnica | Descripción |
|---|---|---|
| Collection | T1555 | Credenciales desde password stores (navegadores) |
| Command & Control | T1071 | Comunicación con servidor C2 |
| Defense Evasion | T1070.004 | Autoeliminación de archivos |
| Execution | T1059 | Ejecución de comandos vía cmd.exe |

---

## Análisis pregunta por pregunta

### Q1 — Tiempo de creación del malware

**Pregunta:** ¿Cuándo fue creado el malware?

**Respuesta:** `September 28, 2022, at 17:40:46 UTC`

Esta marca de tiempo refleja cuándo fue compilado el archivo malicioso. Comparar el tiempo de creación con la primera detección permite establecer cuánto tiempo evadió la detección el malware.

**¿Por qué importa?**
- Permite correlacionar con campañas conocidas de threat actors
- Ayuda a establecer una narrativa cronológica del incidente
- Indica si es una amenaza nueva o resurge una antigua

---

### Q2 — Servidor de Comando y Control (C2)

**Pregunta:** ¿Con qué servidor C2 se comunica el malware?

**Respuesta:** `171.22.28.221`

Esta IP aparece en múltiples patrones de memoria y está vinculada a URLs con rutas terminadas en `.php` y `.exe` — patrón típico de malware que descarga payloads adicionales.

**¿Qué es un servidor C2?**
Es el centro de control remoto desde donde el atacante envía instrucciones al malware — comandos de exfiltración, payloads adicionales, o instrucciones de autodestrucción.

**Acción defensiva:** bloquear o monitorear tráfico hacia esta IP en el firewall/SIEM.

---

### Q3 — Primera librería solicitada tras la infección

**Pregunta:** ¿Cuál es la primera librería que solicita el malware después de infectar el sistema?

**Respuesta:** `sqlite3.dll`

Esta librería se asocia con gestión de bases de datos. El malware la solicita para acceder a las bases de datos de los navegadores web donde se almacenan credenciales guardadas (contraseñas, cookies, tokens de sesión).

**¿Por qué sqlite3.dll?**
Los navegadores como Chrome y Firefox almacenan contraseñas en bases de datos SQLite locales. Al cargar esta librería, el malware se prepara para leer y exfiltrar esas credenciales.

---

### Q4 — Clave RC4 de descifrado

**Pregunta:** ¿Qué clave RC4 usa el malware para descifrar sus cadenas codificadas en Base64?

**Respuesta:** `5329514621441247975720749009`

El malware usa **RC4** para ofuscar sus strings de configuración (URLs del C2, rutas, comandos). Los descifra en tiempo de ejecución para evadir análisis estático.

**¿Por qué Base64 + RC4?**
- Base64 solo es codificación (reversible sin clave) → no es seguro solo
- RC4 añade cifrado real → sin la clave no puedes leer la configuración
- Juntos dificultan el análisis estático del malware

---

### Q5 — Técnica MITRE para robo de contraseñas

**Pregunta:** ¿Qué técnica MITRE principal usa el malware para robar contraseñas?

**Respuesta:** `T1555 — Credentials from Password Stores`

Esta técnica implica acceder a ubicaciones comunes de almacenamiento de contraseñas:
- Navegadores web (Chrome, Firefox, Edge)
- Gestores de contraseñas
- Keychain del sistema

---

### Q6 — Directorio objetivo para eliminación de DLLs

**Pregunta:** ¿A qué directorio apunta el malware para eliminar archivos DLL?

**Respuesta:** `C:\ProgramData\`

El malware ejecuta comandos de `cmd.exe` para eliminar:
- `C:\Users\admin\AppData\Local\Temp\VPN.exe` (el ejecutable principal)
- `C:\ProgramData\*.dll` (todas las DLLs cargadas durante la ejecución)

---

### Q7 — Tiempo de autoeliminación

**Pregunta:** ¿Cuántos segundos tarda el malware en autoeliminarse después de exfiltrar datos?

**Respuesta:** `5 segundos`

El malware programa su propia eliminación con un delay de 5 segundos para asegurarse de que la exfiltración se complete antes de borrar sus rastros.

---

## Flujo completo del ataque

```
1. Compilación
   └── Malware creado el 28/09/2022 17:40:46 UTC

2. Infección
   └── Víctima ejecuta el malware (disfrazado como VPN.exe)

3. Inicialización
   └── Descifra configuración con RC4
   └── Carga sqlite3.dll para acceder a bases de datos de navegadores

4. Command & Control
   └── Conecta a 171.22.28.221
   └── Descarga recursos adicionales (.php, .exe)

5. Robo de credenciales (T1555)
   └── Lee bases de datos SQLite de Chrome/Firefox
   └── Extrae contraseñas, cookies, tokens de sesión

6. Exfiltración
   └── Envía credenciales al servidor C2

7. Autoeliminación (después de 5 segundos)
   └── Elimina VPN.exe de AppData\Local\Temp
   └── Elimina *.dll de C:\ProgramData\
```

---

## Indicadores de Compromiso (IoCs)

| Tipo | Valor |
|---|---|
| IP C2 | 171.22.28.221 |
| Archivo malicioso | VPN.exe |
| Directorio temporal | C:\Users\admin\AppData\Local\Temp\ |
| Directorio DLLs | C:\ProgramData\ |
| Librería cargada | sqlite3.dll |
| Clave RC4 | 5329514621441247975720749009 |
| Fecha de compilación | 28/09/2022 17:40:46 UTC |

---

## Lecciones aprendidas para el Blue Team

1. **Monitorear carga de sqlite3.dll por procesos no autorizados**
2. **Alertar sobre conexiones salientes a IPs no conocidas**
3. **Detectar autoeliminación de archivos** — comandos `cmd.exe /c del` ejecutados poco después de la ejecución de un proceso son sospechosos
4. **Revisar C:\ProgramData\ y AppData\Temp** — rutas comunes de malware
5. **Implementar protección de credential stores** — no guardar contraseñas en navegadores

---

## Conceptos clave

- **Infostealer**: malware diseñado para robar credenciales y datos sensibles y exfiltrarlos a un C2.
- **Any.Run**: sandbox interactivo de análisis de malware que permite observar el comportamiento en tiempo real.
- **RC4**: cifrado de flujo simétrico usado por malware para ofuscar configuración.
- **sqlite3.dll**: librería SQLite usada por navegadores para almacenar credenciales localmente.
- **T1555**: técnica MITRE de robo de credenciales desde password stores.
- **Autoeliminación**: técnica de evasión donde el malware se borra a sí mismo tras completar su misión.
- **C2 (Command & Control)**: servidor del atacante que recibe datos exfiltrados y envía instrucciones.
