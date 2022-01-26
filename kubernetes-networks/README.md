## kubernetes-networks

> Подготовка к выполнению ДЗ
На домашнем компьютере запустил в vmware player два экземпляра CentOS 8. Сеть в режиме бриджа, адресация из домашней сети. Установил kubectl & minikube

На основе ранее созданного [web-pod.yaml](https://github.com/otus-kuber-2021-12/Zeppelinuz_platform/blob/main/kubernetes-intro/web-pod.yaml) создал кривой манифест с добавлением .readinessProbe & .livenessProbe
Запустил под с новой конфигурацией

**Вопрос для самопроверки**
1 - Почему следующая конфигурация валидна, но не имеет смысла?
```
$ minikube ssh
$ ps aux | grep my_web_server_process
docker     14228  0.0  0.0   3312   664 pts/1    S+   10:32   0:00 grep --color=auto my_web_server_process
```
как видим, возвращяется сам процесс **grep** - что бессмысленно
2 - Бывают ли ситуации, когда она все-таки имеет смысл?
если необходимо именно grep использовать, тогда вароятно, имеет смысл использовать такую команду:
```
$ ps ax | grep my_web_server_process | grep -v grep
$
```
**Создание Deployment**
Создал манифест для Deployment, подглядывая в предоставленный пример чтобы с отступами не напутать (всё равно в двух местах напутал). Применил манифест. Describe показывает ошибку MinimumReplicasUnavailable:
```
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  <none>
NewReplicaSet:   web-765dc5dbfd (1/1 replicas created)
```
Исправил .ReadinessProbe и увеличил кол-во реплик до трёх, применил манифест, завелось:
```
$ kubectl get pod -A | head -4
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
default       web-59fd746bc8-7ng5d               1/1     Running   0             2m11s
default       web-59fd746bc8-bm7k8               1/1     Running   0             2m42s
default       web-59fd746bc8-nxl7c               1/1     Running   0             2m28s
```
```
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   web-59fd746bc8 (3/3 replicas created)
```
Эксперименты с вариантами деплоя .maxSurge & .maxUnavailable
1. оба 0 - ошибка:
```
The Deployment "web" is invalid: spec.strategy.rollingUpdate.maxUnavailable: Invalid value: intstr.IntOrString{Type:0, IntVal:0, StrVal:""}: may not be 0 when `maxSurge` is 0
```
2. оба 100% - ok
```
$ kubespy trace deploy web
[MODIFIED apps/v1/Deployment]  default/web
    Rolling out Deployment revision 2
    ✅ Deployment is currently available
    ✅ Rollout successful: new ReplicaSet marked 'available'

ROLLOUT STATUS:
- [Current rollout | Revision 2] [ADDED]  default/web-59fd746bc8
    ✅ ReplicaSet is available [3 Pods available of a 3 minimum]
       - [Ready] web-59fd746bc8-bm7k8
       - [Ready] web-59fd746bc8-7ng5d
       - [Ready] web-59fd746bc8-nxl7c
```
3. maxUnavailable - 0, maxSurge - 100%
....а вот дальше меняешь, применяешь, говорит что configured, а поды не перезапускает
вон они, по полчаса висят уже:
```
$ kubectl get pod -A | head -4
NAMESPACE     NAME                               READY   STATUS    RESTARTS       AGE
default       web-59fd746bc8-7ng5d               1/1     Running   0              32m
default       web-59fd746bc8-bm7k8               1/1     Running   0              32m
default       web-59fd746bc8-nxl7c               1/1     Running   0              32m
```
> ладно, не страшно, из предыдущих занятий, мы помним, что
.maxUnavailable - максимальное количество подов, которые могут быть недоступны в процессе обновления,
.maxSurge - максимальное количество подов, которые можно создать сверх желаемого количества подов.

**Создание Service**
Создал манифест, применил, есть контакт:
```
$ kubectl get service
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP   3h16m
web-svc-cip   ClusterIP   10.105.54.20   <none>        80/TCP    15s
```
проверяем:
```
# curl -s http://10.105.54.20/index.html | grep KUBER
export KUBERNETES_PORT='tcp://10.96.0.1:443'
export KUBERNETES_PORT_443_TCP='tcp://10.96.0.1:443'
export KUBERNETES_PORT_443_TCP_ADDR='10.96.0.1'
export KUBERNETES_PORT_443_TCP_PORT='443'
export KUBERNETES_PORT_443_TCP_PROTO='tcp'
export KUBERNETES_SERVICE_HOST='10.96.0.1'
export KUBERNETES_SERVICE_PORT='443'
export KUBERNETES_SERVICE_PORT_HTTPS='443'
#
# ping 10.105.54.20
PING 10.105.54.20 (10.105.54.20) 56(84) bytes of data.
--- 10.105.54.20 ping statistics ---
6 packets transmitted, 0 received, 100% packet loss, time 5109ms
#
# ip add sh | grep "10.105.54.20"
#
```
> *arp - команды не было, ставить не стал и так понятно что нету

а вот в iptables:
```
Chain KUBE-SERVICES (2 references)
   10   600 KUBE-SVC-6CZTMAROCN3AQODZ  tcp  --  *      *       0.0.0.0/0            10.105.54.20         /* default/web-svc-cip cluster IP */ tcp dpt:80
...
Chain KUBE-SVC-6CZTMAROCN3AQODZ (1 references)
    10   600 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.105.54.20         /* default/web-svc-cip cluster IP */ tcp dpt:80
```
**Включение IPVS**
с помощью команды
```
kubectl --namespace kube-system edit configmap/kube-proxy
```
отредактировал
> это ничего, что у меня в отличие от примера в документации ДЗ:
    параметр metricsBindAddress: 127.0.0.1:10249
стоит другой IP:
    metricsBindAddress: 0.0.0.0:10249
    ?

сохранил, перезапустил
```
$ kubectl --namespace kube-system edit configmap/kube-proxy
configmap/kube-proxy edited
$ kubectl --namespace kube-system delete pod --selector='k8s-app=kube-proxy'
pod "kube-proxy-npdtv" deleted
$ kubectl get pod -A | grep kube-proxy
kube-system   kube-proxy-thkzm                   1/1     Running   0               26s
```
после очистки правил iptables, последний изменился

Как посмотреть конфигурацию IPVS?
В ВМ выполним команду toolbox
> вот тут затык, видимо потоу что у меня podman и там контейрена такого нет ?
```
$ minikube ssh
Last login: Sat Jan 22 13:25:46 2022 from 192.168.49.1
docker@minikube:~$
docker@minikube:~$ toolbox
-bash: toolbox: command not found
docker@minikube:~$
docker@minikube:~$ docker ps | grep tool
docker@minikube:~$
```

в обoем не важно, ping есть и интерфейс тоже
```
$ ping 10.105.54.20
PING 10.105.54.20 (10.105.54.20) 56(84) bytes of data.
64 bytes from 10.105.54.20: icmp_seq=1 ttl=64 time=0.046 ms
64 bytes from 10.105.54.20: icmp_seq=2 ttl=64 time=0.039 ms
64 bytes from 10.105.54.20: icmp_seq=3 ttl=64 time=0.037 ms

--- 10.105.54.20 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2067ms
rtt min/avg/max/mdev = 0.037/0.040/0.046/0.003 ms
```
```
$ ip a sh kube-ipvs0
26: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether 02:0a:06:d8:c2:2e brd ff:ff:ff:ff:ff:ff
    inet 10.96.0.10/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.105.54.20/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.1/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
```
## Работа с LoadBalancer и Ingress 

**Установка MetalLB**

> Продолжил на следующий день, запустил виртуалки, применил манифесты, добавил ipvs, в общем вчерашние процедуры. Всё завелось, естественно изменился IP на интерфейсе kube-ipvs0.

применил манифесты
```
$ kubectl --namespace metallb-system get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-6fb5bff444-pngdj   1/1     Running   0          3m14s
pod/speaker-cdxrc                 1/1     Running   0          3m15s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset.apps/speaker   1         1         1       1            1           beta.kubernetes.io/os=linux   3m15s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           3m15s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-6fb5bff444   1         1         1       3m14s
```
создал metallb-config.yaml, применил
```
$ kubectl apply -f ./metallb-config.yaml
configmap/config created
```
создал и применил web-svc-lb.yaml, IP адрес - такой:
```
$ kubectl --namespace metallb-system logs pod/controller-6fb5bff444-pngdj | grep "IP address assigned by controller"
{"caller":"service.go:114","event":"ipAllocated","ip":"172.17.255.1","msg":"IP address assigned by controller","service":"default/web-svc-lb","ts":"2022-01-23T04:57:56.560503712Z"}
$
$ kubectl describe svc web-svc-lb | grep LoadBalancer
Type:                     LoadBalancer
LoadBalancer Ingress:     172.17.255.1
```
найдём IP виртуалки с Minikube:
```
$ minikube ssh
$ ip a sh eth0 | grep inet
    inet 192.168.49.2/24 brd 192.168.49.255 scope global eth0
```
или проще:
```
$ minikube ip
192.168.49.2
```
добваим маршрут
```
$ sudo route add -net 172.17.255.0/24 gw 192.168.49.2
$
```
**проверяем наличие космических кораблей:**
```
# lynx http://172.17.255.1
0111010011111011110010000111011000001110000110010011101000001100101011110010100111010001111101001011000001110110101110111001000110
1001000000100011101100010111010111001011010111111001101101100100111111101101101001001111010111100111101010010011011010010100111110
1001100000000010100100000010100110111100010100101101011100010011011010101111010110101101101011001111110101000001011110111101100110
0101110001011010100110110010110000110110011110010100110111010010000000101011010011001100100100001001101101110011010000000011100010
1111110111110011100010001101110000000110110010100101011010010111011100000101111110110010101011110110001110100111001010111110010111
1110101000011101110011100010000000100001010110010100101101101111111110000111100111111010010000100001010001010111010100110010111001
1001100111101000100001010001010111001110010010100100110101101101110110110111110101000001101100100110000110001110100111011100000110
1101000101100100010011000111000100100111101001110101100011000100110000010101111111100101111011111111110110010101100010010101111010
1001100110010111001111111000101100001111000011010010010010111110010110010110010010101001001010011100100010000000111110000001101110
1000001110001100100010000101010100000010101001101001010010000011111011011100100010010011010001000000001010110001100111101101100111
1010110011001001011011011111101001110000101110000111100000110110110110001110010101010001110001100100101001000010011011111101101101
0000111110010010100000100110111000101100101011110011011111101011000111100101011001111100011110000110111010011000010000101000110111
1111101000000001011011100000101110100000110101111100110100110100000011101011001010010110000000100011011110000011010110100100001110
1100111100010010010001011011001000001011110110100111110000010010101111110110101111100110111000110010101000001011001011000011101110
1011000100111111010010010000101001111101111011011101010111010111111101000101000101010111011010111111111000011000100110111100101010
0001000101100001100100001011000011001101011101010111010000001011010001010111000001110110100000001000011100000101000100011011110101
1100101010110001001000010100011101111111000011110011100101101011111111110101100000001101010000000101100100101111011010011001001111
1010010100111000001001100110110101000101011100111001001010010010010111010110111111111101101100000001001011100110110110010101110100
0001011110100010110011110110000000101111110100110000110101000100101010010111110000101111110100110100001101111110010001101101011001
0101001000100011101111101100010010101001111110100101111010011100001110010101010010010100111101011000110010001100110101011010100010
0000011010010001001010011010111010010011010101110100011001011000110111111111000101010001101001111011001101100001100111110001010001
0010010110101010000100011110111101111101101011010011100101001110111100010111111110111001110010101001101110000101011110111011011001
0111101010001110001010001001100001010101111011101101110110101001010110011110000111100110011100100101101010011001000111001110011100
0101111100101011000000001011000110100000101010010001000100110010100100000011001110000110100111101110011011010111111101000011011000
0000000101101100010011101100001000111011001011100111010100110011000011110000000010000001011100111010011100111100111011110011010011
0100010001101110110001100011010010000011001001011101011111010111111101001000001111111100000110110001111101101001000010111001111100
0110111101010010101010101101101111011010110011101000110110101110111010011110111000110100110111010000110110110001010111001111000011
1010110000100110111001011010010111000010001100101011001010110101010010010010011100011000100010101011111001000000111110101011011111
1111001101001010101110010111100001001000111001000101101010111110111000011000100111010000101001010100100100110110100111001000110001
0110111011111001011011101010110110010110011101101111000101010110101100010011110101011010001111011110101110111000011110110110100000
0001001111111101010101100101111100100110001111000011000010111101010111111110100111001101101110100111011101101100110001001011001011
0001001101101010101111111010011101110000111010101100100111011010111011010011011001011000010110100000011100110000100110111010010110
0111110000111100111101010010110110101111000100010100110100001010110000111101101110011111010001101001111100111111110010001000101100
1001110110110101101000111101110111010001101100011011110000111100011111111110010000001001111100010000110111001101110110011110001111
0110001011000010111110000110011010010100110110101100001111010111101100000110001100100101100011101111101111011010101010100111101101
0110010101001000110001100010011110111010110000111011101110010010111100011100110000011100000001101010010110100000111100101000111110
0101111100010111001010001111011100001000000111001000001110100000010100010101111101000011000010010110010010101010101111111011010100
1100011000000111101100010110000010011010100101011100111111001001011100011010000000110010001110010010001001001110100011000000101001
0010001111001001101111110010001110101010101111101100110000001111100110101110100010011101010000101011001111100010010001011110000101
0000011011010001000100111000111010001000101011101111000010011001111001101100111100101001111111010111010010010001011100001101001011
1010110100111110000011011100001001110010111101110111101000111100110110100011100111111001101010000001110111101001001111110100111001
1010010111101011111011001011110110111101001000110101111011100000100000011011100110011000110011101011100100010111010001010101100110
1000011010101011110110110011111000101011101101100010110111111001010010001001011011100000101010110110100010011010011001010010111001
0010010100000001101101011010000101100000111001001001011111011101111100001001111110011110101011110010110101000000110100101100101110
0110010100110000011010100010111000000010011001110101010101010000000100000110110001010101111011011010011011000011001011111000010101
1010001101000001101101110001101110100111010010000111011011110001110101001101001010110100100000110010100110111101111111010000010010
1001111010100010011010011010100101000110110110001001001111100101100010100010111101011100111100100101011011111111100000010111110110
1111110011010101001000011100110000010101000011110110010000111111000000110010000011001001101100010101100110001101001110011110000101
1000101101011101110110110111000111110010101010011001110010101111000011110110110001001000110010111001101011011101010110100011101001
0010011101000101111101011111101011010000001010000000010101011000010011010111000101000010110100000001000010000010010010001110010111
1100001100011001010101000110100001011100100011111101010000101001101100111000111100001000110100111101010000000000111000010110011001
1000011000011100000010101100110000001010001000010100111101110010001011110001111110011001100001011000110101001010011111011100001101
1010011110011101011000100111110000111010001101101010001111011110100110111001101001011000110111111100111101000111010111110010001100
1000101110101101111000111110011000110100000000111101010110100111001011111000100101011011111110000000010110011000111010100000010000
1100101111001111101101010011111000101011011111000110000001101111100000111011000011101000111110011011000010000000100011100100011111


  Mountpoints

overlay on / type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/XWHDSIZJIMQTR6FHWXCJSUTW3E:/var/lib/docker/overlay2/l/WY5E7XU4DTEERI2PUDXLYM6VSI,upperdir=/var/lib/docker/overlay2/0b5e2f7b0d64c94bbeddea3050e5ee7e969876be993a8f9adbad821fed40eac3/diff,w
orkdir=/var/lib/docker/overlay2/0b5e2f7b0d64c94bbeddea3050e5ee7e969876be993a8f9adbad821fed40eac3/work)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,size=65536k,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
sysfs on /sys type sysfs (ro,nosuid,nodev,noexec,relatime)
tmpfs on /sys/fs/cgroup type tmpfs (rw,nosuid,nodev,noexec,relatime,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (ro,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/memory type cgroup (ro,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (ro,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/blkio type cgroup (ro,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (ro,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/freezer type cgroup (ro,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/devices type cgroup (ro,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/perf_event type cgroup (ro,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/rdma type cgroup (ro,nosuid,nodev,noexec,relatime,rdma)
cgroup on /sys/fs/cgroup/rdma/libpod_parent/libpod-fa20f681213f0096dc9ea930b304fe81aa947822d567618d7514da392d7e7e81 type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
cgroup on /sys/fs/cgroup/pids type cgroup (ro,nosuid,nodev,noexec,relatime,pids)
... и т.д.
```
Это тот кораблик или где то другой есть?


**Задание со ⭐ | DNS через MetalLB**
> спасибо за [Hint](https://metallb.universe.tf/usage/)
воспользуемся "IP address sharing" - By default, Services do not share IP addresses. If you have a need to colocate services on a single IP, you can enable selective IP sharing by adding the .metallb.universe.tf/allow-shared-ip annotation to services.


Нарисовал два манифеста для каждого протокола, 

> сначала попробовал использовать protocol: ANY, но получил ошибку:
The Service "dns-sharing" is invalid: spec.ports[0].protocol: Unsupported value: "ANY": supported values: "SCTP", "TCP", "UDP"
о, SCTP поддерживает :)

применил два манифеста для каждого протокола, проверил сервисы
```
$ kubectl apply -f ./coredns/sharing-tcp-dns.yaml
service/dns-sharing-tcp created
$ kubectl apply -f ./coredns/sharing-udp-dns.yaml
service/dns-sharing-udp created
$
$ kubectl get service
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
dns-sharing-tcp   LoadBalancer   10.97.111.222    172.17.255.53   53:30158/TCP   2m10s
dns-sharing-udp   LoadBalancer   10.107.144.27    172.17.255.53   53:30736/UDP   2m4s
```
IP на интерфейсе имеется:
```
minikube:~# ip a sh kube-ipvs0
18: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether de:a0:36:53:73:ca brd ff:ff:ff:ff:ff:ff
.....
    inet 172.17.255.53/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
```
**Создание Ingress**

Ссылка в документации ДЗ - 404,
В [документации](https://kubernetes.github.io/ingress-nginx/deploy/) другая
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
deployment.apps/ingress-nginx-controller created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
serviceaccount/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created                                                                      job.batch/ingress-nginx-admission-patch created
```
Создал файл nginx-lb.yaml, применил
```
$ kubectl apply -f ./nginx-lb.yaml
service/ingress-nginx created
```
смотрим какой IP
> свет выключали на два часа, поэтому всё заново создавал - все IP поменялись

```
$ kubectl get service --namespace ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
ingress-nginx                        LoadBalancer   10.98.139.15     172.17.255.3   80:30580/TCP,443:31710/TCP   3m36s
ingress-nginx-controller             LoadBalancer   10.101.163.193   172.17.255.2   80:31103/TCP,443:30199/TCP   4m7s
ingress-nginx-controller-admission   ClusterIP      10.104.153.126   <none>         443/TCP                      4m7s
```
проверяем get & curl
```
$ ping 172.17.255.3
PING 172.17.255.3 (172.17.255.3) 56(84) bytes of data.
64 bytes from 172.17.255.3: icmp_seq=1 ttl=64 time=0.100 ms
64 bytes from 172.17.255.3: icmp_seq=2 ttl=64 time=0.066 ms
^C
--- 172.17.255.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1051ms
rtt min/avg/max/mdev = 0.066/0.083/0.100/0.017 ms
[dos@minikube-2 kubernetes-networks]$ curl http://172.17.255.3
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```
идём дальше
создал новый манифест, применил, результат:
```
$ kubectl apply -f ./web-svc-headless.yaml
service/web-svc created
$ kubectl get service
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
dns-sharing-tcp   LoadBalancer   10.105.155.147   172.17.255.53   53:31601/TCP   10m
dns-sharing-udp   LoadBalancer   10.106.232.57    172.17.255.53   53:30179/UDP   10m
kubernetes        ClusterIP      10.96.0.1        <none>          443/TCP        35m
web-svc           ClusterIP      None             <none>          80/TCP         2s
web-svc-cip       ClusterIP      10.108.128.143   <none>          80/TCP         28m
web-svc-lb        LoadBalancer   10.100.37.86     172.17.255.1    80:32670/TCP   17m
```
CLUSTER-IP для сервиса web-svc = None

далее скопипастил web-ingress.yaml
применил и получил:
```
error: unable to recognize "./web-ingress.yaml": no matches for kind "Ingress" in version "networking.k8s.io/v1beta1"
```
поменял на "networking.k8s.io/v1" - получил ошибку:
```
error: error validating "./web-ingress.yaml": error validating data: ValidationError(Ingress.spec.rules[0]): unknown field "paths" in io.k8s.api.networking.v1.IngressRule; if you choose to ignore these errors, turn validation off with --validate=false
```
**Как выяснилось, с версии 1.19 синтаксис изменился**
`Ingress` and `IngressClass` resources have graduated to `networking.k8s.io/v1`. Ingress and IngressClass types in the `extensions/v1beta1` and `networking.k8s.io/v1beta1` API versions are deprecated and will no longer be served in 1.22+. Persisted objects can be accessed via the `networking.k8s.io/v1` API. Notable changes in v1 Ingress objects (v1beta1 field names are unchanged):
* `spec.backend` -> `spec.defaultBackend`
* `serviceName` -> `service.name`
* `servicePort` -> `service.port.name` (for string values)
* `servicePort` -> `service.port.number` (for numeric values)
* `pathType` no longer has a default value in v1; "Exact", "Prefix", or "ImplementationSpecific" must be specified
Other Ingress API updates:
* backends can now be resource or service backends
* `path` is no longer required to be a valid regular expression

Как видим, не только "path" поменялся - [тут документация](https://kubernetes.io/docs/concepts/services-networking/ingress/)
В общем переписал весь манифест, применил
```
$ kubectl apply -f ./web-ingress.yaml
ingress.networking.k8s.io/web created
```
проверяем
```
$ kubectl describe ingress/web
Name:             web
Labels:           <none>
Namespace:        default
Address:
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /web   web-svc:8000 (172.17.0.3:8000,172.17.0.4:8000,172.17.0.5:8000)
Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
```
IP адреса - нет и ошибка **error: endpoints "default-http-backend" not found**
на вызовы curl-om, сплошные 404 в ответ
```
$ kubectl get service
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
dns-sharing-tcp   LoadBalancer   10.105.155.147   172.17.255.53   53:31601/TCP   55m
dns-sharing-udp   LoadBalancer   10.106.232.57    172.17.255.53   53:30179/UDP   55m
kubernetes        ClusterIP      10.96.0.1        <none>          443/TCP        80m
web-svc           ClusterIP      None             <none>          80/TCP         45m
web-svc-cip       ClusterIP      10.108.128.143   <none>          80/TCP         73m
web-svc-lb        LoadBalancer   10.100.37.86     172.17.255.1    80:32670/TCP   62m
$
$ curl http://172.17.255.1/web/index.html
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```
адреса нет:
```
$ kubectl get ingress -n default
NAME   CLASS    HOSTS   ADDRESS   PORTS   AGE
web    <none>   *                 80      42m
```
**куда копать, где я адрес пропустил ?**

+++++++++++++++++++++++++++++++

После [этой](https://kubernetes.github.io/ingress-nginx/user-guide/default-backend/) и [этой](https://kubernetes.github.io/ingress-nginx/examples/rewrite/) подсказок

Спасибо Олегу Константинову!

всё завелось после модернизации манифеста web-ingress.yaml
```
$ curl http://172.17.255.2/web/index.html | grep KUBER
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 83864  100 83864    0     0  79.9M      0 --:--:-- --:--:-- --:--:-- 79.9M
export KUBERNETES_PORT='tcp://10.96.0.1:443'
export KUBERNETES_PORT_443_TCP='tcp://10.96.0.1:443'
export KUBERNETES_PORT_443_TCP_ADDR='10.96.0.1'
export KUBERNETES_PORT_443_TCP_PORT='443'
export KUBERNETES_PORT_443_TCP_PROTO='tcp'
export KUBERNETES_SERVICE_HOST='10.96.0.1'
export KUBERNETES_SERVICE_PORT='443'
export KUBERNETES_SERVICE_PORT_HTTPS='443'
```

** без звёздочек


