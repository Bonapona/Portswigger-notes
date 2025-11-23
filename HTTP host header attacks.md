El host header es una cabecera de HTTP/1.1 que pone a donde va dirigida la request, sirve para ayudar a saber a que back-end va dirigida la request

Virtual host -> un back-end una IP muchos dominios
Routing traffic via an intermediary -> varios back-ends , un proxy (una IP) , muchas web

## Intro a Host header attacks

básicamente si el servidor coge el input del Host directamente para usarlo ta mal
Ejemplo:

un server que para craftear una URL hace esto : 

`<a href="https://_SERVER['HOST']/support">Contact support</a>`


## Como testear este ataque

### Básico

Probar a cambiar el host header a algo random

En caso de que te redirija a la misma página que estás significa que el server tiene como página de default, en caso de que el header esté mal,la web en la que estás

### Bypass de bloqueo 

Si al ponerlo no te sale ningún mensaje de `Invalid Host header` si no que la request está siendo bloqueada por algún tipo de medida de seguridad habrá que entender como parsea la web el header, ejemplos  :

1.
La web valida en el header el nombre del dominio pero no el puerto, podríamos inyectar algo así : 

```
GET /example HTTP/1.1 
Host: vulnerable-website.com:bad-stuff-here
```

2.
Podría darse que el server revisase si en el host está la palabra : `vulnerable-website.com` pero no revisase si hay algo mas aparte de ese string, por ende podríamos explotarlo de esta manera:

```
GET /example HTTP/1.1 
Host: notvulnerable-website.com
```

3.
También podríamos hacer uso de algún subdominio ya hackeado anteriormente : 

```
GET /example HTTP/1.1 
Host: hacked-subdomain.vulnerable-website.com
```



#### Inject duplicate Host headers

Pa ver como se comporta el servidor al enviar dos headers iguales, veremos a ver cual es que tiene preferencia o si hay algun tipo de discrepancia con las tecnologías que usa : 

```
GET /example HTTP/1.1 
Host: vulnerable-website.com 
Host: bad-stuff-here
```

Por ejemplo imeginemos que el front-end valida el primer header haciendo que nuestra request pase como valida al back-end, pero el back-end hace caso del segundo header inyectando nuesto payload

#### Supply an absolute URL

Hay servers que suelen estar preparados para entender una url entera, si le ponemos a url entera y además el host header podríamos provocar alguna discrepancia de como el server lo interpreta y poder explotarla:

```
GET https://vulnerable-website.com/ HTTP/1.1 
Host: bad-stuff-here
```

Nota: el como se comporta el servidor con respecto a esto puede cambiar según si la url proporcionada usa http o https así que deberemos probara con ambas 

#### Add line wrapping

```
GET /example HTTP/1.1 
	Host: bad-stuff-here 
Host: vulnerable-website.com
```

Además de poder explotar discrepancias con esto podría darse el caso de que la web bloqueara las peticiones con dos host headers y podría bypassearse con esta técnica


### Inject host override headers

Si no consigues reescribir el host header con las técnicas anteriores, podrías intentar hacer verb tampering.

En arquitecturas donde se usa un intermediaro como una reverse proxy el server podría recibir el host header de la proxy y no del dominio del cual fue enviada la petición originalmente, por ello algunos servidores usan el header `X-Forwarded-Host`
donde se podrá el dominio original, esto se procesrá aunque el `X-Forwarded-Host` no lo ponga el front-end.
 
Ejemplos:
```
GET /example HTTP/1.1 
Host: vulnerable-website.com 
X-Forwarded-Host: bad-stuff-here
```

Aunque etse header suele ser el usado en estas ocasiones también puede darse el caso de que se usen estos otros headers : 
- `X-Host`
- `X-Forwarded-Server`
- `X-HTTP-Host-Override`
- `Forwarded`


## How to exploit the HTTP Host header

### Password reset poisoning

Attackers can sometimes use the Host header for password reset poisoning attacks.

### Web cache poisoning via the Host header

Si el host header se refleja en una web que usa caché podremos intentar inyectar un xss 

LAB: duplicas los headers y el último se le hace una petición de un script, así que le metes el exploit server y pillará el script de tu exploit server

### Accessing restricted functionality

LAB: te deja meterte en /admin si eres usuario en localhost por ende cambias el host a localhost y ya xd

### Routing-based SSRF

SSRF directamente al host header , se prueba fácil poniendo el burp collaborator y viendo si tenemos alguna request dns luego para explotarlo identificaremos IP de dentro si no lo conseguimos encontrar ni haciendo scan de los dominios ni na pues podríamos simplemente hacer brute forcing de rangos de IP's como : `192.168.0.0/16` (del 0 al 255)

LAB:
```
GET https://0aaf00740326bea68075b37c00a00064.web-security-academy.net/ HTTP/2
Host: 1tvg8hjq0ql4r3kjkis0f5wpfgl795xu.oastify.com
```

Permitía SSRF si en el GET se le ponía la url entera


### Connection state attacks

En un principio un servidor puede parecer que tiene el host header seguro, pero también puede ser que el servidor solo ponga medidas de seguridad en las primeras request de la conexion tcp del host header, es decir que si en una misma conexión le metemos dos request peude que el servidor solo valida el host header de la primera puesto que asume que ese valor no puede cambiar cosa que con burp se puede hacer

Para explotarlo pillaremos la request la duplicaremos y luego en el header connection lo pondremos en `keep-alive` luego hacer un grupo con las dos request y mandarlas en una misma conexión tcp
