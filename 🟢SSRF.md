(puede salir en el refer header)
## Bypassing SSRF defenses

### SSRF with blacklist-based input filters

Si una web no te deja hacer en un SSRF  a /admin o 127.0.0.1 o localhost :

-Si no deja 127.0.0.1 o localhost -> usa `2130706433`, `017700000001` o `127.1`(otras representaciones de la ip 127.0.0.1)

-Sino funciona puedes crear tu propio dominio que resuelva a la ip 127.0.0.1 con `spoofed.burpcollaborator.net`

-Si no deja /admin puedes doble url-encodear una letra

### SSRF with whitelist-based input filters

Esto es expert, no entra 

Si la web te deja hacer SSRF cuando hay ciertas palabras o hosts:

Ejemplos:

```
https://expected-host:fakepassword@evil-host
https://evil-host#expected-host
https://expected-host.evil-host
```

### Bypassing SSRF filters via open redirection

Imaginemos que tiene un open redirection de este tipo:

`/product/nextProduct?currentProductId=6&path=http://evil-user.net` -> `http://evil-user.net`

puedes aprovecharte de esta redirección para hacer esto :

```
POST /product/stock HTTP/1.0 
Content-Type: application/x-www-form-urlencoded 
Content-Length: 118 

stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin
```


## Blind SSRF vulnerabilities

Blind es cuando no vemos el resultado de la query enviada, por ende la mejor forma de confirmar que es vulnerable es con ataques OAST, basicamente haremos que haga una petición a nuestro buro collaborator y ver si le llega algo  

## Finding hidden attack surface for SSRF vulnerabilities

### Partial URLs in requests

Sometimes, an application places only a hostname or part of a URL path into request parameters. The value submitted is then incorporated server-side into a full URL that is requested. If the value is readily recognized as a hostname or URL path, the potential attack surface might be obvious. However, exploitability as full SSRF might be limited because you do not control the entire URL that gets requested.

### URLs within data formats

Some applications transmit data in formats with a specification that allows the inclusion of URLs that might get requested by the data parser for the format. An obvious example of this is the XML data format, which has been widely used in web applications to transmit structured data from the client to the server. When an application accepts data in XML format and parses it, it might be vulnerable to XXE injection. It might also be vulnerable to SSRF via XXE. We'll cover this in more detail when we look at XXE injection vulnerabilities.

### SSRF via the Referer header

Some applications use server-side analytics software to tracks visitors. This software often logs the Referer header in requests, so it can track incoming links. Often the analytics software visits any third-party URLs that appear in the Referer header. This is typically done to analyze the contents of referring sites, including the anchor text that is used in the incoming links. As a result, the Referer header is often a useful attack surface for SSRF vulnerabilities.