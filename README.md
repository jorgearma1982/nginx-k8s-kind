# Nginx en Kubernetes con kind

## Introducción

En este guía mostramos como instalar un cluster de `Kubernetes` en la laptop usando la implementación
de `kind`, la cual corre cada componente en un contenedor en lugar de usar máquinas virtuales, originalmente
fue diseñado para probar kubernetes en sí, pero también puede ser usado para desarrollo local ó CI.

Este proyecto puede servir para comprender los conceptos, la arquitectura y adentrarnos más en lo que
son los contenedores, los pods y su relación con los microservicios.

Instalaremos `Nginx` como Ingress Controller y una aplicación web simple para validar la funcionalidad de Nginx
como Ingress.

## Requisitos

Es necesario tener instalado y en ejecución docker para poder gestionar contenedores, este ejercicio lo
realizaremos en un equipo con MacOS, por lo que instalaremos la implementación `colima` para correr docker
en local, si tienes Linux puedes instalar docker usando tu manejador de paquetes favorito.

Iniciamos instalando colima y el cliente docker:

```shell
$ brew install colima docker
```

Ahora debemos iniciar colima:

```shell
$ colima start
INFO[0000] starting colima
INFO[0000] runtime: docker
INFO[0000] preparing network ...                         context=vm
INFO[0000] creating and starting ...                     context=vm
INFO[0023] provisioning ...                              context=docker
INFO[0023] starting ...                                  context=docker
INFO[0028] done
```

**NOTA:** Por default colima levanta una máquina virtual con `2` vCPUs y `2` GB de RAM, si se desea modificar
esto para asignar más CPU o RAM, puedes agregar los parámetros `--cpu 4` y `--memory 4`.

Ahora instalamos los paquetes para kubernetes con `kind`, también instalamos el cliente `kubectl` y
`k6` la herramienta de pruebas de carga de aplicaciones web:

```shell
$ brew install kind kubectl k6
```

Validamos la instalación de las herramientas, iniciamos con kind:

```shell
$ kind --version
kind version 0.14.0
```

Ahora veamos la versión de `kubectl`:

```shell
$ kubectl version --client=true
Client Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.3"}
Kustomize Version: v4.5.4
```

Y finalmente la versión de `k6`:

```shell
$ k6 version
k6 v0.39.0 ((devel)) 
```

## Instalación de cluster

Definimos la configuración del cluster con dos nodos, uno con rol de `control-plane` y otro de `worker`.

La configuración está almacenada en el archivo `kind/cluster-multi-ingress.yml`

```shell
$ cat kind/cluster-multi-ingress.yml
---
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
  - role: control-plane
  - role: worker
    extraPortMappings:
    - containerPort: 31682
      hostPort: 80
      listenAddress: "127.0.0.1"
      protocol: TCP
```

En la configuración de arriba podemos ver para el role `worker` se define el `extraPortMapping`, lo cual significa
que kind realizará una re dirección de puertos adicional, esta configuración básicamente hace un port forward del
puerto en el host hacia el puerto en un servicio dentro del cluster, los puertos que se redireccionan son:

* TCP `31682` al `80` para acceder a los servicios que expone Nginx en modo HTTP

Note también que los puertos que se re direccionan se asocian a la dirección local `127.0.0.1`.

Ahora creamos el cluster versión `1.23.4` con la configuración en el archivo `kind/cluster-multi-ingress.yml`:

```shell
$ kind create cluster --name nginxcluster --image kindest/node:v1.23.4 --config=kind/cluster-multi-ingress.yml
Creating cluster "nginxcluster" ...
 ✓ Ensuring node image (kindest/node:v1.23.4) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-nginxcluster"
You can now use your cluster with:

kubectl cluster-info --context kind-nginxcluster

Thanks for using kind! 😊
```

Listo!! Ya tenemos un cluster con un nodo de control plane y un worker, hagamos un listado de los clusters de kind:

```shell
$ kind get clusters
nginxcluster
```

La salida del comando de arriba muestra un cluster llamado `nginxcluster`.

Veamos que pasó a nivel contenedores docker:

```shell
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED              STATUS              PORTS                       NAMES
00de2ab27904   kindest/node:v1.23.4   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   127.0.0.1:80->31682/tcp     nginxcluster-worker
493b57478be4   kindest/node:v1.23.4   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   127.0.0.1:53428->6443/tcp   nginxcluster-control-plane
```

Arriba se puede ver hay dos contenedores en ejecución asociados a los nodos del cluster.

## Validación del cluster

Además de que el proceso de instalación fue super rápido, kind ya agregó un contexto a la configuración de
`kubectl` local:

```shell
$ kubectl config get-contexts
CURRENT   NAME                       CLUSTER             AUTHINFO                                                NAMESPACE
*         kind-nginxcluster          kind-nginxcluster   kind-nginxcluster
```

Ahora mostramos la información de dicho cluster:

```shell
$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:53551
CoreDNS is running at https://127.0.0.1:53551/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Como se pude ver, el cluster está corriendo en `localhost`.

Mostremos la salud del cluster:

```shell
$ kubectl get --raw '/healthz?verbose'
[+]ping ok
[+]log ok
[+]etcd ok
[+]poststarthook/start-kube-apiserver-admission-initializer ok
[+]poststarthook/generic-apiserver-start-informers ok
[+]poststarthook/priority-and-fairness-config-consumer ok
[+]poststarthook/priority-and-fairness-filter ok
[+]poststarthook/start-apiextensions-informers ok
[+]poststarthook/start-apiextensions-controllers ok
[+]poststarthook/crd-informer-synced ok
[+]poststarthook/bootstrap-controller ok
[+]poststarthook/rbac/bootstrap-roles ok
[+]poststarthook/scheduling/bootstrap-system-priority-classes ok
[+]poststarthook/priority-and-fairness-config-producer ok
[+]poststarthook/start-cluster-authentication-info-controller ok
[+]poststarthook/aggregator-reload-proxy-client-cert ok
[+]poststarthook/start-kube-aggregator-informers ok
[+]poststarthook/apiservice-registration-controller ok
[+]poststarthook/apiservice-status-available-controller ok
[+]poststarthook/kube-apiserver-autoregistration ok
[+]autoregister-completion ok
[+]poststarthook/apiservice-openapi-controller ok
healthz check passed
```

Listamos los nodos del cluster:

```shell
$ kubectl get nodes
NAME                         STATUS   ROLES                  AGE     VERSION
nginxcluster-control-plane   Ready    control-plane,master   2m10s   v1.23.4
nginxcluster-worker          Ready    <none>                 94s     v1.23.4
```

Como se puede ver tenemos un nodo que es el maestro, es decir, la capa de control, y tenemos otro que es el worker.

Listemos los pods de los servicios que están en ejecución:

```shell
$ kubectl get pods -A
NAMESPACE            NAME                                                 READY   STATUS    RESTARTS   AGE
kube-system          coredns-64897985d-8fhfg                              1/1     Running   0          2m7s
kube-system          coredns-64897985d-qkvtl                              1/1     Running   0          2m7s
kube-system          etcd-nginxcluster-control-plane                      1/1     Running   0          2m22s
kube-system          kindnet-5w8lw                                        1/1     Running   0          109s
kube-system          kindnet-qtwgs                                        1/1     Running   0          2m7s
kube-system          kube-apiserver-nginxcluster-control-plane            1/1     Running   0          2m24s
kube-system          kube-controller-manager-nginxcluster-control-plane   1/1     Running   0          2m23s
kube-system          kube-proxy-dbxmh                                     1/1     Running   0          109s
kube-system          kube-proxy-p2ndh                                     1/1     Running   0          2m7s
kube-system          kube-scheduler-nginxcluster-control-plane            1/1     Running   0          2m24s
local-path-storage   local-path-provisioner-5ddd94ff66-lvl2q              1/1     Running   0          2m7s
```

Esto se ve bien, todos los pods están `Running` :), en su mayoría son los servicios del cluster:

* kube-apiserver
* kube-scheduler
* kube-proxy
* kube-controller-manager
* etcd
* kindnet
* coredns
* local-path-provisioner

Todo indica a que el cluster tiene todo listo para desplegar nuestras aplicaciones.

## Despliegue de Nginx

Instalaremos Nginx Ingress Controller usando la implementación open source.

Usando helm, agregamos el repositorio de nginx:

```shell
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

Ahora actualizamos los repositorios:

```shell
$ helm repo update
```

Creamos un namespace para Nginx:

```shell
$ kubectl create namespace ingress
namespace/ingress created
```

Listamos los namespaces:

```shell
$ kubectl get namespace ingress
NAME      STATUS   AGE
ingress   Active   1s
```

Ejecutamos la instalación con los parámetros personalizados para habilitar el servicio de admin y postgresql:

```shell
$ helm install nginx ingress-nginx/ingress-nginx \
  --namespace ingress \
  --set controller.ingressClass=nginx \
  --set controller.hostPort.enabled=true \
  --set controller.publishService.enabled=false \
  --set controller.extraArgs.publish-status-address="localhost" \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=31682
NAME: nginx
LAST DEPLOYED: Sun Aug 28 12:24:58 2022
NAMESPACE: ingress
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
Get the application URL by running these commands:
  export POD_NAME=$(kubectl --namespace ingress get pods -o jsonpath="{.items[0].metadata.name}" -l "app=ingress-nginx,component=controller,release=nginx")
  kubectl --namespace ingress port-forward $POD_NAME 8080:80
  echo "Visit http://127.0.0.1:8080 to access your application."

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

Cuando la instalación haya terminado nos da algunos ejemplos de como implementar un las reglas ingress
con el nuevo controlador nginx.

Para verificar que los componentes de nginx esten listos, hagamos un listado de los recursos en el namespace de
ingress:

```shell
$ kubectl -n ingress get all
NAME                                                  READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-nginx-controller-68487bdd78-pkgjz   1/1     Running   0          4h13m

NAME                                               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/nginx-ingress-nginx-controller             NodePort    10.96.65.118   <none>        80:31682/TCP,443:32337/TCP   4h13m
service/nginx-ingress-nginx-controller-admission   ClusterIP   10.96.195.45   <none>        443/TCP                      4h13m

NAME                                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-nginx-controller   1/1     1            1           4h13m

NAME                                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-ingress-nginx-controller-68487bdd78   1         1         1       4h13m
```

En el listado vemos que hay un deployment llamado `nginx-ingress-nginx-controller`, el cual tiene los siguientes pods:

* nginx-ingress-nginx-controller 

En el listado de servicios, el servicio `nginx-ingress-nginx-controller` es de tipo `NodePort` en el puerto `80` y `443`.

Hagamos una petición a nginx al puerto TCP/80 donde se exponen los servicios:

```shell
$ curl http://localhost/
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html
```

Listo!!! Ya tenemos Nginx respondiendo en el localhost peticiones HTTP en el puerto TCP/80.

## Despliegue aplicación

Realizamos el despliegue de una aplicación:

```shell
$ kubectl apply -f whoami/1_deployment.yml
deployment.apps/whoami created
```

Creamos el service de una aplicación:

```shell
$ kubectl apply -f whoami/2_service.yml
service/whoami created
```

Creamos el ingress de una aplicación:

```shell
$ kubectl apply -f whoami/3_ingress.yml
ingress.networking.k8s.io/whoami created
```

Esperamos unos segundos a que levanten los servicios y continuamos con la validaciones.

## Validación aplicación

Ahora validamos listando todos los recursos del namespace `default`:

```shell
$ kubectl -n default get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/whoami-6977d564f9-nrrg2   1/1     Running   0          92s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    16h
service/whoami       ClusterIP   10.96.144.246   <none>        8080/TCP   87s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/whoami   1/1     1            1           92s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/whoami-6977d564f9   1         1         1       92s
```

Como se puede ver se tiene los recursos `deployment`, el `replicaset`, los `pods` y el `service`.

Ahora validamos que responda el servicio whoami a través de nginx:

```shell
$ curl http://localhost/whoami
Hostname: whoami-6977d564f9-dq5pr
IP: 127.0.0.1
IP: ::1
IP: 10.244.1.3
IP: fe80::dc3f:a9ff:fe84:9283
RemoteAddr: 10.244.1.2:58086
GET / HTTP/1.1
Host: localhost
User-Agent: curl/7.64.1
Accept: */*
Connection: keep-alive
X-Forwarded-For: 10.244.1.1
X-Forwarded-Host: localhost
X-Forwarded-Path: /whoami
X-Forwarded-Port: 80
X-Forwarded-Prefix: /whoami
X-Forwarded-Proto: http
X-Real-Ip: 10.244.1.1
```

Listo!, ya tenemos una respuesta de `whoami`.

## Pruebas de carga a la aplicación web

Usaremos `k6` para realizar pruebas de carga en la aplicación que exponemos a través de nginx:

Ahora ejecutamos el script con las pruebas:

```shell
$ k6 run k6/script.js

          /\      |‾‾| /‾‾/   /‾‾/
     /\  /  \     |  |/  /   /  /
    /  \/    \    |     (   /   ‾‾\
   /          \   |  |\  \ |  (‾)  |
  / __________ \  |__| \__\ \_____/ .io

  execution: local
     script: k6/script.js
     output: -

  scenarios: (100.00%) 1 scenario, 50 max VUs, 2m10s max duration (incl. graceful stop):
           * default: Up to 50 looping VUs for 1m40s over 3 stages (gracefulRampDown: 30s, gracefulStop: 30s)


running (1m40.6s), 00/50 VUs, 2398 complete and 0 interrupted iterations
default ✓ [======================================] 00/50 VUs  1m40s

     ✓ status code is 200

   ✓ checks.........................: 100.00% ✓ 2398      ✗ 0
     data_received..................: 1.4 MB  14 kB/s
     data_sent......................: 271 kB  2.7 kB/s
     http_req_blocked...............: avg=7.79µs  min=0s    med=2µs    max=591µs   p(90)=3µs    p(95)=5µs
     http_req_connecting............: avg=4.93µs  min=0s    med=0s     max=368µs   p(90)=0s     p(95)=0s
   ✓ http_req_duration..............: avg=1.31ms  min=693µs med=1.2ms  max=54.46ms p(90)=1.68ms p(95)=1.87ms
       { expected_response:true }...: avg=1.31ms  min=693µs med=1.2ms  max=54.46ms p(90)=1.68ms p(95)=1.87ms
     http_req_failed................: 0.00%   ✓ 0         ✗ 2398
   ✓ http_req_rate..................: 50.00%  ✓ 4796      ✗ 4796
     http_req_receiving.............: avg=17.48µs min=4µs   med=15µs   max=164µs   p(90)=28µs   p(95)=34.14µs
     http_req_sending...............: avg=8.48µs  min=2µs   med=7µs    max=68µs    p(90)=14µs   p(95)=19µs
     http_req_tls_handshaking.......: avg=0s      min=0s    med=0s     max=0s      p(90)=0s     p(95)=0s
     http_req_waiting...............: avg=1.29ms  min=674µs med=1.17ms max=54.39ms p(90)=1.64ms p(95)=1.83ms
     http_reqs......................: 2398    23.838753/s
     iteration_duration.............: avg=1s      min=1s    med=1s     max=1.05s   p(90)=1s     p(95)=1s
     iterations.....................: 2398    23.838753/s
     vus............................: 2       min=1       max=50
     vus_max........................: 50      min=50      max=50
```

## Limpieza

Para destruir el cluster ejecutamos:

```shell
$ kind delete nginxcluster
Deleting cluster "nginxcluster" ...
```

También podemos limpiar colima:

```shell
$ colima delete
are you sure you want to delete colima and all settings? [y/N] y
INFO[0001] deleting colima
INFO[0001] deleting ...                                  context=docker
INFO[0001] done
```

Y listo todo se ha terminado.

## Problemas conocidos

Si usas una mac m1 es probable que tengas errores al descargar las imágenes de los contenedores, si es un error
relacionado a resolución de nombres DNS, puedes probar agregando la configuración de `lima` para que no use
el dns del host y en su lugar use el de google, por ejemplo:

Creamos configuración para dns de lima:

```shell
$ vim ~/.lima/_config/override.yaml
```

Con el siguiente contenido:

```shell
useHostResolver: false
dns:
  - 8.8.8.8
```

Se recomienda que hagas un reset de colima, haciendo delete, y nuevamente start.

También puedes iniciar colima con la opción `--dns`, por ejemplo:

```shell
$ colima start --dns 8.8.8.8
```

## Comandos útiles

Listado versiones:

* kubectl version

Listado contextos:

* kubectl config get-contexts

Detalles de cluster:

* kubectl cluster-info

Gestión de nodos:

* kubectl get nodes
* kubectl describe node NODENAME

Gestión de pods:

* kubectl get pods
* kubectl describe pod PODNAME
* kubectl logs PODNAME
* kubectl delete pod PODNAME

Gestión de services:

* kubectl get services
* kubectl describe service SVCNAME
* kubectl delete service SVCNAME

Gestión de namespaces:

* kubectl get namespaces
* kubectl describe namespace NSNAME
* kubectl delete namespace NSNAME

Gestión de recursos en modo declarativo:

* kubectl apply -f YAMLFILE
* kubectl delete -f YAMLFILE

Gestión de deployments:

* kubectl get deployment
* kubectl describe deployment PODNAME
* kubectl delete deployment PODNAME

Gestión charts:

* helm ls
* helm install CHARTNAME
* helm uninstall CHARTNAME

## Referencias

La siguiente es una lista de referencias externas que pueden serle de utilidad:

* [Colima - container runtimes on macOS (and Linux) with minimal setup](https://github.com/abiosoft/colima)
* [Kubernetes - Orquestación de contenedores para producción](https://kubernetes.io/es/)
* [kind - home](https://kind.sigs.k8s.io/)
* [kind - quick start](https://kind.sigs.k8s.io/docs/user/quick-start/)
* [Kubernetes - Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
* [Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx)
* [Nginx - Installation Guide](https://kubernetes.github.io/ingress-nginx/deploy/)