# Zeppelinuz_platform
Zeppelinuz Platform repository

Домашняя работа 1 - kubernetes-intro
kube-apiserver предназначен для горизонтального масштабирования;
etcd распределённое и высоконадёжное хранилище данных в формате "ключ-значение", которое используется как основное хранилище всех данных кластера в Kubernetes;
kube-scheduler отслеживает созданные поды без привязанного узла и выбирает узел, на котором они должны работать;
kube-controller-manager компонент Control Plane запускает процессы контроллера (Node Controller, Replication Controller, Endpoints Controller, ccount & Token Controllers) 
Ещё есть для работы с облачными сервисами - cloud-controller-manager
А Core-DNS (реализован как Deployment) является дополнением, вроде не обязательным, однако при этом у всех Kubernetes-кластеров должен быть кластерный DNS.
Продленная работа:
Развёрнут кластер Minikube, установлен kubectl
Создан Dockerfile с минимальным конфигом на основе nginx
docker build --no-cache -t nginx-8000 .
контейнер запускается так:
docker run --mount type=bind,source=/app,target=/app,readonly -p 8000:8000 -P -d nginx-8000
после проверки, помещён в Docker Hub
манифест web-pod.yaml приложен
с frontend аналогично

$ kubectl get pod -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
default       frontend                           1/1     Running   0          26m
default       web                                1/1     Running   0          3h19m
kube-system   coredns-78fcd69978-f8kds           1/1     Running   1          26h
kube-system   etcd-minikube                      1/1     Running   5          26h
kube-system   kube-apiserver-minikube            1/1     Running   5          26h
kube-system   kube-controller-manager-minikube   1/1     Running   5          26h
kube-system   kube-proxy-nxlpl                   1/1     Running   1          26h
kube-system   kube-scheduler-minikube            1/1     Running   5          26h
kube-system   storage-provisioner                1/1     Running   0          26h

