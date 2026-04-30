# Krypton Nivel 1 → Nivel 2

* **Enlace del Reto:** [Krypton Level 1](https://overthewire.org/wargames/krypton/krypton1.html)
* **Categoría:** Criptografía Clásica / Cifrado de Sustitución (ROT13)

---

## Descripción del Desafío
En este nivel, se nos indica que la contraseña para el nivel 2 se encuentra en el archivo `krypton2` dentro del directorio `/krypton/krypton1/`. El archivo `README` especifica que el texto está cifrado utilizando **ROT13**, un cifrado de sustitución simple.

A diferencia de los formatos estándar de criptografía que agrupan letras en bloques de 5, este archivo mantiene los límites de las palabras originales.

---

## Teoría de Explotación
El **ROT13** ("rotate by 13 places") es una variante del cifrado César. Su funcionamiento consiste en sustituir cada letra del alfabeto por la que se encuentra 13 posiciones después.

Características importantes:
*   Es **simétrico**: Debido a que el alfabeto inglés tiene 26 letras, aplicar la rotación de 13 posiciones dos veces ($13 + 13 = 26$) regresa al carácter original.
*   No requiere una clave compleja, solo el conocimiento del algoritmo.

---

## Paso a Paso
1.  **Explorar el directorio:** Accedemos a la carpeta del reto:
    ```bash
    cd /krypton/krypton1
    ```
2.  **Leer el texto cifrado:**
    ```bash
    cat krypton2
    # Salida: YRIRY GJB CNFFJBEQ EBGGRA
    ```
3.  **Descifrar usando `tr`:** Utilizamos la herramienta de traducción de caracteres de Linux para rotar el alfabeto:
    ```bash
    cat krypton2 | tr 'A-Za-z' 'N-ZA-Mn-za-m'
    ```
4.  **Análisis de salida:** El comando devuelve: `LEVEL TWO PASSWORD ROTTEN`.

---

## Solución
*   **Contraseña Decodificada:** `ROTTEN`
*   **Comando de Acceso:** 
    
```bash
    ssh krypton2@krypton.labs.overthewire.org -p 2231
    ```
