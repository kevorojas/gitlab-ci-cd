## Instalar Minikube

En este caso vamos a instalarlo descargando un ejecutable auto contenido:

```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
```

Luego de esto, para tener el comando **minikube** disponible debemos moverlo a la siguiente ruta `/usr/local/bin`:

```
sudo cp minikube /usr/local/bin && rm minikube
```

## Iniciar Minikube

En este caso vamos a usar **Docker** como Hipervisor

```
minikube start --driver=docker
```

Una vez hayamos iniciado minikube podemos ejecutar el siguiente comando para instalar kubectl 

```
minikube kubectl get all
```

## Instalar Gitlab

Para instalar gitlab lo haremos con docker igualmente para no estar creando VM. 

Entonces para instalar nos vamos a [Docker hub](https://hub.docker.com/_/gitlab-community-edition/plans/6a33c5d4-c1cc-48f4-ae30-e033126ffd7f?tab=instructions) para descargar la imagen mas reciente o la estable, en este caso usaremos la version: gitlab/gitlab-ee:latest

```
docker pull gitlab/gitlab-ee:latest
```

Antes de ejecutar la imagen debemos crear esta variable de entorno para los volumenes lo que permitira que la data sea persistente

```
export GITLAB_HOME=/srv/gitlab
```

y por ultimo ejecutamos el contenedor

```
sudo docker run --detach \
  --hostname gitlab.example.com \
  --network minikube \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume $GITLAB_HOME/config:/etc/gitlab \
  --volume $GITLAB_HOME/logs:/var/log/gitlab \
  --volume $GITLAB_HOME/data:/var/opt/gitlab \
  gitlab/gitlab-ee:latest
```

La instalacion de gitlab puede tardar unos minutos ya que esta image lo que hace es ejectuar o correr una receta de chef donde provisiona todo lo necesario, para ver lo que esta sucediendo podemos ejecutar este comando:

```
docker logs -f gitlab
```

## Agregar cluster de minikube a Gitlab

Antes que nada debemos habilitar los requests to the local network in gitlab en **settings > network > outbound requests**

Tambien debemos modificar el campo **Custom Git clone URL for HTTP(S)**

debemos colocar la URL pero con la **ip de gitlab**

Para agregar nuestro cluster necesitamos tres cosas:

- API URL

- CA Certificate

- Service Token

Para conseguir nuestro **API URL** debemos ejecutar este comando:

```
minikube kubectl cluster-info | grep 'Kubernetes master' | awk '/http/ {print $NF}'
```

Para conseguir el **CA certificate** ejecutamos lo siguiente:

Primero necesitamos saber el nombre del secret que contiene el CA para eso ejecutamos el siguiente comando:

```
minikube kubectl get secrets
```

Luego ejecutamos este comando reemplanzado el <secret name> con el secret name que te aparecio:

```
minikube kubectl -- get secret <secret name> -o jsonpath="{['data']['ca\.crt']}" | base64 --decode
```

Copiamos todo el certificado y lo pegamos a Gitlab.

Por utlimo necesitamos el **Service Token**

GitLab se autentica contra Kubernetes mediante tokens de servicio, que tienen como ámbito un espacio de nombres en particular

Para crear este token o servicio necesitos crear un archivo llamado **00-gitlab-admin-service-account.yaml** con el siguiente contenido:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: gitlab
    namespace: kube-system
```

Para crear el servicio ejecutamos el siguiente comando:

```
minikube kubectl -- apply -f 00-gitlab-admin-service-account.yaml
```

Ahora necesitamos obtener el token para gitlab, para ello ejecutamos lo siguiente:

```
minikube kubectl -- -n kube-system describe secret $(minikube kubectl -- -n kube-system get secret | grep gitlab | awk '{print $1}')
```

Copiamos el **token** y lo pegamos en gitlab

Una vez hayamos hecho todo esto ya nuestro cluster en teoria ya estaria agregado a gitlab, para confirmar podemos ver los **namespaces** y deberia aparecernos el que gitlab creo

Para eso ejecutamos:

```
minikube kubectl get ns
```

Para hacer que nuestro runner pueda conectarse correctamente a gitlab debemos modificar la siguiente variable de entorno ubicada en la seccion de container:

```
- name: CI_SERVER_URL
  value: 'http://direccion IP de Gitlab/'
```

Luego debemos crear dos archivos uno llamado **01-clusterrole.yaml**

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1 
metadata:
  name: deployment-manager
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # You can also use ["*"]
```

y lo ejecutamos con el siguiente comando

```
minikube kubectl -- apply -f 01-clusterrole.yaml
```

El otro sera llamado **02-clusterrolebinding.yaml**

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: runner-gitlab-runner-deployments
subjects:
- kind: ServiceAccount
  namespace: "gitlab-managed-apps"
  name: "default"
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: "deployment-manager"
  apiGroup: ""
```

Lo aplicamos con el siguiente comando:

```
minikube kubectl -- apply -f 02-clusterrolebinding.yaml
```

Una vez hemos modificado eso ya podriamos subir nuestro repositorio

Debemos agregar las variables de entorno para poder conectarnos a nuestro docker hub y poder crear la imagen

luego de eso veremos que nuestro CI/CD funciona correctamente

Por ultimo debemos crear un Servicio para acceder a nuestra aplicacion de kubernetes

en este caso lo hare manualmente pero se podria investigar la forma de que se ejecute desde el CI/CD

Entonces para crear el servicio yo cree este archivo llamado **03-svcnodeport.yaml**

```
apiVersion: v1
kind: Service
metadata:
  name: hola-mundo
  namespace: gitlab-managed-apps
spec:
  type: NodePort
  selector:
    app: hola-mundo
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007
```

Para aplicarlo ejecutamos el siguiente comando

```
minikube kubectl -- apply -f 03-svcnodeport.yaml
```

## LINK DE VIDEO EN YOUTUBE

https://youtu.be/UjF_9gIjyKE
