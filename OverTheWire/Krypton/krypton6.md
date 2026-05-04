# Krypton 6 → 7

## Información del nivel

| Campo | Valor |
|---|---|
| Wargame | Krypton (OverTheWire) |
| Nivel | krypton6 |
| Contraseña obtenida | LFSRISNOTRANDOM |

---

## Reconocimiento

```bash
krypton6@krypton:/krypton/krypton6$ ls -la
-rwsr-x--- 1 krypton7 krypton6 16528 encrypt6
-rw-r----- 1 krypton6 krypton6   164 HINT1
-rw-r----- 1 krypton6 krypton6    11 HINT2
-rw-r----- 1 krypton7 krypton7    11 keyfile.dat
-rw-r----- 1 krypton6 krypton6    15 krypton7
```

```
HINT1: The 'random' generator has a limited number of bits, and is periodic. There is a pattern!
HINT2: 8 bit LFSR
krypton7 (cifrado): PNUKLYLWRQKGKBE
```

---

## Conceptos clave

### ¿Qué es un Stream Cipher?

Un cifrado de flujo genera un **keystream** de bytes pseudo-aleatorios y los combina con el plaintext byte por byte:

```
ciphertext = plaintext + keystream  (mod 26 para letras)
```

Para descifrar:
```
plaintext = ciphertext - keystream  (mod 26)
```

### ¿Qué es un LFSR?

**Linear Feedback Shift Register** — generador de números pseudo-aleatorios basado en operaciones de bits.

Imagina un registro de 8 bits:
```
[1][0][1][1][0][0][1][0]
```

En cada tick:
1. Toma ciertos bits (los "taps") y los XORea
2. Empuja todos los bits a la derecha
3. El resultado del XOR entra por la izquierda
4. El bit que salió por la derecha es el output

### ¿Por qué es inseguro?

Un LFSR de 8 bits tiene 2^8 = 256 estados posibles. El estado todo-ceros es inútil, entonces el periodo máximo es **255**.

Después de 255 ticks, el LFSR vuelve al estado inicial y la secuencia **se repite**. Esto lo hace predecible y criptográficamente débil.

---

## Vulnerabilidad: Known Plaintext Attack

El binario `encrypt6` cifra cualquier archivo usando el keystream del LFSR. Si ciframos un texto plano **conocido** (como 30 As), podemos deducir el keystream:

```
ciphertext = plaintext + keystream
keystream  = ciphertext - plaintext
```

La `A` es ideal como plaintext porque vale 0:
```
keystream = ciphertext - 0 = ciphertext
```

El ciphertext de cifrar puras As **es directamente el keystream**.

---

## Explotación

### Paso 1: Generar el plaintext conocido

```bash
cd /tmp
python3 -c "print('A'*30)" > plain_marco.txt
```

### Paso 2: Cifrar con encrypt6

```bash
/krypton/krypton6/encrypt6 plain_marco.txt cipher_marco.txt
cat cipher_marco.txt
```

Output:
```
EICTDGYIYZKTHNSIRFXYCPFUEOCKRN
```

Este es el **keystream** — los primeros 30 valores del generador LFSR.

### Paso 3: Descifrar krypton7

```python
keystream  = "EICTDGYIYZKTHNSIRFXYCPFUEOCKRN"
ciphertext = "PNUKLYLWRQKGKBE"

plaintext = ''
for i, c in enumerate(ciphertext):
    k = ord(keystream[i]) - ord('A')
    p = (ord(c) - ord('A') - k) % 26
    plaintext += chr(p + ord('A'))

print(plaintext)
```

### Ejemplo paso a paso (primeras 3 letras):

**i=0, c='P', keystream[0]='E':**
```
k = ord('E') - ord('A') = 4
p = (ord('P') - 65 - 4) % 26 = 11
chr(11 + 65) = 'L'
```

**i=1, c='N', keystream[1]='I':**
```
k = ord('I') - ord('A') = 8
p = (ord('N') - 65 - 8) % 26 = 5
chr(5 + 65) = 'F'
```

**i=2, c='U', keystream[2]='C':**
```
k = ord('C') - ord('A') = 2
p = (ord('U') - 65 - 2) % 26 = 18
chr(18 + 65) = 'S'
```

→ `LFS...` → **LFSRISNOTRANDOM**

---

## Resultado

```
Contraseña: LFSRISNOTRANDOM
```

Un mensaje apropiado: **"LFSR is not random"**.

---

## Comparación con krypton5 (Vigenère)

| Aspecto | Krypton5 (Vigenère) | Krypton6 (LFSR) |
|---|---|---|
| Tipo de cifrado | Polialfabético con clave fija | Stream cipher con keystream generado |
| Periodo | Longitud de la clave (9) | 255 (LFSR de 8 bits) |
| Ataque | IC + χ² | Known plaintext attack |
| Clave | String `KEYLENGTH` | Keystream generado algorítmicamente |
| Vulnerabilidad | Clave repetida cíclicamente | Periodo finito y predecible |

En ambos casos la vulnerabilidad es el **ciclo predecible** — pero el método de descubrimiento es diferente:
- Vigenère: análisis estadístico del ciphertext
- LFSR: cifrar texto plano conocido para revelar el keystream

---

## Apéndice: Fórmulas estadísticas usadas en krypton5

### Index of Coincidence (IC)

```
IC = Σ (f_i × (f_i - 1)) / (N × (N - 1))
```

- `f_i` = frecuencia de cada letra en el texto
- `N` = total de letras

**Ejemplo** con `HELLOWORLD` (N=10):
```
L aparece 3 veces: 3 × 2 = 6
O aparece 2 veces: 2 × 1 = 2
Resto aparece 1 vez: 1 × 0 = 0

IC = (6 + 2) / (10 × 9) = 8/90 = 0.089
```

Valores de referencia: Inglés ~0.067, Aleatorio ~0.038.

### Chi-Squared (χ²)

```
χ² = Σ (observado - esperado)² / esperado
```

**Ejemplo** con shift=10 correcto (100 letras):
```
E: (13 - 12.7)² / 12.7 = 0.007
T: (9 - 9.0)²  / 9.0  = 0.000
...
χ² total = 0.8  ← bajo = correcto
```

**Ejemplo** con shift=5 incorrecto:
```
Z: (15 - 0.07)² / 0.07 = 3185
...
χ² total = 3185  ← alto = incorrecto
```

El shift con **χ² mínimo** es la letra correcta de la clave.
