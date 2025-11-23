Timing
# Username enumeration via account lock

basicamente aqui si tenemos un usario valido y lo brute forceamos nos va a bloquear diciendo que tenemos demasiados intentos hechos, así que lo que haremos será hacer un ataque donde cada nombre se ponga 5 veces para ver si alguno se bloquea, esto lo conseguiremos usando clouster bomb attack y poniendo un payload en usuario y otro payload pero null en contrasella que itere 5 veces


para saltarse el time-out por hacer demasiados intentos podemos usar (en este lab)  el header `X-Forwarded-For` con el que podremos spoofear nuestra IP , despues hacemos bruteforcing y determinamos que un usuario es valido por la diferencia en el tiempo de respuesta 