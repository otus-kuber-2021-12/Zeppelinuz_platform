## kubernetes-logging

Через веб-интерфейс создал 1 ноду n1-standard-2 в default-pool и три ноды в infra-pool.
Stackdriver (логирование средствами облачной платформы) - выключил.
Пришлось повозится, так как создавать не хотел с ошибкой:
```
Not all instances running in IGM after 31.245272567s. Expected 3, running 0, transitioning 3. Current errors:....
The zone ... does not have enough resources available to fulfill the request. Try a different zone, or try again later. 
```
после того как завелось:
```
$ kubectl get nodes
NAME                                       STATUS   ROLES    AGE   VERSION
gke-f4-hw-efk-default-pool-2ff6b893-1jkc   Ready    <none>   16m   v1.21.6-gke.1500
gke-f4-hw-efk-infra-pool-fdeb8191-9mf4     Ready    <none>   16m   v1.21.6-gke.1500
gke-f4-hw-efk-infra-pool-fdeb8191-bcrs     Ready    <none>   16m   v1.21.6-gke.1500
gke-f4-hw-efk-infra-pool-fdeb8191-c3wl     Ready    <none>   16m   v1.21.6-gke.1500
```
присвоил taints для nodes
```
$ kubectl taint nodes gke-f4-hw-efk-infra-pool-fdeb8191-9mf4 node-role=infra:NoSchedule
node/gke-f4-hw-efk-infra-pool-fdeb8191-9mf4 tainted
$ kubectl taint nodes gke-f4-hw-efk-infra-pool-fdeb8191-bcrs node-role=infra:NoSchedule
node/gke-f4-hw-efk-infra-pool-fdeb8191-bcrs tainted
$ kubectl taint nodes gke-f4-hw-efk-infra-pool-fdeb8191-c3wl node-role=infra:NoSchedule
node/gke-f4-hw-efk-infra-pool-fdeb8191-c3wl tainted
```
проверил:
```
$ kubectl get nodes -o json | jq '.items[].spec.taints'
null
[
  {
    "effect": "NoSchedule",
    "key": "node-role",
    "value": "infra"
  }
]
[
  {
    "effect": "NoSchedule",
    "key": "node-role",
    "value": "infra"
  }
]
[
  {
    "effect": "NoSchedule",
    "key": "node-role",
    "value": "infra"
  }
]
```
становил hipstershop по методичке, при этом adservice не завёлся со статусом ImagePullBackOff,
проверил по источнику - отсутсвует указанная версия, остальные нормально
#### Установка EFK стека | Helm charts
поставил компоненты, пока без дополнительных настроек
```
$ helm repo add elastic https://helm.elastic.co
"elastic" has been added to your repositories
$
$ kubectl create ns observability
namespace/observability created
$
$ helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability
Release "elasticsearch" does not exist. Installing it now.
W0219 20:50:25.445123   69383 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
W0219 20:50:29.911106   69383 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
NAME: elasticsearch
LAST DEPLOYED: Sat Feb 19 20:50:24 2022
NAMESPACE: observability
STATUS: deployed
REVISION: 1
NOTES:
1. Watch all cluster members come up.
  $ kubectl get pods --namespace=observability -l app=elasticsearch-master -w2. Test cluster health using Helm test.
  $ helm --namespace=observability test elasticsearch
$
$ helm upgrade --install kibana elastic/kibana --namespace observability
Release "kibana" does not exist. Installing it now.
NAME: kibana
LAST DEPLOYED: Sat Feb 19 20:51:07 2022
NAMESPACE: observability
STATUS: deployed
REVISION: 1
TEST SUITE: None
$
$ helm upgrade --install fluent-bit stable/fluent-bit --namespace observability
Release "fluent-bit" does not exist. Installing it now.
Error: failed to download "stable/fluent-bit"
```
> Release "fluent-bit" does not exist. Installing it now.
Error: failed to download "stable/fluent-bit"

а его там нет, опять танцы
добавил другой репозитарий и устанивил
```
$ helm repo add fluent https://fluent.github.io/helm-charts
"fluent" has been added to your repositories
$
$ helm search repo fluent
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
fluent/fluent-bit       0.19.19         1.8.12          Fast and lightweight log processor and forwarde...
fluent/fluentd          0.3.5           v1.12.4         A Helm chart for Kubernetes
$
$ helm upgrade --install fluent-bit fluent/fluent-bit --namespace observability
Release "fluent-bit" does not exist. Installing it now.
NAME: fluent-bit
LAST DEPLOYED: Sat Feb 19 21:14:08 2022
NAMESPACE: observability
STATUS: deployed
REVISION: 1
NOTES:
Get Fluent Bit build information by running these commands:

export POD_NAME=$(kubectl get pods --namespace observability -l "app.kubernetes.io/name=fluent-bit,app.kubernetes.io/instance=fluent-bit" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace observability port-forward $POD_NAME 2020:2020
curl http://127.0.0.1:2020
```
далее по методичке с созданием манифеста elasticsearch.values.yaml
результат манипуляций:
```
$ kubectl get pods -n observability -o wide
NAME                            READY   STATUS    RESTARTS   AGE     IP           NODE                                       NOMINATED NODE   READINESS GATES
elasticsearch-master-0          1/1     Running   0          7m36s   10.40.3.9    gke-f4-hw-efk-infra-pool-fdeb8191-9mf4     <none>           <none>
elasticsearch-master-1          1/1     Running   0          9m4s    10.40.1.5    gke-f4-hw-efk-infra-pool-fdeb8191-bcrs     <none>           <none>
elasticsearch-master-2          1/1     Running   0          10m     10.40.2.3    gke-f4-hw-efk-infra-pool-fdeb8191-c3wl     <none>           <none>
fluent-bit-bvc6n                1/1     Running   0          81s     10.40.0.18   gke-f4-hw-efk-default-pool-2ff6b893-1jkc   <none>           <none>
kibana-kibana-fc976c796-xd5wx   1/1     Running   0          24m     10.40.0.17   gke-f4-hw-efk-default-pool-2ff6b893-1jkc   <none>           <none>
```
#### Установка nginx-ingress | Самостоятельное задание
результат:
```
$ kubectl get pods -n nginx-ingress
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-557cbb4fcb-c8sp2   1/1     Running   0          65s
ingress-nginx-controller-557cbb4fcb-p7kvg   1/1     Running   0          65s
ingress-nginx-controller-557cbb4fcb-w2nhd   1/1     Running   0          65s
$
$ kubectl get pods -n nginx-ingress -o wide
NAME                                        READY   STATUS    RESTARTS   AGE     IP           NODE                                     NOMINATED NODE   READINESS GATES
ingress-nginx-controller-557cbb4fcb-c8sp2   1/1     Running   0          2m38s   10.40.3.10   gke-f4-hw-efk-infra-pool-fdeb8191-9mf4   <none>           <none>
ingress-nginx-controller-557cbb4fcb-p7kvg   1/1     Running   0          2m38s   10.40.1.6    gke-f4-hw-efk-infra-pool-fdeb8191-bcrs   <none>           <none>
ingress-nginx-controller-557cbb4fcb-w2nhd   1/1     Running   0          2m38s   10.40.2.4    gke-f4-hw-efk-infra-pool-fdeb8191-c3wl   <none>           <none>
```
#### Установка EFK стека | Kibana
установил: http://kibana.35.225.123.83.nip.io
дальше игры с fluent-bit
```
$ helm upgrade --install fluent-bit fluent/fluent-bit --namespace observability -f ./fluentbit.values.yaml
Release "fluent-bit" has been upgraded. Happy Helming!
NAME: fluent-bit
LAST DEPLOYED: Sat Feb 19 22:16:40 2022
NAMESPACE: observability
STATUS: deployed
REVISION: 2
NOTES:
Get Fluent Bit build information by running these commands:

export POD_NAME=$(kubectl get pods --namespace observability -l "app.kubernetes.io/name=fluent-bit,app.kubernetes.io/instance=fluent-bit" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace observability port-forward $POD_NAME 2020:2020
curl http://127.0.0.1:2020
```
#### Мониторинг ElasticSearch
поставил прометеус и эластиксерч-эксплорер
```
$ helm upgrade --install prometheus-operator prometheus-community/kube-prometheus-stack -n observability --create-namespace -f ./prometheus.values.yaml
Release "prometheus-operator" does not exist. Installing it now.
NAME: prometheus-operator
LAST DEPLOYED: Sat Feb 19 23:55:25 2022
NAMESPACE: observability
STATUS: deployed
REVISION: 1
$
$ helm upgrade --install elasticsearch-exporter prometheus-community/prometheus-elasticsearch-exporter --set es.uri=http://elasticsearch-master:9200 --set serviceMonitor.enabled=true --namespace=observability
Release "elasticsearch-exporter" does not exist. Installing it now.
NAME: elasticsearch-exporter
LAST DEPLOYED: Sun Feb 20 00:02:56 2022
NAMESPACE: observability
STATUS: deployed
REVISION: 1
```
прометеус - http://prometheus.35.225.123.83.nip.io/
графана - http://grafana.35.225.123.83.nip.io/
логин - пасс: admin - prom-operator123
и далее по методичке
#### EFK | nginx ingress
логов nginx нет
внёс изменения в fluentbit.values.yaml чтобы запускались на нодах infra, запустил
изменил лог-формат, создал дашборд в kibane, экспортировал, приложил и по ссылкам доступны

#### Loki
поставил, получилось следующее:
```
$ kubectl get pods -n observability
NAME                                                              READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-operator-kube-p-alertmanager-0            2/2     Running   0          18h
elasticsearch-exporter-prometheus-elasticsearch-exporter-c47pd2   1/1     Running   0          18h
elasticsearch-master-0                                            1/1     Running   0          21h
elasticsearch-master-1                                            1/1     Running   0          21h
elasticsearch-master-2                                            1/1     Running   0          21h
fluent-bit-457nt                                                  1/1     Running   0          18h
fluent-bit-b6jzp                                                  1/1     Running   0          18h
fluent-bit-t5p9h                                                  1/1     Running   0          18h
fluent-bit-tkjc5                                                  1/1     Running   0          18h
kibana-kibana-fc976c796-xd5wx                                     1/1     Running   0          22h
loki-0                                                            1/1     Running   0          96s
loki-promtail-s96s2                                               1/1     Running   0          97s
loki-promtail-s99ln                                               1/1     Running   0          97s
loki-promtail-t4kfm                                               1/1     Running   0          97s
loki-promtail-txp9g                                               1/1     Running   0          97s
prometheus-operator-grafana-5f45b9dc57-x6tvr                      3/3     Running   0          18h
prometheus-operator-kube-p-operator-654ccf58ff-fg8bd              1/1     Running   0          18h
prometheus-operator-kube-state-metrics-764767b8f5-pb24d           1/1     Running   0          18h
prometheus-operator-prometheus-node-exporter-6xpd8                1/1     Running   0          18h
prometheus-operator-prometheus-node-exporter-bxd5c                1/1     Running   0          18h
prometheus-operator-prometheus-node-exporter-sj4gg                1/1     Running   0          18h
prometheus-operator-prometheus-node-exporter-vw27z                1/1     Running   0          18h
prometheus-prometheus-operator-kube-p-prometheus-0                2/2     Running   0          18h
```
json-дашборда приложен
#### Event logging | k8s-event-logger
```
$ helm repo add deliveryhero https://charts.deliveryhero.io/
"deliveryhero" has been added to your repositories
$
$ helm upgrade --install k8s-event-logger deliveryhero/k8s-event-logger -n observability --create-namespace
Release "k8s-event-logger" does not exist. Installing it now.
W0220 19:53:25.282031    2132 warnings.go:70] rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
W0220 19:53:27.467410    2132 warnings.go:70] rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
NAME: k8s-event-logger
LAST DEPLOYED: Sun Feb 20 19:53:24 2022
NAMESPACE: observability
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To verify that the k8s-event-logger pod has started, run:

  kubectl --namespace=observability get pods -l "app.kubernetes.io/name=k8s-event-logger,app.kubernetes.io/instance=k8s-event-logger"
$
$ kubectl get pods -n observability | grep logger
k8s-event-logger-7798475856-x6qss                                 1/1     Running   0          42s
```

