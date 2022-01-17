## kubernetes-intro

Вопрос 1

kube-apiserver запущен как StaticPods (созданные и управляемые демоном kubelet на определенном узле без наблюдения за ними со стороны API, если такой под дает сбой, kubelet перезапускает его)
```
$ kubectl describe pod kube-apiserver-minikube -n kube-system | grep Controlled
Controlled By:  Node/minikube
```
Pod coredns управляется replicaset-ом:
```
$ kubectl describe pod coredns-78fcd69978-f8kds -n kube-system | grep Controlled
Controlled By:  ReplicaSet/coredns-78fcd69978

$ kubectl get rs -A
NAMESPACE     NAME                 DESIRED   CURRENT   READY   AGE
kube-system   coredns-78fcd69978   1         1         1       2d
```
который управляется деплойментом:
```
$ kubectl describe rs -A
Name:           coredns-78fcd69978
Namespace:      kube-system
Selector:       k8s-app=kube-dns,pod-template-hash=78fcd69978
Labels:         k8s-app=kube-dns
                pod-template-hash=78fcd69978
Annotations:    deployment.kubernetes.io/desired-replicas: 1
                deployment.kubernetes.io/max-replicas: 2
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/coredns
Replicas:       1 current / 1 desired
Pods Status:    1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:           k8s-app=kube-dns
                    pod-template-hash=78fcd69978
  Service Account:  coredns
  Containers:
   coredns:
    Image:       k8s.gcr.io/coredns/coredns:v1.8.4
    Ports:       53/UDP, 53/TCP, 9153/TCP
    Host Ports:  0/UDP, 0/TCP, 0/TCP
    Args:
      -conf
      /etc/coredns/Corefile
    Limits:
      memory:  170Mi
    Requests:
      cpu:        100m
      memory:     70Mi
    Liveness:     http-get http://:8080/health delay=60s timeout=5s period=10s #success=1 #failure=5
    Readiness:    http-get http://:8181/ready delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /etc/coredns from config-volume (ro)
  Volumes:
   config-volume:
    Type:               ConfigMap (a volume populated by a ConfigMap)
    Name:               coredns
    Optional:           false
  Priority Class Name:  system-cluster-critical
Events:                 <none>

```
```
$ kubectl get deployment -A
NAMESPACE     NAME      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   coredns   1/1     1            1           2d
```
```
$ kubectl describe deployment -A
Name:                   coredns
Namespace:              kube-system
CreationTimestamp:      Sat, 15 Jan 2022 13:35:58 +0600
Labels:                 k8s-app=kube-dns
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               k8s-app=kube-dns
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 25% max surge
Pod Template:
  Labels:           k8s-app=kube-dns
  Service Account:  coredns
  Containers:
   coredns:
    Image:       k8s.gcr.io/coredns/coredns:v1.8.4
    Ports:       53/UDP, 53/TCP, 9153/TCP
    Host Ports:  0/UDP, 0/TCP, 0/TCP
    Args:
      -conf
      /etc/coredns/Corefile
    Limits:
      memory:  170Mi
    Requests:
      cpu:        100m
      memory:     70Mi
    Liveness:     http-get http://:8080/health delay=60s timeout=5s period=10s #success=1 #failure=5
    Readiness:    http-get http://:8181/ready delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /etc/coredns from config-volume (ro)
  Volumes:
   config-volume:
    Type:               ConfigMap (a volume populated by a ConfigMap)
    Name:               coredns
    Optional:           false
  Priority Class Name:  system-cluster-critical
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   coredns-78fcd69978 (1/1 replicas created)
Events:          <none>
```
Вот он его и запускает,
поэтому в статистике рестартов цифры у coredns и kube-apiserver различаются:
```
$ kubectl get pod -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
default       frontend                           1/1     Running   0          23h
default       web                                1/1     Running   0          26h
kube-system   coredns-78fcd69978-f8kds           1/1     Running   1          2d2h
kube-system   etcd-minikube                      1/1     Running   5          2d2h
kube-system   kube-apiserver-minikube            1/1     Running   5          2d2h
kube-system   kube-controller-manager-minikube   1/1     Running   5          2d2h
kube-system   kube-proxy-nxlpl                   1/1     Running   1          2d2h
kube-system   kube-scheduler-minikube            1/1     Running   5          2d2h
kube-system   storage-provisioner                1/1     Running   0          2d1h
```

Продленная работа:

Развёрнут кластер Minikube, установлен kubectl.

Создан Dockerfile с минимальным конфигом на основе nginx
```
docker build --no-cache -t nginx-8000 .
```
контейнер запускается 
```
docker run --mount type=bind,source=/app,target=/app,readonly -p 8000:8000 -P -d nginx-8000
```
после проверки, помещён в Docker Hub

с frontend аналогично, после запуска выдавал ошибку:
```
$ kubectl logs frontend
...
panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set
...
```
после добавления environment аналогично yaml-файлу с подскзкой, под поднялся в состояние running (см выше).



