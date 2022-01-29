# kubernetes-templating

Через GUI развернул кластер в облаке GCP.
На локальной машине установил gcloud, настроил локальный kubectl на облако, установил helm:
```
$ gcloud version
Google Cloud SDK 370.0.0
bq 2.0.73
core 2022.01.21
gsutil 5.6
$
$ gcloud container clusters list
NAME            LOCATION           MASTER_VERSION   MASTER_IP       MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
cluster-202112  europe-central2-b  1.21.5-gke.1302  34.116.131.193  e2-medium     1.21.5-gke.1302  3          RUNNING
$
$ kubectl get pod -A
NAMESPACE     NAME                                                       READY   STATUS    RESTARTS   AGE
kube-system   event-exporter-gke-5479fd58c8-f4bn5                        2/2     Running   0          73m
kube-system   fluentbit-gke-9mgjt                                        2/2     Running   0          72m
kube-system   fluentbit-gke-mslkw                                        2/2     Running   0          72m
kube-system   fluentbit-gke-nttvw                                        2/2     Running   0          72m
kube-system   gke-metrics-agent-2n2sg                                    1/1     Running   0          72m
kube-system   gke-metrics-agent-btq99                                    1/1     Running   0          72m
kube-system   gke-metrics-agent-d4lrn                                    1/1     Running   0          72m
kube-system   konnectivity-agent-7db65b5645-d64rg                        1/1     Running   0          72m
kube-system   konnectivity-agent-7db65b5645-lbsnr                        1/1     Running   0          73m
kube-system   konnectivity-agent-7db65b5645-rl495                        1/1     Running   0          72m
kube-system   konnectivity-agent-autoscaler-5c49cb58bb-vvnmn             1/1     Running   0          73m
kube-system   kube-dns-697dc8fc8b-67jjs                                  4/4     Running   0          73m
kube-system   kube-dns-697dc8fc8b-nxwpp                                  4/4     Running   0          72m
kube-system   kube-dns-autoscaler-844c9d9448-6j2gq                       1/1     Running   0          73m
kube-system   kube-proxy-gke-cluster-202112-default-pool-af9068a0-3jft   1/1     Running   0          71m
kube-system   kube-proxy-gke-cluster-202112-default-pool-af9068a0-mmp5   1/1     Running   0          71m
kube-system   kube-proxy-gke-cluster-202112-default-pool-af9068a0-xqh4   1/1     Running   0          72m
kube-system   l7-default-backend-865b4c8f8b-6m9tm                        1/1     Running   0          73m
kube-system   metrics-server-v0.4.4-857776bc9c-xhhmq                     2/2     Running   0          72m
kube-system   pdcsi-node-fkmzk                                           2/2     Running   0          72m
kube-system   pdcsi-node-nzh4h                                           2/2     Running   0          72m
kube-system   pdcsi-node-ptg26                                           2/2     Running   0          72m
$
$ helm version
version.BuildInfo{Version:"v3.8.0", GitCommit:"d14138609b01886f544b2025f5000351c9eb092e", GitTreeState:"clean", GoVersion:"go1.17.5"}
```
Добавил репозиторий stable:
```
$ helm repo list
Error: no repositories to show
$
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com
Error: repo "https://kubernetes-charts.storage.googleapis.com" is no longer available; try "https://charts.helm.sh/stable" instead
$
$ helm repo add stable https://charts.helm.sh/stable
"stable" has been added to your repositories
$
$ helm repo list
NAME    URL
stable  https://charts.helm.sh/stable
```
После чего установил устаревшую версию nginx-ingress:
```
$ helm upgrade --install nginx-ingress stable/nginx-ingress --wait --namespace=nginx-ingress --version=1.41.3
Release "nginx-ingress" does not exist. Installing it now.
WARNING: This chart is deprecated
NAME: nginx-ingress
LAST DEPLOYED: Thu Jan 27 17:00:38 2022
NAMESPACE: nginx-ingress
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*******************************************************************************************************
* DEPRECATED, please use https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx *
*******************************************************************************************************


The nginx-ingress controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace nginx-ingress get services -o wide -w nginx-ingress-controller'

An example Ingress that makes use of the controller:

  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                servicePort: 80
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

Проверим LB
```
$ kubectl --namespace nginx-ingress get services
NAME                            TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                      AGE
nginx-ingress-controller        LoadBalancer   10.84.3.225   34.116.149.237   80:32585/TCP,443:30535/TCP   3m43s
nginx-ingress-default-backend   ClusterIP      10.84.2.240   <none>           80/TCP                       3m43s
```

Добавил репозиторий, в котором хранится актуальный helm chart cert-manager:
```
$ helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories
```
Перед установкой cert-manager, создадим в кластере некоторые CRD
```
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.crds.yaml
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 Cust    omResourceDefinition
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
```
Выругался, но вроде все стоят:
```
$ kubectl get CustomResourceDefinition -A
NAME                                             CREATED AT
backendconfigs.cloud.google.com                  2022-01-27T09:36:59Z
capacityrequests.internal.autoscaling.gke.io     2022-01-27T09:36:35Z
certificaterequests.cert-manager.io              2022-01-27T11:14:05Z
certificates.cert-manager.io                     2022-01-27T11:14:05Z
challenges.acme.cert-manager.io                  2022-01-27T11:14:06Z
clusterissuers.cert-manager.io                   2022-01-27T11:14:06Z
frontendconfigs.networking.gke.io                2022-01-27T09:37:00Z
issuers.cert-manager.io                          2022-01-27T11:14:08Z
managedcertificates.networking.gke.io            2022-01-27T09:36:38Z
orders.acme.cert-manager.io                      2022-01-27T11:14:08Z
serviceattachments.networking.gke.io             2022-01-27T09:37:02Z
servicenetworkendpointgroups.networking.gke.io   2022-01-27T09:37:01Z
storagestates.migration.k8s.io                   2022-01-27T09:36:41Z
storageversionmigrations.migration.k8s.io        2022-01-27T09:36:41Z
updateinfos.nodemanagement.gke.io                2022-01-27T09:36:42Z
volumesnapshotclasses.snapshot.storage.k8s.io    2022-01-27T09:36:39Z
volumesnapshotcontents.snapshot.storage.k8s.io   2022-01-27T09:36:40Z
volumesnapshots.snapshot.storage.k8s.io          2022-01-27T09:36:40Z
```
Установил cert-manager:
```
Error: create: failed to create: namespaces "cert-manager" not found
```
woops...
```
$ kubectl create namespace cert-manager
namespace/cert-manager created
```
С большим количество ругани на deprecated установился:
```
NAME: cert-manager
LAST DEPLOYED: Thu Jan 27 17:29:10 2022
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/
```
Проверим, что там
```
$ kubectl get pods --namespace cert-manager
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-cainjector-fc6c787db-dmz8h   1/1     Running   0          2m1s
cert-manager-d994d94d7-sl72w              1/1     Running   0          2m1s
cert-manager-webhook-5c5ddb797c-5v8kt     1/1     Running   0          2m1s
```
**Самостоятельное задание (определите, что еще требуется установить для корректной работы)**
Так оно же после установки написало, что ещё требуется:
> In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

[Срисовал](https://cert-manager.io/docs/configuration/acme/) манифест -  kubernetes-templating/cert-manager/clusterissuer.yaml

**chartmuseum**

отредактировал манифест values.yaml, применил
```
$ helm upgrade --install chartmuseum stable/chartmuseum --wait --namespace=chartmuseum --version=2.13.2 -f ./values.yaml
Release "chartmuseum" does not exist. Installing it now.
W0127 19:56:37.073985 2406959 warnings.go:70] networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
W0127 19:56:39.046741 2406959 warnings.go:70] networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAME: chartmuseum
LAST DEPLOYED: Thu Jan 27 19:56:36 2022
NAMESPACE: chartmuseum
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

Get the ChartMuseum URL by running:

  export POD_NAME=$(kubectl get pods --namespace chartmuseum -l "app=chartmuseum" -l "release=chartmuseum" -o jsonpath="{.items[0].metadata.name}")
  echo http://127.0.0.1:8080/
  kubectl port-forward $POD_NAME 8080:8080 --namespace chartmuseum
```
проверил
```
$ helm ls -n chartmuseum
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART               APP VERSION
chartmuseum     chartmuseum     1               2022-01-27 19:56:36.376197696 +0600 +06 deployed        chartmuseum-2.13.2  0.12.0
```
проверяем доступность ресурса и валидность сертификата...
...ну ресурс доступен, а вот сертификат
> Организация -- Acme Co
Общее имя -- Kubernetes Ingress Controller Fake Certificate

походу надо было production всё таки использовать, а не staged
переписал манифесты, перезапустил... нифига, опять кривой...
ещё раз всё проверил, с третьей попытки завелось, сертификат валидный
[https://chartmuseum.34.116.149.237.nip.io/](https://chartmuseum.34.116.149.237.nip.io/)

**harbor**

Add Helm repository
```
$ helm repo add harbor https://helm.goharbor.io
"harbor" has been added to your repositories
```
скопировал из chartmuseum манифест, начал редактировать... и зачем только надо было эту портянку копировать вообще ((


как то странно, установка вываливается в ошибку (да опять наверное конфликты со старыми версиями):
```
$ helm upgrade --install harbor harbor/harbor --wait --namespace=harbor --version=1.1.2 -f ./harbor/values.yaml
Release "harbor" does not exist. Installing it now.
W0127 21:23:45.868573 2429075 warnings.go:70] extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
W0127 21:23:52.966022 2429075 warnings.go:70] extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
W0127 21:23:53.097110 2429075 warnings.go:70] extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
Error: timed out waiting for the condition
```
но запускается и сертификат в порядке: [https://harbor.34.116.149.237.nip.io](https://harbor.34.116.149.237.nip.io/harbor/sign-in?redirect_url=%2Fharbor%2Fprojects)

## Создаем свой helm chart

по инструкции, запускаю, готово
```
$ helm upgrade --install hipster-shop kubernetes-templating/hipster-shop --namespace hipster-shop
Release "hipster-shop" does not exist. Installing it now.
NAME: hipster-shop
LAST DEPLOYED: Thu Jan 27 22:20:50 2022
NAMESPACE: hipster-shop
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
**frontend**
```
$ helm create ./kubernetes-templating/frontend
Creating ./kubernetes-templating/frontend
```
перезапускаем hipster-shop без frontend-а
```
$ helm upgrade --install hipster-shop ./hipster-shop --namespace hipster-shop
Release "hipster-shop" has been upgraded. Happy Helming!
NAME: hipster-shop
LAST DEPLOYED: Thu Jan 27 22:36:58 2022
NAMESPACE: hipster-shop
STATUS: deployed
REVISION: 2
TEST SUITE: None
```
а теперь отдельно frontend
```
$ helm upgrade --install frontend ./frontend --namespace hipster-shop
Release "frontend" does not exist. Installing it now.
NAME: frontend
LAST DEPLOYED: Thu Jan 27 22:41:01 2022
NAMESPACE: hipster-shop
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
**создаём свой helm chart**
создал values.yaml, добавил .image.tag
изменил templates/deployment.yaml, перезапустил сервис, ничего не произошло:
```
$ helm upgrade --install hipster-shop ../hipster-shop --namespace hipster-shop
Release "hipster-shop" has been upgraded. Happy Helming!
NAME: hipster-shop
LAST DEPLOYED: Fri Jan 28 12:01:17 2022
NAMESPACE: hipster-shop
STATUS: deployed
REVISION: 7
TEST SUITE: None
```
создал values.yaml с параметрами, изменил templates/deployment.yaml & templates/service.yaml
удалил release frontend из кластера
добавил зависимость в манифест ../hipster-shop/Chart.yaml про frontend
обновил
```
$ helm dep update ../hipster-shop
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "harbor" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Deleting outdated charts
```
в папке charts, появился архив
```
$ ll ../../kubernetes-templating/hipster-shop/charts/
total 4
-rw-rw-r-- 1 gdos gdos 1482 Jan 28 12:58 frontend-0.1.0.tgz
```
обновил
```
$ helm upgrade --install hipster-shop ../hipster-shop --namespace hipster-shop
W0128 13:04:39.026544 2665894 warnings.go:70] networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
W0128 13:04:47.687800 2665894 warnings.go:70] networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
W0128 13:04:47.818296 2665894 warnings.go:70] networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
Release "hipster-shop" has been upgraded. Happy Helming!
NAME: hipster-shop
LAST DEPLOYED: Fri Jan 28 13:04:38 2022
NAMESPACE: hipster-shop
STATUS: deployed
REVISION: 8
TEST SUITE: None
```
[проверил](https://shop.34.116.149.237.nip.io/) - работает
> из CI-системы мы можем менять параметры helm chart, описанные в values.yaml с помощью специального ключа --set
напрмимер helm upgrade --install hipster-shop kubernetes-templating/hipster-shop --namespace hipster-shop --set frontend.service.NodePort=31234

**Kubecfg**

выпилил из all-hipster-shop.yaml манифесты описывающие service и deployment для сервисов paymentservice и shippingservice
```
$ ll ../kubecfg/
total 16
-rw-rw-r-- 1 gdos gdos 811 Jan 28 13:53 paymentservice-deployment.yaml
-rw-rw-r-- 1 gdos gdos 188 Jan 28 13:54 paymentservice-service.yaml
-rw-rw-r-- 1 gdos gdos 957 Jan 28 13:56 shippingservice-deployment.yaml
-rw-rw-r-- 1 gdos gdos 190 Jan 28 13:56 shippingservice-service.yaml
```
обновил release hipster-shop, сервисы пропали
```
$ kubectl get services -n hipster-shop
NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
adservice               ClusterIP   10.84.3.155    <none>        9555/TCP       15h
cartservice             ClusterIP   10.84.7.212    <none>        7070/TCP       15h
checkoutservice         ClusterIP   10.84.2.180    <none>        5050/TCP       15h
currencyservice         ClusterIP   10.84.15.209   <none>        7000/TCP       15h
emailservice            ClusterIP   10.84.14.193   <none>        5000/TCP       15h
frontend                NodePort    10.84.12.66    <none>        80:31925/TCP   56m
productcatalogservice   ClusterIP   10.84.4.148    <none>        3550/TCP       15h
recommendationservice   ClusterIP   10.84.0.16     <none>        8080/TCP       15h
redis-cart              ClusterIP   10.84.6.249    <none>        6379/TCP       15h
```
и в магазине, при попытке добавить товар в корзину, получаем ошибку:
```
Uh, oh!
Something has failed. Below are some details for debugging.

HTTP Status: 500 Internal Server Error

rpc error: code = Unavailable desc = all SubConns are in TransientFailure, latest connection error: connection error: desc = "transport: Error while dialing dial tcp: lookup shippingservice on 10.84.0.10:53: no such host"
failed to get shipping quote
main.(*frontendServer).viewCartHandler
	/go/src/github.com/GoogleCloudPlatform/microservices-demo/src/frontend/handlers.go:205
net/http.HandlerFunc.ServeHTTP
	/usr/local/go/src/net/http/server.go:1995
github.com/GoogleCloudPlatform/microservices-demo/src/frontend/vendor/github.com/gorilla/mux.(*Router).ServeHTTP
	/go/src/github.com/GoogleCloudPlatform/microservices-demo/src/frontend/vendor/github.com/gorilla/mux/mux.go:212
main.(*logHandler).ServeHTTP
	/go/src/github.com/GoogleCloudPlatform/microservices-demo/src/frontend/middleware.go:81
main.ensureSessionID.func1
	/go/src/github.com/GoogleCloudPlatform/microservices-demo/src/frontend/middleware.go:103
net/http.HandlerFunc.ServeHTTP
	/usr/local/go/src/net/http/server.go:1995
github.com/GoogleCloudPlatform/microservices-demo/src/frontend/vendor/go.opencensus.io/plugin/ochttp.(*Handler).ServeHTTP
	/go/src/github.com/GoogleCloudPlatform/microservices-demo/src/frontend/vendor/go.opencensus.io/plugin/ochttp/server.go:82
net/http.serverHandler.ServeHTTP
	/usr/local/go/src/net/http/server.go:2774
net/http.(*conn).serve
	/usr/local/go/src/net/http/server.go:1878
runtime.goexit
	/usr/local/go/src/runtime/asm_amd64.s:1337
```
установил kubecfg
```
$ kubecfg version
kubecfg version: v0.24.1
jsonnet version: v0.18.0
client-go version: v0.0.0-master+$Format:%H$
```
стянул kube.libsonnet
```
kubecfg]$ wget https://raw.githubusercontent.com/bitnami-labs/kube-libsonnet/master/kube.libsonnet
```
нарисовал services.jsonnet (воспользовался подсказкой)
проверил генерацию манифестов, вроде норм, попробую установить:
```
$ kubecfg update services.jsonnet --namespace hipster-shop
INFO  Validating deployments paymentservice
INFO  validate object "apps/v1, Kind=Deployment"
INFO  Validating services paymentservice
INFO  validate object "/v1, Kind=Service"
INFO  Validating deployments shippingservice
INFO  validate object "apps/v1, Kind=Deployment"
INFO  Validating services shippingservice
INFO  validate object "/v1, Kind=Service"
INFO  Fetching schemas for 4 resources
INFO  Creating services paymentservice
INFO  Creating services paymentservice
INFO  Creating deployments paymentservice
INFO  Creating deployments shippingservice
```
paymentservice & paymentservice - появились
```
$ kubectl get services -n hipster-shop
NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
adservice               ClusterIP   10.84.3.155    <none>        9555/TCP       16h
cartservice             ClusterIP   10.84.7.212    <none>        7070/TCP       16h
checkoutservice         ClusterIP   10.84.2.180    <none>        5050/TCP       16h
currencyservice         ClusterIP   10.84.15.209   <none>        7000/TCP       16h
emailservice            ClusterIP   10.84.14.193   <none>        5000/TCP       16h
frontend                NodePort    10.84.12.66    <none>        80:31925/TCP   107m
paymentservice          ClusterIP   10.84.5.95     <none>        50051/TCP      69s
productcatalogservice   ClusterIP   10.84.4.148    <none>        3550/TCP       16h
recommendationservice   ClusterIP   10.84.0.16     <none>        8080/TCP       16h
redis-cart              ClusterIP   10.84.6.249    <none>        6379/TCP       16h
shippingservice         ClusterIP   10.84.12.170   <none>        50051/TCP      68s
```
[магазин](https://shop.34.116.149.237.nip.io/) заработал, ошибки пропали

**Kustomize**
[ссылку](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/) бы добавили в ДЗ ;)
...запилил, работает вроде
```
$ kustomize build ./kustomize/overrides/production/
apiVersion: v1
kind: Service
metadata:
  labels:
    environment: prod
  name: prod-cartservice
  namespace: hipster-shop-prod
spec:
  ports:
  - name: grpc
    port: 7070
    targetPort: 7070
  selector:
    app: cartservice
    environment: prod
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    environment: prod
  name: prod-cartservice
  namespace: hipster-shop-prod
spec:
  selector:
    matchLabels:
      app: cartservice
      environment: prod
  template:
    metadata:
      labels:
        app: cartservice
        environment: prod
    spec:
      containers:
      - env:
        - name: REDIS_ADDR
          value: redis-cart:6379
        - name: PORT
          value: "7070"
        - name: LISTEN_ADDR
          value: 0.0.0.0
        image: gcr.io/google-samples/microservices-demo/cartservice:v0.1.3
        livenessProbe:
          exec:
            command:
            - /bin/grpc_health_probe
            - -addr=:7070
            - -rpc-timeout=5s
          initialDelaySeconds: 15
          periodSeconds: 10
        name: server
        ports:
        - containerPort: 7070
        readinessProbe:
          exec:
            command:
            - /bin/grpc_health_probe
            - -addr=:7070
            - -rpc-timeout=5s
          initialDelaySeconds: 15
        resources:
          limits:
            cpu: 300m
            memory: 128Mi
          requests:
            cpu: 200m
            memory: 64Mi
```

