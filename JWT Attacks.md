`jwt.io` es una web que ayuda con este tipo de ataques
## JWT format

Estos tokens se dividen en 3 partes :

1. Header
2. Payload
3. signature

Ejemeplo:

```JSON
eyJraWQiOiI5MTM2ZGRiMy1jYjBhLTRhMTktYTA3ZS1lYWRmNWE0NGM4YjUiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTY0ODAzNzE2NCwibmFtZSI6IkNhcmxvcyBNb250b3lhIiwic3ViIjoiY2FybG9zIiwicm9sZSI6ImJsb2dfYXV0aG9yIiwiZW1haWwiOiJjYXJsb3NAY2FybG9zLW1vbnRveWEubmV0IiwiaWF0IjoxNTE2MjM5MDIyfQ.SYZBPIBg2CRjXAJ8vCER0LA_ENjII1JakvNQoP-Hw6GG1zfl4JyngsZReIfqRvIAEi5L4HV0q7_9qGhQZvy9ZdxEJbwTxRs_6Lb-fZTDpW6lKYNdMyjw45_alSCZ1fypsMWz_2mTpQzil0lOtps5Ei_z7mM7M8gCwe_AGpI53JxduQOaB5HkT5gVrv9cKu9CsW5MS6ZbqYXpGyOG5ehoxqm8DL5tFYaW3lB50ELxi0KsuTKEbD0t5BCl0aCR2MBJWAbN-xeLwEenaqBiwPVvKixYleeDQiBEIylFdNNIMviKRgXiYuAvMziVPbwSgkZVHeEdF5MQP1Oe2Spac-6IfA
```

### Header

El header no es más que objetos JSON encodeados en base64 y url, este header tiene metadata sobre el token, por ejemplo el header deljwt de arriba es : 

```JSON
{ 
	"iss": "portswigger",
	"exp": 1648037164, 
	"name": "Carlos Montoya", 
	"sub": "carlos", 
	"role": "blog_author", 
	"email": "carlos@carlos-montoya.net", 
	"iat": 1516239022 
}
```

### Signature

Normalmente el servidor genera una signature hasheando el header y el payload, en otros casos después de hacer esto cifran el hash correspondiente 

```
Nota: En la vida real no se suele usar JWT tal cual, si no JWS(JSON Web Signature) o JWE (JSON Web Encryption) que son derivados de JWT
```


## Exploiting flawed JWT signature verification

### Básico

Por defecto los servidores no suelen guardar información sobre los tokens JWT, si no que los propios tokens son los que guardan la información, por ende si el servidor no comprueba bien la firma del token un atacante puede modificar el JWT y hacerlo pasar por verdadero

Ejemplo:

```JSON
{ 
	"username": "carlos", 
	"isAdmin": false 
}
```


### Accepting arbitrary signatures

Hay veces que los developers al escribir el código usan la misma función para verificar y decodear el token, por ejemplo en Node.js si importas la librería `jsonwebtoken` tendrás las funciones `verify()` y `decode()` , si el developer se confunció y en vez de poner `verify()` puso `decode()`, la firma nunca será comprobada, esto admitirá cualquier tipo de firma en el token

### Accepting tokens with no signature

En el header siempre hay un campo llamado `alg` que le dice al servidor que algoritmo se usó para firmar el token y por ende cual tiene que usar para verificarlo

```JSON
{ 
	"alg": "HS256", 
	"typ": "JWT" 
}
```

Como esto está en la parte del usuario un atacante podría mandar un token que dijese que el JWT no está firmado, muchos servidores rechaazn este tipo de tokens, pero si está mal configurado un server podría aceptar algo así : 

```JSON
{ 
	"alg": "none", 
	"typ": "JWT" 
}
```


```
Nota: Para hacer este ataque habrá que poner none en alg y en el payload cambiar el usuario a administrator, además hay que eliminar toda la parte de la firma hasta el punto, ejemplo:

eyJraWQiOiI0ZWM4YzhmZi1hZDU1LTRkMTctODBhMC04ZjJiNTNhM2FjMzQiLCJhbGciOiJub25lIn0%3d.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc2NTEzNjEwNywic3ViIjoiYWRtaW5pc3RyYXRvciJ9.
```

## Brute-forcing secret keys

Wordlist: https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list 

Crackeamos el JWT con hashcat : 

```
hashcat -a 0 -m 16500 <jwt> <wordlist>
```


Una vez bruteforceado el secreto nos vamos a jwt.io en la parte de decoder, ponemos el secreto y los campos que queremos modificar 

![[Pasted image 20251207201106.png]]

## JWT header parameter injections

Otros parámetros que pueden tener los headers son : 

- `jwk` (JSON Web Key) - Provides an embedded JSON object representing the key.
 
- `jku` (JSON Web Key Set URL) - Provides a URL from which servers can fetch a set of keys containing the correct key.

- `kid` (Key ID) - Provides an ID that servers can use to identify the correct key in cases where there are multiple keys to choose from. Depending on the format of the key, this may have a matching `kid` parameter.
### jwk header

Imaginemos este header :

```JSON
{ 
	"kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG", 
	"typ": "JWT", 
	"alg": "RS256", 
	"jwk": { 
		"kty": "RSA", 
		"e": "AQAB", 
		"kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG", 
		"n": "yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9m"
	 } 
}
```

Idealmente un servidor solo usará una whitelist de public keys para verificar el token, pero a veces los servidores usan cualquier llave que esté embebida en el parámetro jwk 

1.

![[Pasted image 20251207202609.png]]
2.
![[Pasted image 20251207202623.png]]
3.Darle a generate y ok
![[Pasted image 20251207202704.png]]
4.
![[Pasted image 20251207202730.png]]
5.
![[Pasted image 20251207202752.png]]
6.Attack ->embebed jwk
![[Pasted image 20251207202842.png]]
7.ok
![[Pasted image 20251207202907.png]]

Y así conseguimos un JWT token válido, todo esto con el usuario administrator cambiado en el payload

### jku header

A veces los jwt tokens están expuestos en directorios como `/.well-known/jwks.json`

1. En JWT Editor generamos una nueva llave RSA 
2. En nuestro servidor malicioso metemos este body :
```
{ 
	"keys": [
		 <RSA KEY>
	] 
}
```
2. Cambia el valor del parámetro kid de tu token al de la llave RSA generada
3. Añade un nuevo parámetro `jku` en el header y pon en el valor la url del servidor malicioso
4. En la extensión JSON Web Token pulsa "sign"
![[Pasted image 20251207204721.png]]
5. pulsamos ok
![[Pasted image 20251207204743.png]]

### kid header

Son vulnerables a path traversal para verificar la firma con un archivo:

```JSON
{ 
	"kid": "../../path/to/file", 
	"typ": "JWT", 
	"alg": "HS256", 
	"k": "asGsADas3421-dfh9DGN-AFDFDbasfd8-anfjkvc" 
}
```

Si hacemos path traversal a `/dev/null` entonces el token se firmará con un string vacío 

La symmetric key con la que firmemos el token se tiene que ver así en el campo k :

![[Pasted image 20251207211446.png]]
Esto es porque AA== es un carácter nulo 
