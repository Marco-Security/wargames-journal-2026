# Lespion — CyberDefenders

## Información del lab

| Campo | Valor |
|---|---|
| Plataforma | CyberDefenders |
| Track | SOC Analyst Tier 1 — Level 1 |
| Categoría | Threat Intelligence / OSINT |
| Dificultad | Easy |
| Herramientas | Sherlock, Google Images, GitHub |
| Tiempo estimado | 30 minutos |

---

## Escenario

Un compromiso de red resultó en una interrupción significativa. Los respondedores iniciales determinan que el ataque se originó desde una **cuenta de usuario interna** — posible insider threat. El objetivo es rastrear la huella digital del sospechoso usando técnicas OSINT para identificar credenciales expuestas, herramientas maliciosas y actividad en redes sociales.

---

## Tácticas MITRE ATT&CK

| Táctica | Técnica | Descripción |
|---|---|---|
| Credential Access | T1552.001 | Credenciales en archivos (API key y password en repositorio público) |
| Resource Development | T1583 | Infraestructura para cryptojacking (XMRig) |
| Reconnaissance | T1593.001 | OSINT en redes sociales |

---

## Análisis pregunta por pregunta

### Q1 — API Key en GitHub

**Pregunta:** ¿Cuál es la API key que el insider agregó a sus repositorios?

**Respuesta:** `aJFRaLHjMXvYZgLPwiJkroYLGRkNBW`

**Cómo se encontró:**
- Perfil GitHub: `EMarseille99`
- Repositorio: `Project-Build---Custom-Login-Page`
- Archivo: `Login Page.js` — historial de commits
- La variable `API Key` fue añadida directamente al código fuente

**¿Por qué es grave?**
Hardcodear API keys en código fuente público expone credenciales a cualquier persona. Puede usarse para acceder a servicios conectados, bases de datos o sistemas externos sin autorización.

---

### Q2 — Contraseña en texto plano

**Pregunta:** ¿Cuál es la contraseña en texto plano en el repositorio?

**Respuesta:** `PicassoBaguette99`

**Cómo se encontró:**
En el mismo archivo `Login Page.js`, el campo de contraseña contenía:
```
UGljYXNzb0JhZ3VldHRlOTk=
```

Decodificando Base64:
```bash
echo "UGljYXNzb0JhZ3VldHRlOTk=" | base64 -d
# Output: PicassoBaguette99
```

**Lección:** Base64 NO es cifrado — es solo codificación. Es completamente reversible sin clave. Almacenar credenciales así es equivalente a texto plano.

---

### Q3 — Herramienta de criptominería

**Pregunta:** ¿Qué herramienta de minería de criptomonedas usó el insider?

**Respuesta:** `XMRig`

El repositorio GitHub del insider tiene un fork de **XMRig** — software open-source para minar Monero (XMR). Es ampliamente usado en ataques de **cryptojacking**: instalar el software en máquinas comprometidas para minar criptomonedas usando los recursos de la víctima sin su conocimiento.

**Características de XMRig:**
- Mina Monero (XMR) — criptomoneda centrada en privacidad
- Compatible con algoritmos RandomX, CryptoNight, AstroBWT
- Optimizado para CPU y GPU
- Usado frecuentemente en malware por su eficiencia y anonimato

---

### Q4 — Sitio de gaming

**Pregunta:** ¿En qué sitio web de gaming tenía cuenta el insider?

**Respuesta:** `Steam`

**Herramienta usada: Sherlock**

```bash
sherlock EMarseille99
```

Entre los resultados devueltos por Sherlock aparece:
```
[+] Steam: https://steamcommunity.com/id/EMarseille99/
```

**¿Qué es Sherlock?**
Herramienta OSINT que busca un username en 300+ plataformas simultáneamente — redes sociales, foros, sitios de gaming, plataformas de desarrollo, etc.

---

### Q5 — Perfil de Instagram

**Pregunta:** ¿Cuál es el enlace al perfil de Instagram del insider?

**Respuesta:** `https://www.instagram.com/EMarseille99/`

También encontrado con Sherlock en el mismo scan de Q4.

---

### Q6 — País de vacaciones

**Pregunta:** ¿A dónde fue el insider de vacaciones?

**Respuesta:** `Singapore`

Visible en las fotos publicadas en el perfil de Instagram del objetivo.

---

### Q7 — Ciudad donde vive la familia

**Pregunta:** ¿Dónde vive la familia del insider?

**Respuesta:** `Dubai`

Dos fotos en el perfil de Instagram muestran Dubai, indicando que la familia reside ahí.

---

### Q8 — Ciudad de la oficina

**Pregunta:** ¿En qué ciudad está ubicada la oficina de la empresa? (archivo: office.jpg)

**Respuesta:** `Birmingham`

**Método:** Google Image Search (búsqueda inversa de imágenes) del archivo `office.jpg` revela que el edificio está en Birmingham, Reino Unido.

---

### Q9 — Estado de la cámara IP

**Pregunta:** ¿En qué estado se encuentra la cámara IP? (archivo: Webcam.png)

**Respuesta:** `Indiana`

**Método:** Google Image Search del archivo `Webcam.png` revela que la imagen corresponde a **Notre Dame** — ubicada en el estado de Indiana, USA.

---

## Flujo de la investigación OSINT

```
1. GitHub (EMarseille99)
   ├── API Key expuesta: aJFRaLHjMXvYZgLPwiJkroYLGRkNBW
   ├── Password en Base64: PicassoBaguette99
   └── Fork de XMRig (cryptojacking)

2. Sherlock (username: EMarseille99)
   ├── Steam: https://steamcommunity.com/id/EMarseille99/
   └── Instagram: https://www.instagram.com/EMarseille99/

3. Instagram (EMarseille99)
   ├── Vacaciones: Singapore
   └── Familia en: Dubai

4. Google Image Search
   ├── office.jpg → Oficina en Birmingham
   └── Webcam.png → Notre Dame, Indiana
```

---

## Indicadores de Compromiso (IoCs)

| Tipo | Valor |
|---|---|
| Username | EMarseille99 |
| API Key expuesta | aJFRaLHjMXvYZgLPwiJkroYLGRkNBW |
| Password expuesta | PicassoBaguette99 |
| Herramienta maliciosa | XMRig |
| Perfil Steam | steamcommunity.com/id/EMarseille99/ |
| Perfil Instagram | instagram.com/EMarseille99/ |

---

## Lecciones aprendidas para el Blue Team

1. **Monitorear repositorios públicos** — herramientas como GitLeaks o TruffleHog detectan automáticamente credenciales expuestas en GitHub
2. **Nunca hardcodear credenciales** — usar variables de entorno o gestores de secretos (AWS Secrets Manager, HashiCorp Vault)
3. **Base64 no es cifrado** — cualquier credencial en Base64 está esencialmente en texto plano
4. **OSINT como técnica defensiva** — buscar regularmente el nombre de la empresa y empleados en plataformas públicas para detectar exposición de información
5. **Detectar cryptojacking** — monitorear uso inusual de CPU/GPU, conexiones salientes a pools de minería conocidos, y presencia de XMRig en endpoints

---

## Herramientas utilizadas

- **Sherlock** — OSINT para búsqueda de usernames en 300+ plataformas
- **GitHub** — revisión de repositorios públicos y historial de commits
- **Google Image Search** — búsqueda inversa de imágenes para geolocalización
- **Base64 decode** — `echo "string" | base64 -d`

---

## Conceptos clave

- **Insider Threat**: amenaza proveniente de un empleado o persona con acceso legítimo que abusa de sus privilegios.
- **OSINT (Open Source Intelligence)**: recopilación de información disponible públicamente para investigar individuos o organizaciones.
- **Hardcoding de credenciales**: práctica insegura de incluir credenciales directamente en el código fuente.
- **Cryptojacking**: uso no autorizado de recursos de cómputo ajenos para minar criptomonedas.
- **XMRig**: software de minería de Monero ampliamente usado en ataques de cryptojacking.
- **Sherlock**: herramienta OSINT que busca usernames en cientos de plataformas simultáneamente.
- **Búsqueda inversa de imágenes**: técnica para identificar el origen o contexto de una imagen usando motores de búsqueda visual.
