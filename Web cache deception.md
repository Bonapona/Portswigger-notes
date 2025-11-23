
## Web caches

Web cache es un sistema por el cual cuando un usuario hace una request a un recurso estático la request pasa primero por el cache y verifica si tiene alguna copia del recurso pedido(cache hit), si no la tiene(cache miss) entonces manda la petición al servidor y lo que devuelve se guarda en el cache (donde se guarda lo decide varias reglas preconfiguradas) y luego lo devuelve al usuario
<img width="1256" height="333" alt="image" src="https://github.com/user-attachments/assets/457c15f3-5944-4660-970b-c8308bb4c4ae" />



### Cache rules

son reglas que pone el caché para que una request sea cacheada o no , no siempre son las mismas ni son estas

- Static file extension rules - These rules match the file extension of the requested resource, for example `.css` for stylesheets or `.js` for JavaScript files.
- Static directory rules - These rules match all URL paths that start with a specific prefix. These are often used to target specific directories that contain only static resources, for example `/static` or `/assets`.
- File name rules - These rules match specific file names to target files that are universally required for web operations and change rarely, such as `robots.txt` and `favicon.ico`.

## Detecting cached responses

Para ver si se usa el cache en la respuesta hay que fijarse en los headers de la respuesta (e.j `X-Cache: hit` , `Cache-Control` ) y en los tiempos de respuesta (si hay mucha variación en los tiempos de respuesta de la misma request puede ser porque la primera request no estaba en cache y tardas más en recibirla y la segunda request al estar ya en cache es más rápida) 

## Exploiting static extension cache rules

Cache rules often target static resources by matching common file extensions like `.css` or `.js`. This is the default behavior in most CDNs.

If there are discrepancies in how the cache and origin server map the URL path to resources or use delimiters, an attacker may be able to craft a request for a dynamic resource with a static extension that is ignored by the origin server but viewed by the cache.

## Using path mapping discrepancies (LAB)

URL path mapping es el proceso en el que se le asocia una URL a un recurso del servidor, hay dos muy comunes:

traditional URL mapping: `http://example.com/path/in/filesystem/resource.html`

RESTful URL mapping (API):
`http://example.com/path/resource/param1/param2`

- `http://example.com` points to the server.
- `/path/resource/` is an endpoint representing a resource.
- `param1` and `param2` are path parameters used by the server to process the request.
Discrepancies in how the cache and origin server map the URL path to resources can result in web cache deception vulnerabilities. Consider the following example:

`http://example.com/user/123/profile/wcd.css`

- An origin server using REST-style URL mapping may interpret this as a request for the `/user/123/profile` endpoint and returns the profile information for user `123`, ignoring `wcd.css` as a non-significant parameter.
- A cache that uses traditional URL mapping may view this as a request for a file named `wcd.css` located in the `/profile` directory under `/user/123`. It interprets the URL path as `/user/123/profile/wcd.css`. If the cache is configured to store resp

Para saber que tipo de mapeo usa si una consulta con una url da una cosa y al añadirle un /directorio da la misma que antes, por ende ignora lo último , estaríamos ante una RESTful: ejemplo : 

`/api/orders/123` y `/api/orders/123/foo` -> devuelven lo mismo

Ahora para saber como funciona el cache con la url intentaremos matchear una regla del cache intentando que se interprete como contenido estático, ejemplo : 
`/api/orders/123/foo` -> `/api/orders/123/foo.js` 

Si la respuesta da indicios de usar cache es porque el cache acepta toda la ulr con tal de que el contenido sea estático y también que deja pasar a las extensiones .js, si no funciona habrá que fuzzear extensiones para ver si alguna es aceptada por el cache

CONCLUSIÓN: 

sabemos que usa un mapeo de la url de tipo RESTful por ende la petición que hagamos sera la misma a `/api/orders/123` que a `/api/orders/123/foo.js`
la diferencia entre esas dos urls es que la que acaba en `/foo.js` será aceptada por el cache pensando que es contenido estático cuando en realidad es contenido dinámico

``` 
Note

Burp Scanner automatically detects web cache deception vulnerabilities that are caused by path mapping discrepancies during audits. You can also use the Web Cache Deception Scanner BApp to detect misconfigured web caches.
```

Con todo esto la explotación se basaría en saber donde se almacena información sensible de un usuario autenticado cacheada y con eso para hacer a otra persona que primero guarde en cache esa info sensible y que nosotros podamos recuperarla:

<img width="1914" height="914" alt="image" src="https://github.com/user-attachments/assets/56bae085-e3b3-4faa-a227-e4a3d8a43663" />


en este laboratorio vemos que al meter nuestro usuario y contraseña nos da una API key gracias al endpoint  /my-account si cambiamos ese endpoint a /my-account/caca.js vemos que la respeusta es la misma y encima está cacheada : 

![[Pasted image 20250820222709.png]]

vemos que esthttp://example.com/path/resource/param1/param2￼￼
á en estado miss por ende el contenido aparentemente estático está ahora guardado en cache y si volvemos a hacer esa misma petición veremos que ahora el cache dira hit por ende la respuesta estará en cache, con esto lo que queremos hacer es que un usuario autenticado ejecute la url que nosotros hemos hecho para que sus datos se guarden en cache y que al acceder nosotros a esa url el cache nos devuelva los datos del usuario, para ello haremos un exploit en el exploit server que nos deja portswinger para poder craftear una url nueva que se le mandara a un usuario para poder nosotros pillar su API key :

![[Pasted image 20250820223329.png]]
después de mandar el exploit a la victima usamos esa url y vemos que efetivamente la respuesta se guardo en cache y hemos conseguido robarle la API key

## Using delimiter discrepancies

### Delimiter discrepancies

Delimitadores en URLs:

/profile?id=123
/profile#section1
/profile;lang=es
___
java spring
___
el `;` es de algunos frameworks como java spring , si el servidor usa ese framework y nosotros colocamos en la url `/profile;foo.css ` el servidor lo interpretaría como `/profile` puesto que todo lo de detrás del `;` lo consideraría como una variable peeero si un usuario visita `/profile;foo.css` como tiene la extensión `.css` el cache lo interpreta como contenido estático y lo cachea , pero el servidor lo interpreta como dinámico puesto que le llega solo `/profile`

____
ruby on rails
_____

Lo mismo pasa con el framework ruby on rails que usa el punto para indicar el formato de respuesta :
/profile -> lo devuelve como html (default)
/profile.ico -> como no hay un .ico de profile pues ruby lo devuelve como html pero si el cache acepta extensiones .ico entonces la respeusta se cachea
_____
OpenLiteSpeed
____

Pasa lo mismo con OpenLiteSpeed si hacemos la request a : `/profile%00foo.js` 
### Exploiting delimiter discrepancies

para ver que framework usa partiendo de usat url:  `/settings/users/list`

primero haremos una request que de error para tomarla como referencia en las siguientes pruebas : `/settings/users/listaaa`

```
Nota: si la request es identica a la que no da error es porque estamos siendo redirigidos así que habrá que probar con otro endpoint
```

ahora probamos con : `/settings/users/list;aaa` si la respeusta es idéntica a la que no da error es porque el símbolo `;` se usa como delimitador.

Una vez que sabemos que `/settings/users/list;aaa` será interpretado por el servidor como `/settings/users/list` probaremos extensiones para ver cuales son cacheadas como por ejemplo .js

## Using delimiter decoding discrepanciesc

### Delimiter decoding discrepancies

El problema surge cuando **el servidor de origen y el caché interpretan de forma distinta los delimitadores codificados**.

Básicamente si nosotros ponemos en la url `/profile%23wcd.css` siendo `%23`
el símbolo `#` pero encodeado en url, el servidor decodificará la petición de manera que todo lo que haya detras del `%23` lo interpretará como una variable de la url por ende devolverá `/profile` en cambio el cache no url decodeará el %23 por ende asumirá que `/profile%23wcd.css` es un archivo estático en si mismo y entonces lo cacheará 

## Using normalization discrepancies


no solo el caché es vulnerable a web cache deception si no que hay directorios en las web que se usan para guardar contenido estático como puede ser `/static`, `/assets`, `/scripts`, o `/images` puediendo ser vulnerables a los mismo que el caché
%23
Imaginemos que tenemos un path traversal y nuestro payload es este : 

`/static/..%2fprofile`

la web al ser vulnerable a path traversal y decodear el caráter devolverá lo que hay en `/profile` mientras que el caché como no decodifica el %2f asume que está guardando el archivo : ``/static/..%2fprofile`` y si en las reglas del caché deja guardar archivos provenientes de `/static` entonces se guardará asumiendo que es estático cuando no lo es.

Para saber si es vulnerable a path traversal primero haz una request a /profile y la comparas con la de /aaa/..%2fprofile si son iguales es porque hay path traversal%23

CUANDO EL CACHE SI DECODEA URL PERO LA WEB NO 

en este caso si pasa esto el path traversal no será suficiente puesto que  si nuestro payload es 

`/static%2f%2e%2e%2fprofile`

el servidor dará error puesto que no existe el archivo `/static%2f%2e%2e%2fprofile` y no se cacheará puesto que /profile no es contenido cacheable, pero si lo hacemos de reves y nuestro payload es 

`/profile%2f%2e%2e%2fstatic` veremos que el server sigue dando error y obtendremos un 404 pero la request se cacheará puesto que /static si es contenido cacheable, ahora para hacer un payload con esto la cosa cambia porque si la web usa un delimitador como `;` entonces si usamos este payload 
`/profile;%2f%2e%2e%2fstatic` 
la web devolverá /profile y la request se cacheara puesto que le llega /static

## Exploiting file name cache rules

También archivos comunes como `robots.txt`, `index.html`, y `favicon.ico` suelen ser cacheados  mira a ver si haciendo un get a alguna de esos archivos hay algun indicio de estar cacheadas
