# Guía Docker

## Tabla de contenidos

- [Enlaces](#enlaces)
- [Comandos más comunes](#comandos-más-comunes)
    - [Uso de Tags para descargar distintas versiones](#uso-de-tags-para-descargar-distintas-versiones)
    - [Lanzar contenedor basado en imagen](#lanzar-contenedor-basado-en-imagen)
    - [Ejecutar Contenedor](#ejecutar-contenedor)
    - [Detener contenedor](#detener-contenedor)
    - [Logs de un contenedor](#logs-de-un-contenedor)
    - [Ejecutar un comando en un contenedor en ejecución](#ejecutar-un-comando-en-un-contenedor-en-ejecución)
    - [Manejar varios contenedores en un sólo comando](#manejar-varios-contenedores-en-un-sólo-comando)
- [Desarrollo de una aplicación con Docker](#desarrollo-de-una-aplicación-web-con-docker)
    - [Crear archivo Dockerfile](#crear-archivo-dockerfile)
    - [Construir el contenedor a partir del Dockerfile](#construir-el-contenedor-a-partir-del-dockerfile)
    - [Lanzar la aplicación a partir del contenedor](#lanzar-la-aplicación-a-partir-del-contenedor)
    - [Persistencia de datos sobre el contenedor](#persistencia-de-datos-sobre-el-contenedor)
    - [Creación de red Docker para que ambos estén en la misma red](#creación-de-red-docker-para-que-ambos-estén-en-la-misma-red)
- [Docker Compose](#docker-compose)
    - [Cómo ejecutar `docker compose`](#cómo-ejecutar-docker-compose)
    - [Borrar todo lo ejecutado con `docker compose`](#borrar-todo-lo-ejecutado-con-docker-compose)


---

## Enlaces
 - Web oficial: https://www.docker.com/
 - Repositorio Docker Hub: https://hub.docker.com/

## Comandos más comunes

- `docker run` - Descargar y ejecutar contenedor
- `docker pull` - Descargar sin ejecutar contenedor
- `docker images | head` - Muestra todas las imágenes descargadas en el PC (`head` sólo muestra las primeras).
- `docker ps` - Mostrar contenedores en ejecución
- `docker ps -a` - Muestra contenedores que fueron ejecutados con anterioridad. Debido al *garbage collector* es posible que no se muestren todos.

### Uso de Tags para descargar distintas versiones

`$ docker pull alpine:latest`

`$ docker pull alpine:3.16.0`

Comprobar resultado:

`$ docker images`

### Lanzar contenedor basado en imagen

En este caso se utiliza la imagen `alpine` y su versión `latest`.

`$ docker run -di alpine:latest`

- `-di` - Interactiva en segundo plano

Se puede indicar un nombre específico si se desea

`$ docker run -di --name nombre_ejemplo alpine:latest`

*El nombre del contenedor puede ser propio si se usa* `--name nombre_propio`*. En caso contrario, el servicio docker generará uno de forma automática.*

### Ejecutar contenedor

En el caso de que el contenedor ya exista y se haya detenido previamente.

`$ docker start container_id`

### Detener contenedor

`$ docker stop container_id`

`$ docker stop container_name`

### Logs de un contenedor

Para mostrar los logs de un contenedor.

`$ docker logs container_id`

`$ docker logs container_name`

`$ docker logs -f container_id` (Hace lo mismo pero la pantalla no avanza.

### Ejecutar un comando en un contenedor en ejecución

`$ docker exec`

*Diferente a* `docker run` *porque ejecuta un comando dentro de un contenedor que ya está corriendo.*

Ejemplo:

`$ docker exec -it container_id sh`

- `it` - Sesión interactiva sobre una terminal `sh`.

### Manejar varios contenedores en un sólo comando

Indicando el comando y a continuación los identificadores o nombres de los contenedores.

`$ docker stop id_1 id_2`

---

## Desarrollo de una aplicación web con Docker

Aplicación basada en node.

### Crear archivo Dockerfile

Dentro del directorio de la aplicación.

- `FROM` - Indica qué imagen se va a utilizar y su versión. Conviene especificar versión para que el contenedor funcione si la versión de la imagen se modifica. Recomendación de utilizar imágenes basadas en `alpine` (como `tag`) por su poco *overhead* agregado a las aplicaciones.
- `WORKDIR` - Declara la ruta en la que se va a trabajar dentro del contenedor, si no existe, se creará.
- `COPY` - Copia todos los archivos que se le indiquen a una ruta determinada del contenedor. `COPY . .` copia todos los archivos de nuestra carpeta al `WORKDIR`.
- `RUN` - Comando que se ejecuta al construir la imagen del contenedor.
- `CMD` - Comando que se ejecuta por defecto cuando se ejecuta la imagen del contenedor
- `ENTRYPOINT` - Permite dejar abierto un comando para pasarle argumentos. Al hacer `docker run` habrá que pasarle un argumento, tal que: `docker run argumento`.

### Construir el contenedor a partir del Dockerfile

`$ docker build - t nombre_a_elegir .`

Conviene asignar un nombre, en caso contrario habrá que utilizar el `ID` del contenedor.

### Lanzar la aplicación a partir del contenedor

Si la aplicación se va a ejecutar sobre un puerto, por ejemplo `8080`. Cuando Docker ejecute el contenedor, el puerto `8080` se corresponde a la red interna de Docker y no al puerto de nuestro `host`. Para ello hay que indicar un mapeo a la hora de lanzar la imagen del contenedor.

`$ docker run -p 49160:8080 -d node-web-app`

Comprobaciones:
- `$ docker ps`
- `$ docker logs node-web-app`
- `$ curl -i localhost:49160`

### Persistencia de datos sobre el contenedor

Por defecto, todos los cambios que se realicen dentro de un contenedor, se perderán al detenerlo.

Para mantener los cambios:

`$ docker -v ruta_host:ruta_contenedor`

La compartición de archivos es bidireccional:
- Si se modifica en el host, se modifica en el contenedor.
- Si se modifica en el contenedor, se modifica en el host.

Aunque el `ID` del contenedor sea distinto, el almacenamiento será persistente.

Combinando con lo anterior:

`$ docker run -d -v /home/alberto/docker/docker-app/:/usr/src/app -p 49160:8080 alberto/node-web-app`

Si se modifica la imagen y se lanza de nuevo el contenedor, la actualización de la modificación sucede de forma inmediata.

Esto es muy útil cuando se usan bases de datos sobre Docker.

Una vez hechos todos los cambios se puede generar una nueva imagen.

`$ docker build . -t alberto/node-web-app_v2`

Varios pasos utilizarán la cache porque Docker reconoce dónde se han realizado los cambios.

### Creación de red Docker para que ambos estén en la misma red

Normalmente, para la ejecución de una aplicación se utilizan varios contenedores a la vez. Puede ser que un contenedor ejecute la aplicación web y otro la base de datos. Estos contenedores deben interactuar entre ellos.

Ejemplo: `mysql` y aplicación web.

Para crer una red en Docker:

`$ docker network create nombre_red`

Además del nombre para identificar a la red, se puede crear un alias para identificar la red. Este alias permite buscar la IP de este contenedor dentro de la red. Asegura que siempre se apunta al mismo contenedor y no afecta que la IP se modifique por cualquier motivo.

`$ docker run -d --network nombre_a_elegir --network-alias mi_nombre`

## Docker Compose

Para evitar tener que recordar todos los elementos que hay que introducir en los comandos, se creó `docker-compose`.

En este caso el archivo a modificar es `docker-compose.yaml`, pero previamente es necesario instalar docker-compose si no ha sido instalado previamente.

Tiene dos secciones relevantes:
- `version` - La versión de la sintaxis de lo que se está incluyendo en el archivo `yaml`.
- `services` - Se declaran los servicios que se van a ejecutar.

Esto permite ejecutar varios contenedores con un sólo archivo y automáticamente en la misma red.

Automáticamente, `docker-compose` crea una nueva red cada vez que se crea un archivo `docker-compose.yaml` donde se incluirán todos los contenedores en la misma red.

### Cómo ejecutar `docker compose`

`$ docker compose up -d`

 - `up -d` Levanta todos los servicios en segundo plano.

Se puede ejecutar las veces que se desee porque si hay algo ejecutándose no se realizará ninguna acción, pero si un servicio se ha caído, al ejecutar de nuevo el comando se levantará.

### Borrar todo lo ejecutado con `docker compose`

`$ docker compose down`

Borrará todo el contenido y la red, pero los datos persistirán si se comparte un volumen.