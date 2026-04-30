# Krypton Nivel 2 → Nivel 3

* **Enlace del Reto:** [Krypton Level 2](https://overthewire.org/wargames/krypton/krypton2.html)
* **Categoría:** Criptografía Clásica / Cifrado César (Ataque de Oráculo)

---

## Descripción del Desafío
La contraseña para el nivel 3 se encuentra en el archivo `/krypton/krypton2/krypton3`. El sistema utiliza un **Cifrado César**, que es una sustitución monoalfabética por desplazamiento. Aunque no conocemos la clave, el reto proporciona un binario llamado `encrypt` que utiliza la clave real del sistema para cifrar cualquier archivo que le proporcionemos.

---

## Teoría de Explotación
Este reto se resuelve mediante un **Known Plaintext Attack** (Ataque de Texto Plano Conocido) utilizando el binario como un "Oráculo de Cifrado". 

En un cifrado César, cada letra se desplaza un número fijo de posiciones ($n$). Si ciframos la letra **'A'** (que representa la posición 0 en el alfabeto), el carácter resultante en el criptograma nos indica directamente el valor del desplazamiento.

En este caso:
*   **Texto de entrada:** `AAA`
*   **Resultado del Oráculo:** `MMM`
*   **Deducción:** La letra 'A' se convirtió en 'M'. Contando desde A (0) hasta M (12), el desplazamiento es de **12** posiciones hacia adelante.

---

## Paso a Paso
1.  **Entorno Temporal:** Creamos un directorio en `/tmp` y enlazamos el archivo de clave (`keyfile.dat`) porque el binario está programado para buscarlo en el directorio de ejecución actual.
    ```bash
    MYDIR=$(mktemp -d)
    cd $MYDIR
    ln -s /krypton/krypton2/keyfile.dat
    chmod 777 .
    ```
2.  **Identificación de la Clave:** Ciframos una cadena conocida para observar la transformación del algoritmo.
    ```bash
    echo "AAA" > prueba.txt
    /krypton/krypton2/encrypt prueba.txt
    cat ciphertext # Resultado obtenido: MMM
    ```
3.  **Descifrado Final:** Dado que el alfabeto cifrado comienza en 'M' en lugar de 'A', aplicamos la traslación inversa al archivo que contiene la contraseña:
    ```bash
    cat /krypton/krypton2/krypton3 | tr 'M-ZA-L' 'A-Z'
    ```

---

## Solución
*   **Desplazamiento identificado:** 12 (A -> M)
*   **Contraseña Decodificada:** `CAESARISEASY`
*   **Comando de Acceso:** 
    ```bash
    ssh krypton3@krypton.labs.overthewire.org -p 2231
    ```
