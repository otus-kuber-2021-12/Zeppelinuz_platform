### kubectl debug / strace

поставил...
```
curl -Lo kubectl-debug https://github.com/JamesTGrant/kubectl-debug/releases/download/v1.0.0/kubectl-debug
```
...что-то ошибки постоянные:
> an error occured executing remote command(s), Internal error occurred: error attaching to container: /var/lib/lxc/lxcfs is not a mount point, please run " lxcfs /var/lib/lxc/lxcfs " before debug
error: Internal error occurred: error attaching to container: /var/lib/lxc/lxcfs is not a mount point, please run " lxcfs /var/lib/lxc/lxcfs " before debug

хотя пути все есть и прав хватает...
мы пойдём другим путём, используя уже имеющийся (с версии 1.18) [функционал debug](https://kubernetes.io/docs/tasks/debug-application-c)
предварительно создал образ с strace-ом и другими полезными утилитами (в будущем пригодится ещё) и пушанул его в регистри
```
$ docker build -t strace .
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM oraclelinux:7
 ---> fd268884dd8a
Step 2/2 : RUN yum install -y  .....
 ---> Running in 4ba65564d1f8
Loaded plugins: ovl, ulninfo
.....
Resolving Dependencies
--> Running transaction check
---> Package bash-completion.noarch 1:2.1-8.el7 will be installed
.....
--> Processing Dependency: libaio.so.1(LIBAIO_0.1)(64bit) for package: fio-3.7-2.el7.x86_64
...и т.д.
...
Complete!
Removing intermediate container 4ba65564d1f8
 ---> e049a23f7163
Successfully built e049a23f7163
Successfully tagged strace:latest

$ docker push dem0niuz/strace
5872674bb016: Pushed
10dc5d653aef: Pushed
```
запускаю собранный контейнер в целевом поде (там где контейнер совсем без shell-а) с нашим образом
```
$ kubectl debug fluent-bit-xx6t2 -n observability -it --image=dem0niuz/strace
Defaulting debug container name to debugger-fpvm5.
If you don't see a command prompt, try pressing enter.
[root@fluent-bit-xx6t2 /]# 
[root@fluent-bit-xx6t2 /]# strace -c -p1
strace: Process 1 attached
```

### iptables-tailer
```
$ kubectl apply -f https://raw.githubusercontent.com/piontec/netperf-operator/master/deploy/crd.yaml
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.
customresourcedefinition.apiextensions.k8s.io/netperfs.app.example.com created
$
$ kubectl apply -f https://raw.githubusercontent.com/piontec/netperf-operator/master/deploy/cr.yaml
netperf.app.example.com/example created
$
$ kubectl apply -f https://raw.githubusercontent.com/piontec/netperf-operator/master/deploy/rbac.yaml
Warning: rbac.authorization.k8s.io/v1beta1 Role is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 Role
role.rbac.authorization.k8s.io/netperf-operator created
Warning: rbac.authorization.k8s.io/v1beta1 RoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/
rolebinding.rbac.authorization.k8s.io/default-account-netperf-operator created
$ 
$ kubectl apply -f https://raw.githubusercontent.com/piontec/netperf-operator/master/deploy/operator.yaml
deployment.apps/netperf-operator created
```
let`s check
```
$ kubectl describe netperf.app.example.com/example
Name:         example
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  app.example.com/v1alpha1
Kind:         Netperf
Metadata:
  Creation Timestamp:  2022-04-14T07:48:43Z
  Generation:          4
  Managed Fields:
    API Version:  app.example.com/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2022-04-14T07:48:43Z
    API Version:  app.example.com/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        .:
        f:clientNode:
        f:serverNode:
      f:status:
        .:
        f:clientPod:
        f:serverPod:
        f:speedBitsPerSec:
        f:status:
    Manager:         netperf-operator
    Operation:       Update
    Time:            2022-04-14T07:49:30Z
  Resource Version:  28389
  UID:               529d3539-abcb-4c02-bf5a-4a3f5620cb52
Spec:
  Client Node:  
  Server Node:  
Status:
  Client Pod:          netperf-client-4a3f5620cb52
  Server Pod:          netperf-server-4a3f5620cb52
  Speed Bits Per Sec:  1784.46
  Status:              Done
Events:                <none>
```
результат:
Speed Bits Per Sec:  1784.46
далее по мотедичке
```
$ kubectl apply -f ./kit/netperf-calico-policy.yaml 
networkpolicy.crd.projectcalico.org/netperf-calico-policy created
```
запускаем тест повторно... хм
    Speed Bits Per Sec:  1856.24
    Status:              Done

видимо в нашей сетевой политике нет ошибок ;-)
и на ноде счётчики iptabes по нулям и на воркере тоже
```
gke-hw-iptables-default-pool-e60ac019-vc5z ~ # journalctl -k | grep calico
gke-hw-iptables-default-pool-e60ac019-vc5z ~ #
```

ладно, принцип понятен, приступаем непосредственно к iptables-tailer
применяем манифесты по ds, смотрим что получилось:
```
$ kubectl describe ds kube-iptables-tailer -n kube-system
...
  Normal   SuccessfulCreate  14s                     daemonset-controller  Created pod: kube-iptables-tailer-4ldmk
  Normal   SuccessfulCreate  14s                     daemonset-controller  Created pod: kube-iptables-tailer-w6w5q
  Normal   SuccessfulCreate  14s                     daemonset-controller  Created pod: kube-iptables-tailer-6p2x2
```
запускаем тест, работает
```
  Speed Bits Per Sec:  1867.09
  Status:              Done
```
никаких дропов в логах
```
$ kubectl describe pod netperf-operator
Events:                      <none>
```
всё доступно и тесты проходят, и networkpolicy имеется
```
$ kubectl get networkpolicy.crd.projectcalico.org/netperf-calico-policy
NAME                    AGE
netperf-calico-policy   130m
```
хотя и calico networkpolicy стоят из манифеста с селектором app == "netperf-operator",
и в момент теста поды имеют соответствующий 
```
$ kubectl describe pod netperf-server-fc3eeea4d1f8
Labels:       app=netperf-operator
```
всё равно
  Speed Bits Per Sec:  1874.23
  Status:              Done

в целом понятно, куда и как смотреть, если не работает ;-)
