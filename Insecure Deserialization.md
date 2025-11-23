**Serialization** is the process of converting complex data structures, such as objects and their fields, into a "flatter" format that can be sent and received as a sequential stream of bytes.

**Deserialization** is the process of restoring this byte stream to a fully functional replica of the original object, in the exact state as when it was serialized. The website's logic can then interact with this deserialized object, just like it would with any other object.

## How to identify insecure deserialization
### PHP serialization format

Serializa en string

Imaginemosel siguiente objeto :

```
$user->name = "carlos"; 
$user->isLoggedIn = true;
```

cuando se serealiza se ve algo asi : 

```
O:4:"User":2:{s:4:"name":s:6:"carlos";s:10:"isLoggedIn":b:1;}
```

- `O:4:"User"` - An object with the 4-character class name `"User"`
- `2` - the object has 2 attributes
- `s:4:"name"` - The key of the first attribute is the 4-character string `"name"`
- `s:6:"carlos"` - The value of the first attribute is the 6-character string `"carlos"`
- `s:10:"isLoggedIn"` - The key of the second attribute is the 10-character string `"isLoggedIn"`
- `b:1` - The value of the second attribute is the boolean value `true`

si tenemos el codigo fuente hay que echarle un ojo a : `serialize()` y `unserialize()`
### Java serialization format

serializa en binario 

los objetos serializados en java siempre empiezan con los mismos bytes:

hexadecimal : `ac ed`
Base64  : `rO0`


si tenemos el codigo fuente hay que echarle un ojo a : `readObject()` 

## Manipulating serialized objects

### Modifying object attributes

Si vemos un objeto que la web usa un objeto serializado User si lo decodificamos nos dara algo así : 

`O:4:"User":2:{s:8:"username";s:6:"carlos";s:7:"isAdmin";b:0;}`

Y si cambiamos isAdmin de 0 a 1 entonces seremos admin : D

### Modifying data types

PHP es bastante vulnerable a este tipo de ataques puesto que su operador `==` dará true una condición como : `5 == "5"` siendo el último 5 un string

admeas php si le pones : `5 == "5 of something"` sera igual que `5 == 5` puesto que solo comprueba la primera parte 

En versiones de PHP 7.X la comparación `0 == "Example string"` sera true puesto que el string será evaluado como 0

En PHP 8.X lo de `0 == "Example string"` ya no funsiona (sad)
EJEMPLO:

```
$login = unserialize($_COOKIE) 
	if ($login['password'] == $password) { 
	// log in successfully 
	}
```

imaginemos que conseguimos hacer que la constraseña se evalue como 0 pues mientras la contraseña real empiece como string (sin números al principio) se evaluara como true puesto que hara 0 == 0

Be aware that when modifying data types in any serialized object format, it is important to remember to update any type labels and length indicators in the serialized data too. Otherwise, the serialized object will be corrupted and will not be deserialized.

## Using application functionality

Por ejemplo imaginemos que una web usar una funcionalidad "Delete user" , donde se borra la foto de perfil del objeto user acceciendo a este path  `$user->image_location` .  si el `$user` es creado de un objeto serializado, Un atacante podría pasar el path de otra cosa a  `image_location` . Por ende al borrar el usuario tambien borraría lo que estuviese en ese path

## Magic methods(no lab)

Los Magic methods son métodos especiales que no se llaman explícitamente: se ejecutan automáticamente cuando ocurre un evento concreto (instanciación, serialización, deserialización, conversión a string, etc.). Suele reconocerse por dobles guiones bajos en el nombre (ej. `__construct`, `__wakeup`).

- PHP: `__construct()` (constructor), `__wakeup()` (al deserializar), `__toString()` (al convertir a string).

- Python: `__init__()` (constructor).

- Java (deserialización): una clase `Serializable` puede declarar

```Java
private void readObject(ObjectInputStream in)
    throws IOException, ClassNotFoundException
{
    // implementación
}
```

## Injecting arbitrary objects


además de editar un objeto serializado, un atacante puede _inyectar objetos de cualquier clase serializable_ disponible en la aplicación para forzar la ejecución de código durante o después de la deserialización.

los métodos y comportamientos de un objeto dependen de su clase. Si la deserialización no valida la clase, se puede entregar un objeto de una clase distinta y éste será instanciado igual; sus métodos (incluyendo métodos mágicos ejecutados en deserialización) pueden ejecutarse antes de que la app detecte el tipo inesperado.

Como explotarlo

1. Revisan el código (si tienen acceso) para listar clases serializables.

2. Buscan clases con _métodos mágicos_ de deserialización (por ejemplo `__wakeup`, `readObject`, etc.) que realicen operaciones con datos entrantes.

3.  Crean un objeto serializado de esa clase con campos controlados por el atacante.

4.  Envían ese objeto para que, al deserializarse, se ejecute la lógica maliciosa.

GET /libs/CustomTemplate.php~ -> source code
## Gadget chains

Un **gadget** es un fragmento de código existente en la aplicación que, por sí solo, puede no ser peligroso, pero que un atacante puede usar para encadenar llamadas y transportar su entrada hasta un **sink** peligroso. El atacante no crea código nuevo: solo controla los datos que se pasan a esos gadgets, normalmente a través de un método mágico invocado al deserializar (un _kick-off gadget_). Muchas vulnerabilidades de deserialización insegura requieren dichas **cadenas de gadgets**; a veces son simples (1–2 pasos) y otras veces son secuencias complejas necesarias para lograr un ataque de alta gravedad.

### Working with pre-built gadget chains

Identificar manualmente cadenas de gadgets es laborioso y casi imposible sin acceso al código fuente. Por suerte existen herramientas con cadenas predescubiertas que puedes probar primero. Estas herramientas permiten detectar y explotar deserializaciones inseguras aún sin el código fuente, porque muchas librerías populares (p. ej. Apache Commons Collections en Java) contienen cadenas explotables reutilizables: si una cadena funciona en una web que usa esa librería, puede funcionar en otras que la usen.

#### ysoserial

herramienta para deserialización Java que incluye cadenas de gadgets predefinidas. Permite seleccionar una cadena conocida (para una librería que sospeches que usa la aplicación objetivo) y generar un objeto serializado que intente ejecutar una acción al deserializarse. Reduce mucho el trabajo frente a construir cadenas manualmente, aunque suele requerir prueba y error. _Nota:_ en Java 16+ es necesario ajustar opciones del JVM para poder ejecutar ciertas cargas generadas por ysoserial como por ejemplo : 

```
java \
  --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \
  --add-exports=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \
  --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \
  --add-opens=java.base/java.net=ALL-UNNAMED \
  --add-opens=java.base/java.util=ALL-UNNAMED \
  -jar ysoserial-all.jar CommonsCollections4 'rm /home/carlos/morale.txt' \
  2>/dev/null | base64

```


No todas las cadenas de ysoserial permiten ejecutar código arbitrario; muchas se usan para detección. 

URLDNS -> genera un objeto que provoca una consulta DNS hacia la URL que le pases. Es muy universal porque no depende de una librería vulnerable concreta ni de la versión de Java; si observas la consulta (por ejemplo con Burp Collaborator), confirma que se realizó deserialización en el servidor.

JRMPClient ->  hace que el servidor intente abrir una conexión TCP a una IP proporcionada (requiere una IP en crudo, no un nombre de host). Es útil cuando el tráfico saliente está restringido o el DNS está bloqueado. Una técnica práctica es generar dos payloads, uno con una IP local y otro con una IP externa bloqueada: si la respuesta al payload con la IP local es inmediata pero la del payload con la IP externa tarda o se queda colgada, esto indica que el servidor intentó conectar a la IP externa y, por tanto, que la deserialización se ejecutó aunque sea de forma “blind”.

#### PHP Generic Gadget Chains

Most languages that frequently suffer from insecure deserialization vulnerabilities have equivalent proof-of-concept tools. For example, for PHP-based sites you can use "PHP Generic Gadget Chains" (PHPGGC).

```
./phpggc Symfony/RCE4 exec 'rm /home/carlos/morale.txt' | base64
```

en este laboratorio para generar una cookie valida tendremos que usar este script :

```
<?php $object = "OBJECT-GENERATED-BY-PHPGGC"; 
$secretKey = "LEAKED-SECRET-KEY-FROM-PHPINFO.PHP"; 
$cookie = urlencode('{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretKey) . '"}'); echo $cookie;
```
### Working with documented gadget chains

Puede que no exista una herramienta dedicada para explotar cadenas de gadgets en el framework que usa la aplicación objetivo. En ese caso, busca en Internet exploits documentados que puedas adaptar manualmente; modificarlos suele requerir conocimientos básicos del lenguaje y del framework, y a veces serializar el objeto tú mismo. Esta opción suele ser mucho menos trabajo que crear un exploit desde cero.
