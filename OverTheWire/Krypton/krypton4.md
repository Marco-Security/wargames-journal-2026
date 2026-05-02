# Krypton Level 4 → 5

## Goal

Resolver el nivel `krypton4` de OverTheWire utilizando análisis de frecuencia para romper un cifrado **Vigenère** con una pista importante: la longitud de la clave ya es conocida.

El objetivo final es obtener la contraseña de `krypton5`.

---

## Acceso al nivel

```bash
ssh krypton4@krypton.labs.overthewire.org -p 2231
cd /krypton/krypton4
ls -la
```

---

## README

```bash
cat README
```

El README explica que este nivel utiliza un cifrado **Vigenère Cipher**, a diferencia de los niveles anteriores donde se usaban sustituciones simples o monoalfabéticas.

También entrega una pista crítica:

> The key length for this exercise is 6.

Esto significa que la clave tiene exactamente **6 letras**.

---

## Idea principal

Un cifrado Vigenère con clave de longitud 6 puede verse como:

```text
6 cifrados César intercalados
```

Es decir:

* posición 1 → usa letra 1 de la clave
* posición 2 → usa letra 2 de la clave
* posición 3 → usa letra 3 de la clave
* ...
* posición 7 → vuelve a usar letra 1

Por eso, si separamos el texto cifrado en 6 grupos, cada grupo puede resolverse como un cifrado César independiente.

---

## Archivos encontrados

```bash
cat found1
cat found2
cat krypton5
```

El archivo importante para la contraseña final es:

```text
HCIKV RJOX
```

Los archivos `found1` y `found2` contienen grandes bloques de texto cifrado que sirven para hacer análisis de frecuencia.

---

## Enfoque

### Paso 1 — Limpiar el texto

Eliminar:

* espacios
  n- saltos de línea
* caracteres no alfabéticos

para quedarnos únicamente con letras `A-Z`.

Ejemplo:

```text
YYICS JIZIB AGYYX
```

se convierte en:

```text
YYICSJIZIBAGYYX
```

---

### Paso 2 — Separar en 6 grupos

Como la clave mide 6:

```python
grupo[i % 6]
```

Esto permite agrupar letras que fueron cifradas con la misma letra de la clave.

Ejemplo:

```text
ABCDEFGH
```

con clave de longitud 3:

```text
Grupo 1 → A D G
Grupo 2 → B E H
Grupo 3 → C F
```

Cada grupo se ataca como un César.

---

### Paso 3 — Análisis de frecuencia

En inglés, las letras más frecuentes suelen ser:

```text
E T A O I N S H R
```

Si en un grupo la letra más repetida es `J`, podemos sospechar:

```text
J → E
```

Eso nos permite estimar el desplazamiento César.

Repitiendo esto para los 6 grupos obtenemos la clave completa.

---

## Script utilizado

```python
from collections import Counter


def limpiar_texto(texto):
    return ''.join(c for c in texto.upper() if c.isalpha())



def dividir_por_longitud_clave(texto, key_len):
    grupos = ['' for _ in range(key_len)]

    for i, char in enumerate(texto):
        grupos[i % key_len] += char

    return grupos


with open("found1", "r") as f:
    ciphertext = limpiar_texto(f.read())

key_len = 6
grupos = dividir_por_longitud_clave(ciphertext, key_len)

for i, grupo in enumerate(grupos):
    print(f"\nGrupo {i+1}")
    print(Counter(grupo).most_common(10))
```

---

## Clave encontrada

Después del análisis de frecuencia:

```text
FREKEY
```

Es un juego de palabras derivado de:

```text
FREQ + KEY
```

(referencia a frequency analysis)

---

## Descifrado final

Archivo:

```text
HCIKV RJOX
```

Usando la clave:

```text
FREKEY
```

obtenemos:

```text
CLEARTEXT
```

---

## Password para el siguiente nivel

```text
CLEARTEXT
```

---

## Login

```bash
ssh krypton5@kargames.labs.overthewire.org -p 2231
```

Password:

```text
CLEARTEXT
```

---

## Conclusión

La lección importante de este nivel no es memorizar la contraseña, sino entender que:

```text
Vigenère = varios César intercalados
```

Y que con:

```text
análisis de frecuencia
```

es posible reconstruir la clave completa cuando la longitud de la clave es conocida.

Este nivel introduce una técnica clásica de criptoanálisis y muestra por qué Vigenère, aunque más fuerte que César, sigue siendo vulnerable bajo ciertas condiciones.

