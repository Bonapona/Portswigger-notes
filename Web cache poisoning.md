Para enterearte de lo que es el caché y demás mirar los apuntes de web caché deception

## PASOS

1.Ver alguna request en la web que sea cacheada

<img width="1196" height="415" alt="image" src="https://github.com/user-attachments/assets/96fd1a71-f739-4154-abef-0bfa250727b8" />

Vemos que tenemos un hit en cache por ende esta request ya estaba cacheada

2.Ver si en la request hay algún parametro que sea parte de la llave del caché
<img width="1137" height="334" alt="image" src="https://github.com/user-attachments/assets/15e28ebd-3bde-4d32-b22c-51eac7d8dd01" />

Esto lo sabremos si cuando cambiamos ese parametro y lo enviamos el caché nos devuelve miss, diciendonos que esa llave cache no coincidía con ninguna 

3.Ver algún valor de la request que se vea reflejada en la respuesta y que no sea parte de la llave caché, esto útimo es porque si conseguimos meter un XSS a una una copia de caché lo que queremos es que se guarde en la página principal no en la copia del caché especifica nuestra. 

En este laboratorio había que buscar algún unkeyed input(input que no forme parte de la llave caché) con paraminer(extensión de burp), en el caso de este laboratorio hay un header válido que es `x-forwarded-host`, este header no se usa para hacer la caché key , pero si nosotros creamos una url que haga que la request se cachee podríamos añadir el `x-forwarded-host: cosa maliciosa` haciendo que la "cosa maliciosa" se ejecute.

4.Probar donde se ve reflejado el unkeyed header en la web.
<img width="783" height="707" alt="image" src="https://github.com/user-attachments/assets/edf88762-cf8f-4057-a826-cc00217de62e" />

En este caso vemos que se refleja en un script, además vemos que completa la ruta de /resources/js/tracking.js

5.Exploit server
<img width="709" height="842" alt="image" src="https://github.com/user-attachments/assets/2c06ace9-dab9-4d8e-a04a-91e5fe444f72" />

Se le cambia el file a ` /resources/js/tracking.js` porque la request hace de normal una petición en el script a 
```
<script src="//victima.com/resources/js/tracking.js"></script>
```
Pero como gracias a X-Forwarded-Host podemos elegir que va en victima.com pues ponemos nuestro exploit server donde en /resources/js/tracking.js tendremos un XSS.

Así que primero hacemos que en mi petición a ?caca=123 el script apunte a mi servidor
<img width="1099" height="658" alt="image" src="https://github.com/user-attachments/assets/995264fb-7c25-44fe-8b7f-62c8bf347a2a" />

De esta forma al acceder a /?caca=123 el XSS se ejecuta

<img width="1049" height="686" alt="image" src="https://github.com/user-attachments/assets/5e8f3665-74a7-4461-9034-06230addc62f" />

Una vez probado esto en ?caca=123 ahora eliminaremos esa query para que se haga en el home de la web 


## Ejemplos de laboratorios

1.
el dato se veía reflejado en : 

```
data = {"host":"0a3f00fb046535a18088b2ad0096002c.web-security-academy.net","path":"/","frontend":"cosa"} 
```

para hacer el xss y escapar las comillas había que hacer "frontend":""-alert()-""}

2.

En este necesitabas dos headers ocultos pero el segundo solo lo encuentras si haces param miner con el primer header oculto puesto, porque al poner el primero cuando brutforcea el segundo el size cambia 

3.
Es un web cache poisoning normal que al ejecutarlo puesto se pone el xss y tal pero enviando el cache envenenado muchas veces no parece que el lab se resuelva, mirando la respuesta del cache envenenado vemos un header `Vary: User-Agent` que nos indica que la caché key también se hace con el user agent (cache key se hace con los headers ed la respuesta entre otras cosas)así que para que el usuario pinche en caché envenenado necesitaremos su user agent para ello vemos que en la sección de comentarios podemos inyectar un xss hacia nuestro server donde en los logs se qeudará el user agent de la víctima de esta forma explotaremos web caché poisoning

4.
Study the cache behavior. Observe that if you add duplicate `callback` parameters, only the final one is reflected in the response, but both are still keyed. However, if you append the second `callback` parameter to the `utm_content` parameter using a semicolon, it is excluded from the cache key and still overwrites the callback function in the response:
```
GET /js/geolocate.js?callback=setCountryCookie&utm_content=foo;callback=arbitraryFunction HTTP/1.1 200 OK 

X-Cache-Key: /js/geolocate.js?callback=setCountryCookie 
… 

	
arbitraryFunction({"country" : "United Kingdom"})
```

Send the request again, but this time pass in `alert(1)` as the callback function:
`GET /js/geolocate.js?callback=setCountryCookie&utm_content=foo;callback=alert`
Básicamente vemos que en callback le podemos meter un xss, pero para que le aparezca a todos callback tiene que tener setCountryCookie en el call back, asi que usamos:

```
utm_content=foo;callback=alert(1)
```
para declarar una nueva variable callback 

5.

El unkeyed input sale porque en un get deja meterle body
