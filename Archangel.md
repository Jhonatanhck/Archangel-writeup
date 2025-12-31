
# Writeup Archangel - TryHackMe

## Reconocimiento y Enumeración

Empezamos con un escaneo de puertos. Vemos que tenemos el puerto 22 (SSH) y el puerto 80 (HTTP) abiertos, así que voy a enumerar la página web.

![](images/Pasted%20image%2020251228202441.png)

Tenemos esta página principal, no veo nada interesante en ella a primera vista.

![](images/Pasted%20image%2020251228202530.png)

Enumerando los directorios encontré estos, que la verdad no tenían nada de especial. El directorio `flags` tenía un simple trolleo.

![](images/Pasted%20image%2020251228202811.png)

## Descubrimiento de Subdominio

Enumerando la página web más a profundidad encontré este subdominio, así que lo voy a agregar al `/etc/hosts` para ver qué contiene.

![](images/Pasted%20image%2020251228204212.png)

Al acceder al subdominio, parece ser una página en desarrollo y encontramos la primera flag.

![](images/Pasted%20image%2020251228204334.png)

Enumerando los directorios de este nuevo sitio, encontré que tiene un `robots.txt` y un `test.php`.

![](images/Pasted%20image%2020251228204513.png)

## Local File Inclusion (LFI)

Cuando abrimos el archivo `test.php` nos aparece este botón:

![](images/Pasted%20image%2020251228204604.png)

Al presionarlo, nos muestra un mensaje y la URL se transforma, añadiendo un parámetro `?view=`. Esto parece vulnerable a LFI.

![](images/Pasted%20image%2020251228204647.png)

Intenté usar filtros de PHP para leer el código fuente. Aplicando este filtro en la URL podemos transformar el código fuente del archivo `mrrobot.php` a Base64 para llevárnoslo a nuestra máquina y verlo en texto claro.

*(La imagen del resultado se borró, pero el código fuente de ese archivo era un simple `echo` que mostraba el mensaje que aparecía al presionar el botón).*

![](images/Pasted%20image%2020251228210031.png)

Aplicando el mismo concepto, me llevé la petición a Burp Suite por comodidad y cambié `mrrobot.php` por `test.php` para ver su código fuente.

![](images/Pasted%20image%2020251228215840.png)

Aquí tenemos el código fuente de `test.php`. En resumen, el código nos dice que hay un filtro que nos impide usar `../` y tampoco podemos salir de la ruta predefinida que es `/var/www/html/development_testing`.

Investigando cómo bypassear el filtro de `../`, encontré que hay otra manera de moverse entre directorios: usando `..//` (doble barra).

![](images/Pasted%20image%2020251228220020.png)

Poniendo ese concepto a prueba pude bypassear la restricción. Usando el path predefinido y yéndome unos directorios atrás con `..//`, pude listar el `/etc/passwd`. ¡Tenemos **LFI (Local File Inclusion)** confirmado!

![](images/Pasted%20image%2020251228223459.png)

## De LFI a RCE (Log Poisoning)

Al darme cuenta de que estamos en un entorno Linux y la página corre en un servidor Apache, utilicé fuzzing para encontrar los logs. Me fijé en los `access.log` y vi que guardan el **User-Agent**. Nos podemos aprovechar de esto para inyectar código PHP malicioso en el User-Agent.

![](images/Pasted%20image%2020251228224613.png)

Cargamos este código malicioso en el User-Agent mediante Burp Suite:

![](images/Pasted%20image%2020251228224907.png)

Ahora, al acceder al log mediante el LFI, vemos que al ejecutar el comando `whoami`, abajo nos aparece `www-data`. Ya tenemos **RCE (Remote Command Execution)**.

![](images/Pasted%20image%2020251228225131.png)

Voy a aprovechar esto para obtener una shell reversa. Donde pusimos `whoami`, ponemos un comando para enviarnos una reverse shell a nuestra IP por el puerto 443.

![](images/Pasted%20image%2020251228225829.png)

¡Ya estamos dentro de la máquina como `www-data`!

![](images/Pasted%20image%2020251228225908.png)

## Escalada de Privilegios (Usuario Archangel)

Enumerando un rato la máquina, encontré una tarea cron que ejecuta un script como el usuario `archangel`. Al revisar los permisos del script, veo que `www-data` tiene permisos de escritura, así que puedo editarlo para mandarme una reverse shell como archangel.

![](images/Pasted%20image%2020251228230859.png)

Edito el archivo añadiendo una reverse shell de bash:

![](images/Pasted%20image%2020251228231712.png)

Esperamos a que se ejecute la cronjob y recibimos la conexión. Ahora somos el usuario **Archangel**. Tenemos que seguir elevando privilegios.

![](images/Pasted%20image%2020251228231744.png)

## Escalada de Privilegios (Root) - PATH Hijacking

Dentro del directorio home de archangel encontré un script ejecutable que tiene permisos **SUID** y pertenece a root.

![](images/Pasted%20image%2020251228232601.png)

Al analizar lo que hace el script usando el comando `strings`, encontré una línea interesante. Utiliza el comando `cp` para copiar todo lo que esté en `/home/usr/archangel/myfiles/` e insertarlo dentro de `/opt/backupfiles`.

![](images/Pasted%20image%2020251228232655.png)

Me di cuenta de que el script **no está llamando a `cp` con su ruta absoluta** (por ejemplo `/bin/cp`), sino que usa la ruta relativa. Esto lo hace vulnerable a un ataque de **PATH Hijacking**.

Vamos a explotar esto. Primero nos metemos en el directorio `/tmp`.

![](images/Pasted%20image%2020251228233324.png)

Creamos nuestro propio ejecutable malicioso y lo llamamos `cp`, para que cuando el sistema busque el comando, encuentre el nuestro primero. En este caso, simplemente haremos que ejecute `/bin/bash`.

![](images/Pasted%20image%2020251228233415.png)

Le damos permisos de ejecución a nuestro `cp` falso.

![](images/Pasted%20image%2020251228233500.png)

Ahora modificamos la variable de entorno **PATH** para decirle al sistema que busque primero en nuestro directorio actual (`/tmp`) antes que en los directorios habituales.

![](images/Pasted%20image%2020251228233610.png)

Finalmente, ejecutamos el script SUID vulnerable. Al intentar usar `cp`, ejecutará nuestro script malicioso con permisos de root.

Ahora somos **root**. ¡PWNED!
