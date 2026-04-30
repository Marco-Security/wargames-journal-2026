# OverTheWire: Krypton Level 3 → 4

### Descripción del Reto
El nivel 3 de Krypton presenta un cifrado de **sustitución monoalfabética**. A diferencia de un cifrado César, aquí no hay un desplazamiento fijo; cada letra del alfabeto ha sido sustituida por otra de forma aleatoria. La clave para romperlo es el **Análisis de Frecuencias**.

---

### Fase 1: Análisis Estadístico (Python)
Para identificar las letras más comunes, analizamos los archivos de pista (`found1`, `found2`, `found3`) con el siguiente script para obtener los porcentajes reales:

```python
import collections
import string

# Script de conteo de frecuencias
archivos = ['found1', 'found2', 'found3']
conteo = collections.Counter()
total = 0

for f_name in archivos:
    with open(f'/krypton/krypton3/{f_name}', 'r') as f:
        data = f.read().upper()
        letras = [c for c in data if c in string.ascii_uppercase]
        conteo.update(letras)
        total += len(letras)

# Top 5 resultados obtenidos:
# S: 12.93% (Candidato: E)
# Q: 9.64%  (Candidato: T)
# J: 8.54%  (Candidato: A)
# U: 7.29%  (Candidato: O)
# B: 6.98%  (Candidato: I)
