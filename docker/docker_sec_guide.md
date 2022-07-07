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

`# RUN groupadd -r testing && useradd -r -g testing testing`

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

`$ docker run --cap-drop all --cap-add $CAPABILITY...`

Ejemplo:

`$ docker run --cap-drop all --cap-add NET_ADMIN -it...`

## Permiso y acceso al File System

Existe la posibilidad de especificar el permiso y acceso al sistema de archivos, esto permite restringir los contenedores al modo de sólo lectura o a un sistema de archivos temporal específico. Esta práctica es interesante para fijar dónde los contenedores pueden almacenar datos y hacer cambios.

Para ejecutar un contenedor con permiso de solo lectura:

`$ docker run --read-only...`

Si se requiere almacenar o modificar datos de alguna ruta en específico, se indica mediante el siguiente comando:

`$ docker run --read-only --tmpfs /tmp...`

Siendo `/tmp` el sistema de archivos temporal, pero podría ser cualquier otro: `/home`, `/opt`...

## Deshabilitar comunicación entre contenedores

Por defecto los contenedores en Docker se ejecuten sobre una misma red común que permite la comunicación entre contenedores. Esta opción está habilita por defecto y se puede consultar de la siguiente forma:

Obtener la lista de redes en Docker:

`$ docker network list`

Obtener las propiedades de la red `bridge` (usada por defecto).

`$ docker inspect bridge`

Y la opción mencionada anteriormente es la siguiente:
```
...
"com.docker.network.bridge.enable_icc": "true",
...
```
Donde `icc` significa Inter Container Communication.

Para evitar esta comunicación entre contenedores por defecto, es necesario crear una red propia y configurarla con la opción anterior como `false`.

Se debe ejecutar el siguiente comando:

`$ docker network create --driver bridge \-o "com.docker.network.bridge.enable_icc": "false" nombre_red`

Esta red debe ser seleccionada de forma manual a la hora de ejecutar los contenedores:

`$ docker run --network nombre_red...`
