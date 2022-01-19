# ДЗ (лекция 3) kubernetes-controllers

Установил kind
При попытке создания кластера из предложенного yaml-файла, получал ошибку:
```
# kind create cluster --config ./kind-config.yaml
ERROR: failed to create cluster: unknown apiVersion: kind.sigs.k8s.io/v1alpha3
```
Выяснив в [документации](https://kind.sigs.k8s.io/docs/user/configuration/) верный apiVersion, изменил yaml-файл и всё завелось:
```
# kubectl get nodes
NAME                  STATUS   ROLES                  AGE     VERSION
kind-control-plane    Ready    control-plane,master   3m26s   v1.21.1
kind-control-plane2   Ready    control-plane,master   2m58s   v1.21.1
kind-control-plane3   Ready    control-plane,master   2m4s    v1.21.1
kind-worker           Ready    <none>                 107s    v1.21.1
kind-worker2          Ready    <none>                 107s    v1.21.1
kind-worker3          Ready    <none>                 107s    v1.21.1
```

Создал rs-манифест для frontend-а, запустил, получил ошибку:
```
# kubectl apply -f ./frontend-replicaset.yaml
error: error validating "./frontend-replicaset.yaml": error validating data: ValidationError(ReplicaSet.spec): missing required field "selector" in io.k8s.api.apps.v1.ReplicaSetSpec; if you choose to ignore these errors, turn validation off with --validate=false
```
добавил в манифест required field "selector", запустил, работает вроде
```
# kubectl apply -f ./frontend-replicaset.yaml
replicaset.apps/frontend created
#
# kubectl get pods -l app=frontend
NAME             READY   STATUS    RESTARTS   AGE
frontend-9tfz5   1/1     Running   0          31s
```

утраиваем
```
# kubectl scale replicaset frontend --replicas=3
#
# kubectl get pods -l app=frontend
NAME             READY   STATUS              RESTARTS   AGE
frontend-8m657   0/1     ContainerCreating   0          8s
frontend-9tfz5   1/1     Running             0          3m45s
frontend-lhz8p   0/1     ContainerCreating   0          8s
#
# kubectl get pods -l app=frontend
NAME             READY   STATUS    RESTARTS   AGE
frontend-8m657   1/1     Running   0          19s
frontend-9tfz5   1/1     Running   0          3m56s
frontend-lhz8p   1/1     Running   0          19s
#
# kubectl get rs frontend
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       4m4s
```
применяем манифест повторно для получения одной реплики
```
# kubectl apply -f ./frontend-replicaset.yaml
replicaset.apps/frontend configured
# kubectl get pods -l app=frontend
NAME             READY   STATUS        RESTARTS   AGE
frontend-c24tv   1/1     Running       0          115s
frontend-fx4vf   0/1     Terminating   0          115s
# kubectl get pods -l app=frontend
NAME             READY   STATUS    RESTARTS   AGE
frontend-c24tv   1/1     Running   0          119s
```
Ставим в манифесте .spec.replicas 3, применяем, получаем три реплики сразу.
Обновил версию образа на DockerHub, добавил версию в манифест, применил его, и действительно, кажется ничего не произошло.

В rs указана версия:
```
# kubectl get replicaset frontend -o=jsonpath='{.spec.template.spec.containers[0].image}'
dem0niuz/frontend:v0.0.2
```
а образ из которого запущены поды, управляемые контроллером, другие
```
# kubectl get pods -l app=frontend -o=jsonpath='{.items[0:3].spec.containers[0].image}'
dem0niuz/frontend dem0niuz/frontend dem0niuz/frontend
```
после удаления, поды запустились уже с обновлённой версией

> Вопрос - почему обновление ReplicaSet не повлекло обновление запущенных pod?

ReplicaSet гарантирует, что определенное количество экземпляров подов (Pods) будет запущено в кластере Kubernetes в любой момент времени, **и всё**. То есть rs не перезапускает поды при обновлении, в отличие от Deployment (rolling-update/rollback).

## Deployment

Микросервис paymentService запущен в 3-х репликах из манифеста, ошибки слизаны из [примера](https://github.com/GoogleCloudPlatform/microservices-demo/blob/main/kubernetes-manifests/paymentservice.yaml).
создал манифест paymentservice-deployment.yaml и применил (предварительно удалив предыдущий, запущенный rs):
```
# kubectl get deployment
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
paymentservice   3/3     3            3           22s
#
# kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
frontend                    3         3         3       84m
paymentservice-5cb9d76cf9   3         3         3       15s
```
обновление и откат
```
# kubectl get pods -l app=paymentservice -o=jsonpath='{.items[0:3].spec.containers[0].image}'
dem0niuz/paymentservice:v0.0.1 dem0niuz/paymentservice:v0.0.1 dem0niuz/paymentservice:v0.0.1
```
после обновления имеем два rs-a
```
# kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
frontend                    3         3         3       92m
paymentservice-5cb9d76cf9   0         0         0       7m46s
paymentservice-6cf967884d   3         3         3       44s
# kubectl get pods -l app=paymentservice -o=jsonpath='{.items[0:3].spec.containers[0].image}'
dem0niuz/paymentservice:v0.0.2 dem0niuz/paymentservice:v0.0.2 dem0niuz/paymentservice:v0.0.2
# kubectl rollout history deployment paymentservice
deployment.apps/paymentservice
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```
После отката
```
# kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
frontend                    3         3         3       95m
paymentservice-5cb9d76cf9   3         3         3       11m
paymentservice-6cf967884d   0         0         0       4m26s
#
# kubectl get pods -l app=paymentservice -o=jsonpath='{.items[0:3].spec.containers[0].image}'
dem0niuz/paymentservice:v0.0.1 dem0niuz/paymentservice:v0.0.1 dem0niuz/paymentservice:v0.0.1
```

> Deployment задание со *
> Аналог blue-green: развернуть три новых pod-а, затем удалить троих старых.
> Reverse Rolling Update: удаление одного старого, создание нового, удаление следующего старого, создание следующего нового и т.д.

Реализуется через стратегию: **.spec.strategy.rollingUpdate**:
**.maxUnavailable** - максимальное количество подов, которые могут быть недоступны в процессе обновления;
**.maxSurge** - максимальное количество подов, которые можно создать сверх желаемого количества подов.

## Probes

Создал frontend-deployment.yaml по аналогии из файла по ссылке в ДЗ, добавил .readinessProbe,
применил манифест, поды запустились:
```
# kubectl get pod -A
NAMESPACE            NAME                                          READY   STATUS    RESTARTS   AGE
default              frontend-6d974455dd-2gqq6                     1/1     Running   0          5m
default              frontend-6d974455dd-hsjk5                     1/1     Running   0          5m
default              frontend-6d974455dd-swnsn                     1/1     Running   0          5m
```
в описании присутствуют параметры:
```
# kubectl describe pod frontend-6d974455dd-2gqq6 | grep Readi
    Readiness:      http-get http://:8080/_healthz delay=10s timeout=1s period=10s #success=1 #failure=3
```

Далее обновил тэг до 0.0.2 и испортил URL /_healthz на /_health
Запустил обновление, создался один pod  с ошибкой
```
Warning  Unhealthy  1s (x12 over 111s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 404
```
Исправил, запустил обновление, как результат
```
# kubectl rollout status deployment/frontend
deployment "frontend" successfully rolled out
```

## DaemonSet

Вбил в поисковик название файла, скачал первый попавшийся, запустил, работает.
```
# kubectl apply -f ./node-exporter-daemonset.yaml
daemonset.apps/node-exporter created
```
**curl localhost:9100/metrics* – выдаёт портянку на 1245 строк

[Здесь](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) написано, что добавить в манифест, чтобы запускалось на мастерах тоже:
```
spec:
    tolerations:
    # this toleration is to have the daemonset runnable on master nodes
    # remove it if your masters can't run pods
```
В моём случае, слитый манифест уже имел эти директивы.



