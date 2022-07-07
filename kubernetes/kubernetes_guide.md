# Guía Kubernetes

- [minikube](https://minikube.sigs.k8s.io/docs/start/): Kubernetes para uso en local

- Libro recomendado: [Site Reliability Engineering](https://sre.google/sre-book/table-of-contents/)

Motivos para usar Kubernetes:
- Cuando se tienen 10-20 instancias con Docker Compose empieza a ser problemático porque docker-compose **no escala**.
- Es recomendable utilizar un servicio que maneje los contenedores. Kubernetes permite orquestar estos contenedores.

## Requisitos

- Instalar [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) (Cliente de Kubernetes)
- Iniciar cluster de kubernetes: `$ minikube start`
- No es necesario usar kubernetes en una nube.
*Los volúmenes y load balancers no se pueden probar con minikube*

Comprobar instalación: 

`$ kubectl version --short` 

`kubelet` - servicio de Kubernetes que permite conectar todos los servicios de Kubernetes entre sí.

Al utilizar `minikube`, `kubectl` apunta directaente a `minikube`.

---
## Definiciones

Contexto: Es una combinación de la URL del servidor con las credenciales que se usan para conectarse a él.

Un nodo es un recurso que ejecuta los contenedores.

`minikube` configura el archivo de configuración automáticamente. Este archivo de configuración se utiliza para conectar el clúster con `kubectl`.

---

## Añadir nodos en cluster Kubernetes

`$ minikube node add [nombre]`

## Listar nodos del cluster

`$ minikube node list`

## Listar comandos `kubectl`

`$ kubectl --help`

## Comandos más utilizados

- `get`: Para obtener recursos. Muy utilizado.
- `edit`: Para editar los recursos
- `delete`: Para borrar recursos
- `apply`: Aplicar un manifiesto de Kubernetes o configuraciones.
- `exec`: Para ejecutar un comando dentro de un contenedor (Igual que en docker)
- `logs`: Para ver los logs
- `cp`: Para copiar archivos de una máquina al contenedor o viceversa
- `port-forward`: Para reenviar puertos para acceder a un contenedor

- `cordon`, `uncordon`, `drain`: Muy útiles a la hora de manejar nodos.
    - `cordon`: Hacer que el nodo no reciba más contenedores
    - `uncordon`: Hacer que el nodo pueda recibir más contenedores
    - `drain`: Sacar todos los contenedores del nodo, para moverlos a otro sitio.


## Mostrar los contextos del archivo de configuración

`$ kubectl config get-contexts`


## Manifiestos de Kubernetes

`namespace`: División lógica del cluster de kubernetes. Permite separar la carga en el cluster. Se suelen utilizar para dividir el tráfico o los recursos (por ejemplo, a partir del uso de políticas).

Muestra los namespace por defecto de cualquier cluster:

`$ kubectl get ns`

Aquellos que comiencen por `kube-*` están reservados para las herramientas de Kubernetes

`pod`: Set de contenedores. Basado en uno o más contenedores. La mayoría de las veces los pods ejecutan un solo contenedor, pero es posible ejecutar varios.

Para mostrar los pods:

`$ kubectl -n kube-system get pods`
- `-n` Para indicar el `namespace` 

- Algunos pods tienen un hash en el nombre para diferenciarse unos de otros al haber varios iguales.
- En `READY` se muestra el número de contenedores en ejecución respecto del total.

Para ver más información de los pods: 

`$ kubectl -n kube-system get pods -o wide`

- Ver en qué nodo se ejecutan los pods o la IP.

Borrar un pod

`$ kubectl -n kube-system delete pod nombre_del_pod`

Al hacer esto nuevamente se crea uno nuevo con otro hash.

Para ver el estado y eventos de un pod:

`$ kubectl describe pod nombre_del_pod`


---

## Manifiesto para pod simple

Secciones:
- `apiVersion`: La versión de la API de este recurso de Kubernetes. Puede cambiar dependiendo del tipo de recurso (`kind`). Hay que utilizar la documentación de Kubernetes para ver qué `apiVersion` usa el tipo `pod`.

- `metadata`: Añadir datos como nombre, etiquetas. El `name` es necesario sí o sí.

- `spec`: Se compone de varias secciones:
    - `containers`: Declarar los contenedores dentro del pod. Estos pods compartirán el `namespace` de red, es decir, todos los contenedores que se ejecuten bajo el mismo pod tendrán la misma IP. Se declara el `name` y la `image` del contenedor.

Archivo de ejemplo: `01-pod.yaml`.

## Aplicar manifiesto de Kubernetes

`$ kubectl apply -f 01-pod.yaml`

Si no se aplica un `namespace` se aplicará el manifiesto en el `default`.

`$ kubectl get pods`

o 

`$ kubectl -n default get pods`

## Ejecutar comando en pod

`$ kubectl exec -it nginx -- sh`

Ejecutar `sh` dentro del pod llamado `nginx`.

*No hace falta hacer ssh al nodo. Sólo se utiliza la API de Kubernetes.* 

## Borrar un pod sin mecanismo de reinicio

`$ kubectl delete pod nginx`

No se vuelve a levantar porque no hay ninguna política referida sobre esto.


## Opciones más comunes dentro del manifiesto de un pod

- `env`: Variables de entorno. Igual que en Docker. Dentro de `env` se indica el nombre de la variable y el valor. El valor se puede heredar a partir de información contenida en Kubernetes ([`downward API`](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)), por ejemplo: `status.hosIP`.

- `resources`: Forma de asignar recursos a los contenedores. Es posible asignar diferentes recursos a distintos contenedores. Existen dos formas de asignar recursos: `requests` y `limits`.

    - `requests`: Son los recursos que se van a garantizar y siempre va a tener disponible. 
    
    - `limits`: Es el límite que el contenedor puede llegar a utilizar. Cuando se sobrepasa el límite, el Kernel de Linux mata el proceso por usar más RAM de la permitida. En el caso de la CPU, el Kernel de Linux utiliza el servicio CPU throttling para ahogar al contenedor hasta que baje su consumo.
    
    Se definen `memory` y `cpu` comúnmente:
        - `memory`: Memoria RAM (ejemplo: `64Mi`)
        - `cpu`: Medidos en unidades de CPU. Una CPU en Kubernetes equivale a 1 vCPU/Core, para asignar recursos de CPU se utiliza `100m` o `0.1` para indicar que se usan `100milicores` o `100miliCPUs` (ejemplo: `200m`). No es posible superar las capacidades del pod. Si tenemos 1 core, no podemos meter más de 5 contenedores que utilicen `200m`.

    NOTA: Es por contenedor y no por pod
    NOTA2: Ya existen pods en ejecución de Kubernetes, hay que tener en cuenta que usan recursos.

- `readinessProbe`: Para indicar a Kubernetes que el contenedor está listo para recibir tráfico. Esto es útil para aplicaciones que tarden en iniciar, así Kubernetes no mata el proceso. Kubernetes esperará un status code 200 (OK).

- `livenessProbe`: Para indicar a Kubernetes que el contenedor está activo/vivo y no se desea que se apague. Kubernetes comprobará que el puerto esté abierto

- `ports`: Para indicar el puerto del contenedor.

Archivo de ejemplo: `02-pod.yaml`

`$ kubectl apply -f 02-pod.yaml`

Para ver el estado del pod:

`$ kubeclt get pod nginx`

Para mostrar información adicional del pod

`$ kubeclt get pod nginx -o wide`

Para mostrar toda la información en `yaml` del pod:

`$ kubeclt get pod nginx -o yaml`

Borrar el pod:

`$ kubectl delete pod nginx`

## Desplegar varios pods mediante `Deployment`

En este caso se utiliza un `Deployment`, que es un template para crear/desplegar pods. La estructura es similar a un pod, cambia el `kind` y el `apiVersion`.

También se ve un `spec` dentro de otro `spec` que sirve como template para los pods.

`replicas`: Cantidad de pods en el deployment. Se le indica a Kubernetes cuántas réplicas de un pod se desean tener. Se definen cuántas réplicas (pods) de la aplicación se desean ejecutar en la definición del `deployment` y Kubernetes hará que esas réplicas de la aplicación se distribuyan entre los nodos. Si se establecen `5` réplicas en 3 nodos, algunos nodos tendrán más de una réplica de la aplicación en ejecución.

Archivo: `04-deployment.yaml`

Despliegue de los pods mediante el deployment:

`$ kubectl apply -f 04-deployment.yaml`

Se crean 2 pods con un has para diferenciarse. Durante el proceso de creación si se consulta el estado de los pods se puede ver:

- `0/1 READY` lo cual indica `readinessProbe` a Kubernetes que no se le debe mandar tráfico.

Una vez ambos pods estén en estado `RUNNING` se puede intentar eliminar uno de ellos para ver si Kubernetes levanta de nuevo al haberle indicado que siempre se desean 2 réplicas. 

Para hacer una consulta de los deployments:

`$ kubectl get deployments -o wide`

Se pueden eliminar los deployments al igual que los pods:

`$ kubectl delete deployment nginx-deployment`

## Desplegar varios pods mediante `Daemonset`

Es una forma de desplegar un pod al igual que se hace con los `Deployment`, pero los `Daemonset` intentan adherirse a un modelo de un pod por nodo, ya sea en todo el clúster o en un subconjunto de nodos. Normalmente utilizados en servicios de monitoreo y captura de logs.

En este caso se utiliza un `Daemonset`, que es un template para crear/desplegar pods. La estructura es similar nuevamente, cambia el `kind` y el `apiVersion`.

No se establecen `replicas` porque en este caso depende de la cantidad de nodos. Si se tienen dos nodos, se desplegará un pod en cada nodo.

Otra ventaja de usar un `Daemonset` es que, si se agrega un nodo al clúster, el `Daemonset` generará automáticamente un pod en ese nodo, cosa que no hace un `Deployment`.

En resumen, un `Daemonset` es un `Deployment` sin réplicas.

Es posible eliminar despliegues de la siguiente forma:

`$ kubectl delete -f 04-deployment.yaml`

Archivo: `03-daemonset.yaml`

Si se consultan los pods:

`$ kubectl get pods -o wide`

Se puede observar que hay 2 nginx porque existen 2 nodos, uno en cada nodo.

Igual que sucedía con los deployments, si se mata a un podo, este vuelve a desplegarse de forma automática.

`$ kubectl delete pod nginx-hash`

## Desplegar varios pods mediante `StatefulSet`

Similar al despliegue con `Deployment`, pero en este caso tiene un volumen asignado, al igual que sucede en Docker con los contenedores. Básicamente es un directorio o un disco vinculado a un pod y el pod siempre estará vinculado a ese disco. Los datos no se perderán. Muy útil para bases de datos desplegadas mediante pods donde los datos tienen que persistir.

Un pod de `mysql` debería desplegare utilizando un `StatefulSet` de forma que el disco siempre estará vinculado al pod y los datos no se perderán. En caso de no hacerlo así, sin volúmenes, cuando el pod se borre los archivos se van a borrar.

Esto se puede probar con un proveedor Cloud como puede ser DigitalOcean.

Para consultar el volumen (PVC - PersistentVolumeClaim ):

`$ kubectl get pvc`

Para consultar los detalles de un pvc determinado:

`$ kubectl describe pvc nombre_pvc`

Para consultar el `StatefulSet`: 

`$ kubectl get StatefulSet`

o 

`$ kubectl get sts`

Muy útil para manejar bases de datos, balanceadores de carga, s3 (Amazon)...

## Networking Kubernetes (Avanzado)

[Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

Comentarios:

- Cada pod tiene su propia IP, pero cada contenedor dentro de un pod compartirá la IP del pod. Si hay un podo con 3 contenedores, todos tendrán la misma IP que es la del pod. Es importante vigilar los puertos de los contenedores.

En Kubernetes existen distintos tipos de servicios, que son el mecanismo que permite comunicar pods. Existen tres servicios:

- `ClusterIP`: IP fija dentro del cluster, sirve como un pequeño load balancer entre todos los pods que se le asigna al servicio.
- `NodePort`: Crea un puerto en cada nodo que va a recibir el tráfico y lo va a mandar a los pods específicos. Por ejemplo, si quiero acceder al pod del servicio A y le pego al worker a un puerto en específico, ese tráfico va a encontrar el nodo que está corriendo ese pod y encontrará el tráfico
- `LoadBalancer`: Enfocado a la nube, crear balanceadores de carga en nuestro proveedor y redireccionará el tráfico entre los pods.

### Ejemplos tipos de servicios

Para esta parte se lanzará un pod con Ubuntu que se dejará en `sleep`.

`$ kubectl apply -f 06-randompod.yaml`

Para acceder al mismo, se puede ejecutar la siguiente línea:

`$ kubectl exec -ti ubuntu /bin/bash`

#### ClusterIP

Se realiza un `Deployment` de la `hello-app` de Google. Además, se añade una `label` con el valor `hello` y el nombre `role`. A continuación, se crea un servicio del tipo `clusterIP` con el nombre `hello` y el puerto `8080` apuntando al puerto `8080` del contenedor.

A este tipo de servicio se le debe indicar que utilice el `selector` con la etiqueta `role` y el valor `hello`, de forma que coincida con lo definido en el `Deployment`.

El servicio redirigirá al tráfico a todos aquellos pods que tengan la etiqueta `label` con el valor especificado en el servicio (`hello`).

Esto se realiza porque las IPs cuando el pod se elimina son modificadas, entonces se evita tener que estar manejando IPs. El servicio encuentra el pod detrás de la IP de forma automática.

Archivo: `07-hello-deployment-svc-clusterIP.yaml`

Para crear el deployment y el servicio:

`$ kubectl apply -f 07-hello-deployment-svc-clusterIP.yaml`

Para acceder a toda la información (servicios, pods, despliegues...):

`$ kubectl get all`

- Se puede observar que habrá 3 pods en ejecución del deployment `hello` de la aplicación de Google.
- También se observa el servicio `hello` creado que se identifica con una IP y el puerto `8080`.

Para ver las IPs de los contenedores de los pods a los que el servicio `hello` está asociado:

`$ kubectl describe svc hello`

- `svc` es la forma abreviada de poner `service`.

En la sección `Endpoints` se pueden ver las IPs de los contenedores de los pods. También se puede ver en la opción `Selector` el valor `role=hello` que es el que se había definido en el servicio.

Para comprobar que esta información coincide con los pods:

`$ kubectl get pods -o wide`

Y se puede comprobar que las IPs son las mismas que las listadas en la opción `Endpoint`.

Si se elimina uno de los pods, el `Deployment` levantará nuevamente el pod con una misma IP o diferente. Esto automáticamente será detectado por el servicio `hello` que modificará el `Endpoint` si es necesario para siempre apuntar al pod correspondiente. Esto quiere decir que el servicio balancea el pod asignándole un nuevo `endpoint` y redirigiéndome a él.

Se comprueba a partir de lo siguiente:

`$ kubectl exec -it ubuntu -- bash`

Y se hace `ping` desde este pod al servicio `hello`, tal que:

`$ ping hello`

También se puede hacer con `curl`:

`$ curl http://hello:8080`

Si se hace esto varias veces, se puede ver que cada vez responde un pod. Esto es debido al balanceador de carga, que lo que hace es elegir el pod que responde, es decir, se balancea la carga entre alguno de los tres.

####  NodePort

Archivo: `08-hello-deployment-svc-nodePort.yaml`

En este caso hay que especificar el tipo de servicio que es, porque en caso contrario por defecto se utiliza el servicio `clusterIP`.

Al utilizar `NodePort` se le puede agregar la opción `nodePort` extra en `ports` para indicar en qué puerto se quiere crear el servicio en cada nodo.

Se hacen los mismos pasos que en el punto anterior, es decir, `apply` y `get all` para obtener toda la información. Aquí se puede ver que se ha creado el servicio `hello ` de tipo `NodePort` y hace uso del puerto `30000`.

El servicio `NodePort` va a crear un puerto en cada nodo (aunque no esté ejecutándose el pod) y hará que el tráfico vaya por ese puerto por cualquiera de los pods.

Para acceder a la información de los tres nodos:

`$ kubectl get nodes -o wide`

A partir de las IPs de los nodos, se puede comprobar el funcionamiento del servicio desde nuestro host:

`$ curl http://ip_externa_de_un_nodo:30000`

Lo que hace a continuación es responder cualquiera de los pods en ese nodo. Se recomienda hacer la prueba con aquel nodo que tenga mayor número de pods para ver que cada vez que se haga la solicitud responderá un pod diferente, haciendo un balance entre pods.

#### Load Balancer

Archivo: `08-hello-deployment-svc-nodePort.yaml`

En este caso también hay que especificar el tipo de servicio que es, porque en caso contrario por defecto se utiliza el servicio `clusterIP`.

Al utilizar `LoadBalancer` no se le agregan más opciones.

Para acceder a la información de los tres nodos:

`$ kubectl get nodes -o wide`

Para acceder a la información del servicio:

`$ kubectl get svc`

En este punto se observa que la `EXTERNAL-IP` tiene el valor `pending`. Lo que hace Kubernetes está solicitando un balanceador de carga. De tal forma que los pods se asignen a esa IP y al hacer una solicitud sobre el balanceador de carga nos responderá con uno de los pods.

La diferencia con `NodePort` es que la IP está vinculada con cada nodo, por lo que hay que saber la IP de cada nodo. En cambio, en `LoadBalancer` la IP siempre va a ser la misma y siempre se podrá llegar al servicio.

A partir de las IP externa, se puede comprobar el funcionamiento del servicio desde nuestro host:

`$ curl http://ip_externa_svc:8080`

*Este servicio dependerá de un servicio en la Nube para obtener la IP externa.*

## Otras cosas de interés:

- [`ConfigMaps`](https://kubernetes.io/es/docs/concepts/configuration/configmap/): Un configmap es un objeto de la API utilizado para almacenar datos no confidenciales en el formato clave-valor. Los Pods pueden utilizar los ConfigMaps como variables de entorno, argumentos de la línea de comandos o como ficheros de configuración en un Volumen.

- [`Secrets`](https://kubernetes.io/es/docs/concepts/configuration/secret/): Los objetos de tipo Secret en Kubernetes te permiten almacenar y administrar información confidencial, como contraseñas, tokens OAuth y llaves ssh. Poniendo esta información en un Secret es más seguro y flexible que ponerlo en la definición de un Pod o en un container image.

