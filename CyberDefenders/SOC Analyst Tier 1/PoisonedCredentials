# PoisonedCredentials — CyberDefenders

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

La organización detecta un surge de actividad sospechosa en la red. El análisis inicial indica el uso de **poisoning attacks** que apuntan a los protocolos **LLMNR** y **NBT-NS** para interceptar tráfico legítimo y robar credenciales de usuario.

---

## Conceptos previos

### ¿Qué es LLMNR?

**Link-Local Multicast Name Resolution** — protocolo que permite a las máquinas de una red local resolver nombres a IPs sin depender de un servidor DNS central.

**Vulnerabilidad:** cuando una máquina no puede resolver un nombre vía DNS, hace un broadcast a toda la red preguntando "¿quién tiene este nombre?". Cualquier máquina en la red puede responder — incluyendo un atacante.

### ¿Qué es NBT-NS?

**NetBIOS Name Service** — funciona igual que LLMNR pero para redes Windows antiguas. Traduce nombres NetBIOS a IPs.

**Vulnerabilidad:** tampoco tiene mecanismos de autenticación. Un atacante puede responder a cualquier query con una IP maliciosa.

### ¿Cómo funciona el ataque de Poisoning?

```
1. Víctima escribe mal un nombre de recurso (ej: \\FILESHAARE)
2. DNS no encuentra el nombre → la máquina hace un broadcast LLMNR/NBT-NS
3. Atacante intercepta el broadcast y responde: "Soy yo, FILESHAARE"
4. Víctima cree la respuesta y envía sus credenciales NTLM al atacante
5. Atacante captura el hash NTLM y puede crackearlo offline
```

---

## Tácticas MITRE ATT&CK

| Táctica | Técnica | Descripción |
|---|---|---|
| Credential Access | T1557.001 | LLMNR/NBT-NS Poisoning |
| Credential Access | T1110 | Brute Force (crackeo offline del hash) |
| Lateral Movement | T1021.002 | SMB para acceso a recursos |

---

## Análisis pregunta por pregunta

### Q1 — Query mal escrita por la víctima

**Pregunta:** ¿Cuál fue la query mal escrita por la máquina 192.168.232.162?

**Respuesta:** `FILESHAARE<20>`

La máquina intentó resolver el nombre `FILESHAARE` (con doble A) en lugar de `FILESHARE`. Al no encontrarlo en DNS, hizo un broadcast NBT-NS a `192.168.235.255` (dirección de broadcast de la red).

**Filtro Wireshark:**
```
nbns
```

El `<20>` indica que es un recurso de tipo **File Server** en NetBIOS.

---

### Q2 — IP de la máquina rogue

**Pregunta:** ¿Cuál es la IP de la máquina rogue (atacante)?

**Respuesta:** `192.168.232.215`

En el paquete 51, esta IP respondió al broadcast de `FILESHAARE<20>` — afirmando ser ese recurso. Como no existe ese servidor legítimo, esta respuesta solo puede venir del atacante.

**Filtro Wireshark:**
```
nbns && ip.src == 192.168.232.215
```

---

### Q3 — Segunda máquina afectada

**Pregunta:** ¿Cuál es la IP de la segunda máquina que recibió respuestas envenenadas?

**Respuesta:** `192.168.232.176`

Filtrando todo el tráfico NBNS relacionado con la IP del atacante, se identifican dos víctimas:
1. `192.168.232.162` (primera víctima)
2. `192.168.232.176` (segunda víctima)

**Filtro Wireshark:**
```
nbns.addr == 192.168.232.215
```

---

### Q4 — Usuario comprometido

**Pregunta:** ¿Cuál es el nombre del usuario comprometido?

**Respuesta:** `janesmith`

Al analizar el tráfico SMB dirigido a la máquina rogue, se observa un **SMB2 Session Setup** con autenticación NTLM. Dentro del payload NTLMSSP se puede leer:
- **Username:** `janesmith`
- **Domain:** `cybercactus.local`
- **Host:** `WORKSTATION`

**Filtro Wireshark:**
```
ip.dst == 192.168.232.215
```

**¿Por qué SMB?**
SMB (Server Message Block) es el protocolo de compartición de archivos de Windows. Cuando la víctima intenta conectarse al recurso falso `FILESHAARE`, envía sus credenciales NTLM automáticamente — interceptadas por el atacante.

---

### Q5 — Hostname de la máquina accedida

**Pregunta:** ¿Cuál es el hostname de la máquina accedida vía SMB?

**Respuesta:** `AccountingPC`

En el paquete SMB2 con el NTLMSSP Challenge, el campo **Target Info → DNS Computer Name** revela:
```
AccountingPC.cybercactus.local
```

**Filtro Wireshark:**
```
ip.dst == 192.168.232.215 && smb2
```

---

## Flujo completo del ataque

```
1. Typo de la víctima
   └── 192.168.232.162 intenta acceder a \\FILESHAARE (mal escrito)
   └── DNS no encuentra el nombre
   └── Broadcast NBT-NS a toda la red

2. Poisoning
   └── 192.168.232.215 (atacante) responde al broadcast
   └── "Soy FILESHAARE, conéctate a mí"
   └── Segunda víctima 192.168.232.176 también afectada

3. Captura de credenciales
   └── Víctima (192.168.232.162) intenta autenticarse vía SMB
   └── Envía credenciales NTLM de janesmith al atacante
   └── Atacante captura el hash NTLM

4. Reconocimiento
   └── Atacante identifica máquina objetivo: AccountingPC.cybercactus.local
   └── Puede crackear el hash offline con herramientas como Hashcat
```

---

## Indicadores de Compromiso (IoCs)

| Tipo | Valor |
|---|---|
| IP víctima 1 | 192.168.232.162 |
| IP víctima 2 | 192.168.232.176 |
| IP atacante (rogue) | 192.168.232.215 |
| Query envenenada | FILESHAARE<20> |
| Usuario comprometido | janesmith |
| Dominio | cybercactus.local |
| Máquina objetivo | AccountingPC |

---

## Lecciones aprendidas para el Blue Team

1. **Deshabilitar LLMNR y NBT-NS** cuando no sean necesarios — son protocolos legacy con vulnerabilidades conocidas
2. **Implementar DNS con DNSSEC** para validar respuestas
3. **Monitorear broadcasts NBNS/LLMNR** en la red — múltiples respuestas al mismo query es señal de poisoning
4. **Usar SMB Signing** — previene que el atacante intercepte y manipule comunicaciones SMB
5. **Alertar sobre autenticaciones NTLM hacia IPs no conocidas** — un usuario autenticándose hacia una IP desconocida es señal de compromiso

---

## Herramientas utilizadas

- **Wireshark** — análisis de PCAP con filtros específicos por protocolo (NBNS, SMB2, NTLMSSP)
- **Filtros de display** — `nbns`, `nbns.addr`, `ip.dst`, `smb2`

---

## Conceptos clave

- **LLMNR**: protocolo de resolución de nombres local sin autenticación. Vector clásico de ataques de poisoning.
- **NBT-NS**: protocolo NetBIOS para resolución de nombres en redes Windows. Igualmente vulnerable.
- **Poisoning attack**: el atacante responde a broadcasts de resolución de nombres con IPs maliciosas para interceptar tráfico.
- **NTLM**: protocolo de autenticación de Windows basado en hash challenge-response. Vulnerable a ataques de relay y crackeo offline.
- **SMB (Server Message Block)**: protocolo de compartición de archivos/recursos en Windows. Transporta las credenciales NTLM.
- **Broadcast**: mensaje enviado a toda la red. Cualquier máquina puede responder — incluyendo atacantes.
- **Responder**: herramienta popular usada por atacantes para realizar LLMNR/NBT-NS poisoning automáticamente.
