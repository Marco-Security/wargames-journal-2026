# Natas 22 → 23

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas22.natas.labs.overthewire.org |
| Usuario | natas22 |
| Contraseña | d8rwGBl0Xslg3b76uh3fEbSlnOUBlozz |
| Contraseña obtenida | dIUQcI3uSus1JEOSSWRAEXBG8KbR8tRs |

---

## Reconocimiento

La página principal está vacía. El código fuente revela que existe un parámetro GET llamado `revelio` que muestra la contraseña — pero con una trampa.

---

## Análisis del código fuente

```php
session_start();

if(array_key_exists("revelio", $_GET)) {
    if(!($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1)) {
        header("Location: /");
    }
}
```

Si la URL contiene `?revelio` y el usuario **no** es admin, el servidor envía un header de redirección:

```
header("Location: /")
```

Esto le dice al navegador que vaya a la página principal. El navegador obedece y nunca muestra la contraseña.

```php
if(array_key_exists("revelio", $_GET)) {
    print "Password: <censored>";
}
```

Sin embargo, el servidor **sí envía la contraseña** en la respuesta HTTP — solo que también envía la redirección. El navegador ve la redirección primero y descarta el resto.

---

## Vulnerabilidad: redirección sin exit()

El problema es que el código no detiene la ejecución después de `header("Location: /")`. En PHP, `header()` solo agrega un encabezado HTTP — no detiene el script. Sin un `exit()` o `die()` después, el código sigue ejecutándose y la contraseña se incluye en la respuesta.

**Código vulnerable:**
```php
header("Location: /");
// falta exit(); aquí
```

**Código correcto:**
```php
header("Location: /");
exit();
```

---

## ¿Qué es una redirección HTTP?

Es un mecanismo donde el servidor le indica al cliente que vaya a otra URL. Se implementa con el header `Location`. Los navegadores siguen estas redirecciones automáticamente.

Los códigos de redirección más comunes son:
- `301` → redirección permanente
- `302` → redirección temporal (la que usa PHP por defecto con `header()`)

---

## Herramienta: curl sin seguir redirecciones

Por defecto, curl **no sigue redirecciones** — recibe la respuesta completa del servidor incluyendo el contenido después de `header()`.

| Flag | Comportamiento |
|---|---|
| Sin `-L` | curl recibe la respuesta completa, ignora la redirección |
| Con `-L` | curl sigue la redirección como un navegador |

En este nivel, **no usar `-L`** es lo que permite ver la contraseña.

---

## Explotación

```bash
curl -s "http://natas22.natas.labs.overthewire.org/?revelio" \
  -u natas22:d8rwGBl0Xslg3b76uh3fEbSlnOUBlozz | grep -i password
```

El servidor envía la contraseña en la respuesta junto con el header de redirección. Curl la recibe antes de que ocurra cualquier redirección.

---

## Conceptos clave

- **`header("Location: /")`**: redirección HTTP en PHP. Solo agrega un encabezado — no detiene la ejecución del script.
- **`exit()`**: detiene la ejecución del script en PHP. Su ausencia después de una redirección es una vulnerabilidad común.
- **curl `-L`**: flag para seguir redirecciones. Sin él, curl recibe la respuesta completa.
- **Respuesta HTTP**: contiene headers y body. La redirección está en los headers, la contraseña en el body — curl los recibe juntos.
