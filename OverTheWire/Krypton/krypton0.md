# Krypton Nivel 0 → Nivel 1

* **Enlace del Reto:** [Krypton Level 0](https://overthewire.org/wargames/krypton/krypton0.html)
* **Categoría:** Codificación / Base64

---

## Descripción del Desafío
El primer nivel de Krypton es un calentamiento centrado en la codificación de datos. Se nos proporciona una cadena que codifica la contraseña para el siguiente nivel usando **Base64**:

`S1JZUFRPTklTR1JFQVQ=`

El objetivo es decodificar esta cadena para obtener las credenciales del usuario `krypton1`.

---

## Teoría de Explotación
**Base64** no es un algoritmo de cifrado; es un **esquema de codificación de binario a texto**. Representa datos binarios en un formato de cadena ASCII traduciéndolos a una representación de base 64.

Características clave para su identificación:
*   Utiliza los caracteres `A-Z`, `a-z`, `0-9`, `+` y `/`.
*   A menudo termina con uno o dos símbolos `=` (relleno o *padding*) para asegurar que la longitud del mensaje sea un múltiplo de 4 bytes.

En contextos de seguridad, Base64 se utiliza a menudo para transmitir datos sin pérdida, pero **no proporciona ninguna confidencialidad**.

---

## Paso a Paso
1.  **Identificar la cadena:** La entrada `S1JZUFRPTklTR1JFQVQ=` presenta el carácter de relleno típico de Base64 `=`.
2.  **Decodificar vía Terminal:** Se utiliza la utilidad `base64` en Linux para procesar la cadena.
3.  **Ejecución del comando:**
    ```bash
    echo "S1JZUFRPTklTR1JFQVQ=" | base64 -d
    ```
4.  **Resultado:** El comando devuelve la contraseña en texto plano.

---

## Solución
*   **Contraseña Decodificada:** `KRYPTONISGREAT`
*   **Comando de Acceso:** 
    ```bash
    ssh krypton1@krypton.labs.overthewire.org -p 2231
    ```
