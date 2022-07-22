# Docker: buenas prácticas

## Tabla de contenidos

- [Ejecutar contenedores con usuarios sin privilegios](#ejecutar-contenedores-con-usuarios-sin-privilegios)
- [Deshabilitar el usuario root](#deshabilitar-el-usuario-root)
- [Evitar escalada de privilegios por contenedores](#evitar-escalada-de-privilegios-por-contenedores)
- [Limitación de las capabilities del Kernel del contenedor](#limitación-de-las-capabilities-del-kernel-del-contenedor)
- [Permiso y acceso al File System](#permiso-y-acceso-al-file-system)
- [Deshabilitar comunicación entre contenedores](#deshabilitar-comunicación-entre-contenedores)

---

## Ejecutar contenedores con usuarios sin privilegios

A la hora de configurar y construir una imagen, es conveniente especificar un usuario sin privilegios dentro del `Dockerfile` mediante el siguiente comando:

`RUN groupadd -r $USER && useradd -r -g $GROUP $USER`

Por ejemplo:

`RUN groupadd -r testing && useradd -r -g testing testing`

Fragmento del Dockerfile:

```
FROM debian:bullseye-slim

LABEL maintainer="NGINX Docker Maintainers <docker-maint@nginx.com>"

RUN groupadd -r testing && useradd -r -g testing testing

ENV NGINX_VERSION   1.23.0
...
```

Para utilizar este usuario, se debe indicar al ejecutar el contenedor:

`$ docker run -u $USER -it --rm nombre_imagen /bin/bash`

Ejemplo:

`$ docker run -u testing -it --rm nombre_imagen /bin/bash`

O indicarlo en el propio `Dockerfile` a partir de la instrucción `USER`:
```
FROM debian:bullseye-slim

LABEL maintainer="NGINX Docker Maintainers <docker-maint@nginx.com>"

RUN groupadd -r testing && useradd -r -g testing testing
USER testing

ENV NGINX_VERSION   1.23.0
...
```

## Deshabilitar el usuario `root`

Una buena práctica es deshabilitar el usuario `root` a partir del archivo `Dockerfile`. Esto es posible modificando la terminal `/bin/bash` por `/usr/bin/nologin`. Esto evita que un usuario cualquier acceda al usuario `root`, aunque este tenga la contraseña del mismo.

`RUN chsh -s /usr/sbin/nologin root`

Tenga en cuenta que esto deshabilita la cuenta `root` por completo.

## Evitar escalada de privilegios por contenedores

Se recomienda ejecutar contenedores con permisos específicos para evitar que exista una escalada de privilegios. Para ello, se debe utilizar el siguiente `flag` al ejecutar el contenedor:

`$ docker run --security-opt=no-new-privileges...`

## Limitación de las capabilities del Kernel del contenedor

Se recomienda ejecutar los contenedores sin utilizar el la instrucción `--privileged` debido a que sobrescribe cualquier otro permiso y restricción de seguridad previa.
- [Información sobre las `capabilities`](https://man7.org/linux/man-pages/man7/capabilities.7.html)

Una buena práctica es descartar todas las `capabilities` del contenedor y añadir las que sean requeridas. Esto se puede hacer en el mismo comando:

`$ docker run --cap-drop all --cap-add $CAPABILITY -it --rm nombre_imagen /bin/bash`

Ejemplo:

`$ docker run --cap-drop all --cap-add NET_ADMIN -it --rm nombre_imagen /bin/bash`

## Permiso y acceso al File System

Existe la posibilidad de especificar el permiso y acceso al sistema de archivos, esto permite restringir los contenedores al modo de sólo lectura o a un sistema de archivos temporal específico. Esta práctica es interesante para fijar dónde los contenedores pueden almacenar datos y hacer cambios.

Para ejecutar un contenedor con permiso de solo lectura:

`$ docker run -ti --read-only --rm test_vuln /bin/bash`

Si se intenta crear un archivo se mostrará un mensaje de error indicando que sólo se tienen permisos de lectura sobre el file system:

```
root@d6442a51ef5c:/# cd home/
root@d6442a51ef5c:/home# touch file.text
touch: cannot touch 'file.text': Read-only file system
```

Si se requiere almacenar o modificar datos en alguna ruta en específico, se indica mediante el siguiente comando:

`$ docker run --read-only --tmpfs $RUTA_DIRECTORIO -ti --rm nombre_imagen /bin/bash`

Siendo `$RUTA_DIRECTORIO` el sistema de archivos temporal, que podría ser: `/home`, `/opt`, `/tmp`...

Ejemplo:

`$ docker run -ti --read-only --tmpfs /home/test_dir --rm nombre_imagen /bin/bash`

Aquí se indica que el sistema de archivos temporal será `/home/test_dir/` de forma que si no existe, Docker lo creará en el contenedor. Sólo en este sistema de archivos se permitirá la creación de archivos.

```
root@d6442a51ef5c:/home# cd test_dir/
root@d6442a51ef5c:/home/test_dir# touch file.txt
root@d6442a51ef5c:/home/test_dir# ls -l
total 0
-rw-r--r-- 1 root root 0 Jul  7 20:05 file.txt
```

## Deshabilitar comunicación entre contenedores

Por defecto los contenedores en Docker se ejecutan sobre una misma red común que permite la comunicación entre contenedores.

Obtener la lista de redes en Docker:

```
$ docker network list
NETWORK ID     NAME                     DRIVER    SCOPE
89c4c58c48b2   app-network              bridge    local
ed3650e99db5   bridge                   bridge    local
65319efea794   host                     host      local
482c6ba910eb   minikube                 bridge    local
a6094b95f724   none                     null      local
```

Obtener las propiedades de la red `bridge` (usada por defecto).

`$ docker inspect bridge`

Y la opción que permite la comunicación entre contenedores es la siguiente:
```
...
"com.docker.network.bridge.enable_icc": "true",
...
```

Donde `icc` significa *Inter Container Communication*.

### Prueba de comunicación entre dos contenedores (Habilitada)

Se despliegan dos contenedores sin especificar ninguna red:

- Contenedor 1: `$ docker run -it --rm nombre_imagen /bin/bash`
- Contenedor 2: `$ docker run -it --rm nombre_imagen /bin/bash`

Se deben instalar las siguientes herramientas dentro de los contenedores:
- `net-tools`: En el contenedor del que se quiera conocer la IP.
- `iputils-ping`: En el contenedor desde el que se vaya a realizar el `ping` al otro contenedor.

Suponemos en ejecución dos contenedores y las herramientas ya instaladas:
- IP contenedor 1: `172.17.0.2`
- Contenedor 2 realiza `ping` a contenedor 1:
```
root@d09bf0baa197:/# ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.258 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.259 ms
64 bytes from 172.17.0.2: icmp_seq=3 ttl=64 time=0.082 ms
^C
--- 172.17.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2027ms
rtt min/avg/max/mdev = 0.082/0.199/0.259/0.083 ms
```

Como se puede observar la comunicación entre contenedores existe al no haber especificado ninguna red y desplegarse estos por defecto en la red `bridge`.

Para evitar esta comunicación entre contenedores por defecto, es necesario crear una red propia y configurarla con la opción anterior como `false`.

Se debe ejecutar el siguiente comando:

`$ docker network create --driver bridge -o "com.docker.network.bridge.enable_icc"="false" nombre_red`

En este comando se crea una nueva red `nombre_red` con la opción `com.docker.network.bridge.enable_icc` como `false` utilizando el driver `bridge` para crear la red, aunque este último parámetro se puede omitir ya que el driver `bridge` es el utilizado por defecto a la hora de crear la red.

Para desplegar un contenedor en la nueva red se debe ejecutar el siguiente comando:

`$ docker run --network nombre_red -ti --rm nombre_imagen /bin/bash`

Ahora, la comunicación entre contenedores está deshabilitada en esta red, como se podrá ver a continuación.

### Prueba de comunicación entre dos contenedores (Deshabilitada)

Se despliegan dos contenedores sin especificar ninguna red:

- Contenedor 1: `$ docker run --network nombre_red -ti --rm nombre_imagen /bin/bash`
- Contenedor 2: `$ docker run --network nombre_red -ti --rm nombre_imagen /bin/bash`

Se deben instalar las siguientes herramientas dentro de los contenedores:
- `net-tools`: En el contenedor que se quiera conocer la IP.
- `iputils-ping`: En el contenedor desde el que se vaya a realizar el `ping` al otro contenedor.

Suponemos en ejecución dos contenedores y las herramientas ya instaladas:
- IP contenedor 1: `172.22.0.3`
- Contenedor 2 realiza `ping` a contenedor 1:
```
root@cd9ad2e896ca:/# ping 172.17.0.3
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
^C
--- 172.17.0.3 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2044ms
```

Como se puede observar todos los paquetes enviados se han perdido al realizar un `ping` puesto que ya no existe comunicación entre contenedores en la nueva red.