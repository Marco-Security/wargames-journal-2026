# Krypton 5 → 6

## Información del nivel

| Campo | Valor |
|---|---|
| Wargame | Krypton (OverTheWire) |
| Nivel | krypton5 |
| Contraseña obtenida | random |

---

## Reconocimiento

```bash
krypton5@krypton:/krypton/krypton5$ ls
found1  found2  found3  krypton6  README

krypton5@krypton:/krypton/krypton5$ cat README
Frequency analysis can break a known key length as well.
Lets try one last polyalphabetic cipher, but this time the key length is unknown.
```

Tres archivos cifrados (`found1`, `found2`, `found3`) con la **misma clave** Vigenère de longitud **desconocida**, y el archivo `krypton6` que contiene la contraseña cifrada.

---

## Vigenère y por qué fue indescifrable 300 años

**Vigenère** es un cifrado polialfabético que usa una clave de varias letras. Cada letra del texto se cifra desplazándola según la letra correspondiente de la clave (que se repite cíclicamente).

```
Texto:  THEQUICKBROWNFOX
Clave:  KEYKEYKEYKEYKEYK
Cifr:   DLCATGSOZBSUXJSH
```

**Por qué fue considerado indescifrable hasta 1863:**

Los cifrados anteriores (César, monoalfabéticos) mantenían el patrón estadístico del idioma — la `E` seguía siendo la letra más común en el ciphertext, solo cambiada por otra. Eso permitía análisis de frecuencias trivial.

Vigenère **rompe ese patrón**. La `E` se cifra con diferentes letras según su posición → no hay una letra dominante en el ciphertext → el análisis de frecuencias clásico no funciona.

**Kasiski (1863)** descubrió la idea clave: si descubres la **longitud de la clave**, puedes dividir el texto en columnas — cada columna fue cifrada con la misma letra de la clave, convirtiéndola en un cifrado César. Y un César sí se rompe con análisis de frecuencias.

---

## Estrategia del ataque

Dos fases:

1. **Encontrar la longitud de la clave** con Index of Coincidence
2. **Recuperar cada letra de la clave** con análisis χ² por columna

---

## Fase 1: Index of Coincidence (IC)

### ¿Qué es el IC?

Es la probabilidad de que dos letras tomadas al azar del texto sean iguales.

```
IC = Σ (f_i × (f_i - 1)) / (N × (N - 1))
```

Valores de referencia:

| Tipo de texto | IC |
|---|---|
| Inglés natural | ~0.067 |
| Cifrado César/monoalfabético | ~0.067 |
| Vigenère con clave larga | ~0.038 |
| Texto aleatorio | ~0.038 |

### Por qué funciona

Si dividimos el texto en N columnas (donde N es la longitud correcta de la clave), cada columna fue cifrada con la misma letra de la clave — es esencialmente un César. El IC de cada columna vuelve a ser ~0.067.

Si N no es la longitud correcta, las columnas mezclan diferentes shifts → IC bajo.

### Script

```python
text = open('/krypton/krypton5/found1').read().replace(' ', '').replace('\n', '').upper()

def ic(s):
    n = len(s)
    if n < 2: return 0
    freq = {c: s.count(c) for c in set(s)}
    return sum(f*(f-1) for f in freq.values()) / (n*(n-1))

print("Longitud | IC promedio")
for keylen in range(1, 15):
    cosets = [text[i::keylen] for i in range(keylen)]
    avg_ic = sum(ic(c) for c in cosets) / keylen
    print(f"   {keylen:2d}    | {avg_ic:.4f}")
```

### Resultado

```
Longitud | IC promedio
    1    | 0.0418
    2    | 0.0419
    3    | 0.0515
    4    | 0.0419
    5    | 0.0418
    6    | 0.0518
    7    | 0.0424
    8    | 0.0423
    9    | 0.0700  ← ¡Este!
   10    | 0.0418
```

Longitud de la clave = **9**.

---

## Fase 2: Análisis Chi-Squared (χ²)

### ¿Qué es χ²?

Mide qué tan diferente es una distribución observada de una distribución esperada:

```
χ² = Σ (observado - esperado)² / esperado
```

Si descifras correctamente, las frecuencias coinciden con las del inglés → χ² **bajo**.

Si descifras mal, las frecuencias son raras → χ² **alto**.

### Cómo encontrar cada letra de la clave

Para cada una de las 9 columnas:
1. Probar los 26 desplazamientos posibles (A-Z)
2. Para cada shift, descifrar la columna y calcular χ²
3. El shift con el χ² mínimo es la letra correcta de la clave

### Script

```python
text = open('/krypton/krypton5/found1').read().replace(' ', '').replace('\n', '').upper()
text = ''.join(c for c in text if c.isalpha())

ENG_FREQ = {
    'A': 0.0817, 'B': 0.0150, 'C': 0.0278, 'D': 0.0425, 'E': 0.1270,
    'F': 0.0223, 'G': 0.0202, 'H': 0.0609, 'I': 0.0697, 'J': 0.0015,
    'K': 0.0077, 'L': 0.0403, 'M': 0.0241, 'N': 0.0675, 'O': 0.0751,
    'P': 0.0193, 'Q': 0.0010, 'R': 0.0599, 'S': 0.0633, 'T': 0.0906,
    'U': 0.0276, 'V': 0.0098, 'W': 0.0236, 'X': 0.0015, 'Y': 0.0197,
    'Z': 0.0007
}

KEYLEN = 9

def chi_squared(group, shift):
    decrypted = ''.join(chr((ord(c) - ord('A') - shift) % 26 + ord('A')) for c in group)
    n = len(decrypted)
    chi = 0
    for letter, freq in ENG_FREQ.items():
        observed = decrypted.count(letter)
        expected = freq * n
        if expected > 0:
            chi += (observed - expected) ** 2 / expected
    return chi

key = ''
for i in range(KEYLEN):
    group = text[i::KEYLEN]
    best_shift = min(range(26), key=lambda s: chi_squared(group, s))
    key += chr(best_shift + ord('A'))

print(f"Clave: {key}")
```

### Resultado

```
Clave: KEYLENGTH
```

Un nombre apropiado para el reto.

---

## Fase 3: Descifrar krypton6

```python
KEY = "KEYLENGTH"
ciphertext = open('/krypton/krypton5/krypton6').read().replace(' ', '').replace('\n', '').upper()
ciphertext = ''.join(c for c in ciphertext if c.isalpha())

plaintext = ''
for i, c in enumerate(ciphertext):
    shift = ord(KEY[i % len(KEY)]) - ord('A')
    plaintext += chr((ord(c) - ord('A') - shift) % 26 + ord('A'))

print(plaintext)
```

### Resultado

```
RANDOM
```

Contraseña de krypton6: `random`

---

## Conceptos clave

- **Vigenère**: cifrado polialfabético que usa una clave repetida. Cada letra se cifra con un César distinto según la letra correspondiente de la clave.
- **Método de Kasiski (1863)**: técnica para encontrar la longitud de la clave buscando repeticiones de secuencias en el ciphertext.
- **Index of Coincidence (IC)**: medida estadística que indica qué tan natural es la distribución de letras. ~0.067 para inglés, ~0.038 para texto aleatorio.
- **Chi-Squared (χ²)**: prueba estadística que mide la diferencia entre distribución observada y esperada. Útil para identificar el shift correcto en un César.
- **Coset**: en este contexto, cada uno de los grupos resultantes de tomar cada N-ésima letra del texto. Si N es la longitud de la clave, cada coset es esencialmente un cifrado César.
- **Bigotry of frequency analysis**: el motivo real por el que Vigenère se consideró indescifrable era que rompía el patrón estadístico que hacía vulnerables a los cifrados anteriores. Hasta Kasiski no se sabía sistemáticamente cómo atacar la longitud de la clave.
