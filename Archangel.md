
![[Pasted image 20251228202441.png]]
Reconocimiento, vemos que tenemos el puerto 22 y el puerto 80 abiertos asi que voy a enumerar la pagina web

![[Pasted image 20251228202530.png]]
Tenemos esta pagina, no veo nada interesante en ella 

![[Pasted image 20251228202811.png]]
Enumerando los directorios encontre estos que la verdad no tenian nada de especial, el directorio flags tenia un simple trolleo

![[Pasted image 20251228204212.png]]
Enumerando la pagina web mas a profundidad encontre este subdomino, asi que lo voy a agregar al /etc/hosts para ver que contiene

![[Pasted image 20251228204334.png]]
Parece ser una pagina en desarrollo y una flag 

![[Pasted image 20251228204513.png]]
enumerando los directorios de la pagina web, encontre que tiene un robots.txt y un test.php

![[Pasted image 20251228204604.png]]
Cuando abrimos el directorio test.php nos aparece este boton

![[Pasted image 20251228204647.png]]
Al presionarlo nos muestra ese mensaje y la url se transforma y parece vulnerable

![[Pasted image 20251228210031.png]]
Aplicanco ese filtro en la URL podemos transformar el codigo fuente del archivo mrrobot.php a base 64 para llevarnoslo a nuestra maquina y verlo en texto claro 

(La imagen se me borro pero el codigo fuente de ese archivo era un echo que mostraba el mensaje que no aparecia cuando presionabamos el boton)

![[Pasted image 20251228215840.png]]
Aplicando el mismo concepto me lleve la peticion a burpsuite por comodidad y cambie mrrobot por el test.php para ver su codigo fuente 

![[Pasted image 20251228220020.png]]
Aqui tenemos el codigo fuente de test.php en resumen el codigo nos esta diciendo que no podemos usar ../ y tampoco podemos salir de la ruta que tiene definida que es /var/www/html/development_testing, el codigo dice que no podemos usar ../ pero investigando encontre que hay otra manera de moverse entre directorios que es con ..// con dos barras 

![[Pasted image 20251228223459.png]]
Poniendo ese concepto en prueba pude bypassear la restriccion y con el path predefinido y yendome unos directorios atras pude listar el /etc/passwd asi que tenemos LFI (Local File Inclusion)

![[Pasted image 20251228224613.png]]
Al darme cuenta que estamos en un entorno linux y la pagina esta corriendo en un servidor apache utilice fuzzing para ver los access.log, asi que me fijo y veo que guarda el user-agent, nos podemos aprovechar de esto para meter codigo malicioso en user-agent

![[Pasted image 20251228224907.png]]
Cargamos este codigo  malicioso en el user-agent

![[Pasted image 20251228225131.png]]
Ahora vemos que al ejecutar whoami abajo nos aparece www-data asi que ya tenemos RCE, voy a aprovechar esto para tener una shell

![[Pasted image 20251228225829.png]]
Donde decia whoami ponemos este comando que es para que nos mande una reverse shell a nuestra ip por el puerto 443

![[Pasted image 20251228225908.png]]
Ya estamos dentro de la maquina

![[Pasted image 20251228230859.png]]
Enumerando un rato la maquina encontre esta tarea cron que ejecuta un script como arcangel y al revisar los permisos del script veo que yo tambien puedo editarlo asi que lo voy a editar para mandarme una reverse shell y tranformarme en archangel

![[Pasted image 20251228231712.png]]
Edito el archivo con este codigo

![[Pasted image 20251228231744.png]]
ahora somos Archangel, tenemos que seguir elevando nuestros privilegios

![[Pasted image 20251228232601.png]]
Dentro del directorio de archangel encontre este script que tiene permisos SUID y lo ejecuta root 

![[Pasted image 20251228232655.png]]
Al leer lo que hace el script con strings encontre esta linea que utiliza cp para copiar todo lo que este en /home/usr/archangel/myfiles/ y lo inserta dentro de /opt/backupfiles

lo que me di cuenta que el script no esta llamando a cp con su ruta absoluta por ejemplo /usr/bin/cp lo que lo hace vulnerabe a un path hijacking

voy a intentar eso

Primero nos metemos en el directorio tmp 

![[Pasted image 20251228233324.png]]
Creamos nuestro ejecutable pero lo llamamos cp para que cuando el sistema busque sea el nuestro

![[Pasted image 20251228233415.png]]
Le damos permisos de ejecucion


![[Pasted image 20251228233500.png]]
Ahora modificamos el PATH para que le diga al sistema que cuando busque el script busque en nuestro directorio 

![[Pasted image 20251228233610.png]]
Lo ejecutamos y ahora somos root, PWNED

# Que aprendi

### 1. Enumeración y LFI (Local File Inclusion)

Lo primero fue detectar que podías leer archivos del servidor, pero con obstáculos.

- **LFI Básico:** Aprendiste que si un parámetro (como `view=`) acepta rutas de archivos, es vulnerable.
    
- **PHP Wrappers (Lectura de Código):** Aprendiste que no basta con intentar leer `/etc/passwd`. Usar `php://filter/convert.base64-encode/resource=archivo.php` es vital para **leer el código fuente** sin que el servidor lo ejecute. Esto te permitió ver la lógica interna y las contraseñas/pistas ocultas.
    
- **Bypass de Filtros (WAF Evasion):**
    
    - Analizaste el código PHP (`test.php`) para entender las reglas: "Debe contener `/var/www/...`" y "No debe contener `../..`".
        
    - Aprendiste a saltar la restricción usando **`..//..//`** (doble barra), una técnica clásica donde PHP limpia la ruta pero el filtro de texto falla al detectarla.
        

### 2. RCE mediante Log Poisoning

Esta fue la parte más técnica y donde más aprendiste sobre cómo funciona Apache.

- **El Concepto:** Convertir una vulnerabilidad de "Solo Lectura" (LFI) en "Ejecución de Código" (RCE) usando los archivos de registro.
    
- **User-Agent Injection:** Aprendiste que el servidor guarda tu navegador (User-Agent) en el `access.log`. Al cambiar tu nombre por código PHP (`<?php system(...) ?>`), el servidor guarda la "bomba" en su disco.
    
- **Manejo de Errores (El Error 500):** Una lección dolorosa pero importante: **los logs se corrompen**. Aprendiste que si inyectas código mal escrito, rompes el archivo entero y PHP crashea. La solución es ser preciso o reiniciar la máquina.
    

### 3. Stabilizing Shell (Tratamiento de la TTY)

Pasaste de tener una conexión inestable que se cerraba con `Ctrl+C` a una terminal completa.

- **Comando clave:** `python3 -c 'import pty; pty.spawn("/bin/bash")'`
    
- **Ajustes:** Usar `stty raw -echo` y `export TERM=xterm` para tener autocompletado, colores y poder usar editores como `nano` o `vi`.
    

### 4. Escalada de Privilegios (Horizontal y Vertical)

Aprendiste dos formas distintas de elevar tus permisos en Linux.

- **Horizontal (A usuario `archangel`): Cron Jobs**
    
    - Viste una tarea programada en `/etc/crontab` que se ejecutaba cada minuto.
        
    - **La vulnerabilidad:** El archivo script (`helloworld.sh`) tenía permisos de escritura para "otros" (o sea, tú).
        
    - **La explotación:** Inyectar una reverse shell en ese archivo y esperar a que el sistema lo ejecute por ti.
        
- **Vertical (A usuario `root`): PATH Hijacking**
    
    - Encontraste un binario SUID que ejecutaba el comando `cp` sin ruta absoluta (sin `/bin/cp`).
        
    - **La vulnerabilidad:** El sistema no sabe dónde está `cp`, así que busca en las carpetas definidas en la variable `$PATH`.
        
    - **La explotación:**
        
        1. Crear un archivo falso llamado `cp` que en realidad lanza una shell `/bin/bash`.
            
        2. Modificar la variable: `export PATH=/tmp:$PATH` (o donde esté tu archivo falso).
            
        3. Al ejecutar el programa, el sistema usa _tu_ `cp` falso con permisos de Root.