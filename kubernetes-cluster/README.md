## kubernetes-cluster
по методичке
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt install docker docker.io -y
apt-get install -y kubelet=1.17.4-00
apt-get install -y kubeadm=1.17.4-00
apt-get install -y kubectl=1.17.4-00
kubeadm init --pod-network-cidr=192.168.0.0/24
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
++++
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.128.0.18:6443 --token lce7zq.eyn2wdan6mbdb0ya \
--discovery-token-ca-cert-hash sha256:dcd8132aa369318611920ee045851eda0df328f8213d5db943c739a092a9cadf 
++++
```
результат
```
root@instance-2:~# kubectl get nodes -o wide
NAME         STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
instance-2   Ready    master   25m   v1.17.4   10.128.0.18   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   docker://20.10.7
worker-1     Ready    <none>   55s   v1.17.4   10.128.0.19   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   docker://20.10.7
worker-2     Ready    <none>   26s   v1.17.4   10.128.0.20   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   docker://20.10.7
worker-3     Ready    <none>   24s   v1.17.4   10.128.0.21   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   docker://20.10.7
```
нагружаем
```
root@instance-2:~# kubectl apply -f ./depl.yaml 
root@instance-2:~# kubectl get pod -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-c8fd555cc-db4fm   1/1     Running   0          24s   192.168.0.129   worker-3   <none>           <none>
nginx-deployment-c8fd555cc-hb5qp   1/1     Running   0          24s   192.168.0.194   worker-2   <none>           <none>
nginx-deployment-c8fd555cc-qgppv   1/1     Running   0          24s   192.168.0.65    worker-1   <none>           <none>
nginx-deployment-c8fd555cc-rztp2   1/1     Running   0          24s   192.168.0.193   worker-2   <none>           <none>
```
обновляем
```
apt-get update && apt-get install -y kubeadm=1.18.0-00 kubelet=1.18.0-00
```
```
root@instance-2:~# kubectl get nodes -A
NAME         STATUS   ROLES    AGE   VERSION
instance-2   Ready    master   35m   v1.18.0
worker-1     Ready    <none>   11m   v1.17.4
worker-2     Ready    <none>   10m   v1.17.4
worker-3     Ready    <none>   10m   v1.17.4
```
версии:
```
root@instance-2:~# kubectl describe pod kube-apiserver-instance-2 -n kube-system
Containers:
  kube-apiserver:
    Image:         k8s.gcr.io/kube-apiserver:v1.17.17
```
```
root@instance-2:~# kubelet --version
Kubernetes v1.18.0
```
смотрим план обновления
```
root@instance-2:~# kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.17.17
[upgrade/versions] kubeadm version: v1.18.0
I0415 10:07:49.622970   16986 version.go:252] remote version is much newer: v1.23.5; falling back to: stable-1.18
[upgrade/versions] Latest stable version: v1.18.20
[upgrade/versions] Latest stable version: v1.18.20
[upgrade/versions] Latest version in the v1.17 series: v1.17.17
[upgrade/versions] Latest version in the v1.17 series: v1.17.17

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     3 x v1.17.4   v1.18.20
            1 x v1.18.0   v1.18.20

Upgrade to the latest stable version:

COMPONENT            CURRENT    AVAILABLE
API Server           v1.17.17   v1.18.20
Controller Manager   v1.17.17   v1.18.20
Scheduler            v1.17.17   v1.18.20
Kube Proxy           v1.17.17   v1.18.20
CoreDNS              1.6.5      1.6.7
Etcd                 3.4.3      3.4.3-0

You can now apply the upgrade by executing the following command:
        kubeadm upgrade apply v1.18.20
Note: Before you can perform this upgrade, you have to update kubeadm to v1.18.20.
```
обновляем
```
root@instance-2:~# kubeadm upgrade apply v1.18.0
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.18.0". Enjoy!
```
```
root@instance-2:~# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:56:30Z", GoVersion:
```
```
root@instance-2:~# kubectl describe pod kube-apiserver-instance-2 -n kube-system
Containers:
  kube-apiserver:
    Image:         k8s.gcr.io/kube-apiserver:v1.18.0
```
готово, теперь воркера(ов)
```
root@instance-2:~# kubectl drain worker-1 --ignore-daemonsets
node/worker-1 already cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-2j8jr, kube-system/kube-proxy-hsp7k
evicting pod kube-system/coredns-66bff467f8-flwph
evicting pod default/nginx-deployment-c8fd555cc-qgppv
pod/nginx-deployment-c8fd555cc-qgppv evicted
pod/coredns-66bff467f8-flwph evicted
node/worker-1 drained
```
```
root@instance-2:~# kubectl get node -o wide
NAME         STATUS                     ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
instance-2   Ready                      master   54m   v1.18.0   10.128.0.18   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   docker://20.10.7
worker-1     Ready,SchedulingDisabled   <none>   29m   v1.17.4   10.128.0.19   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   docker://20.10.7
worker-2     Ready                      <none>   29m   v1.17.4   10.128.0.20   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   docker://20.10.7
worker-3     Ready                      <none>   29m   v1.17.4   10.128.0.21   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   docker://20.10.7
```
далее на самом воркере
```
root@worker-1:~# apt-get install -y kubelet=1.18.0-00 kubeadm=1.18.0-00
root@worker-1:~# systemctl restart kubelet
root@worker-1:~#
```
обновился:
```
root@instance-2:~# kubectl get node
NAME         STATUS                     ROLES    AGE   VERSION
instance-2   Ready                      master   56m   v1.18.0
worker-1     Ready,SchedulingDisabled   <none>   31m   v1.18.0
worker-2     Ready                      <none>   31m   v1.17.4
worker-3     Ready                      <none>   31m   v1.17.4
```
можно возвращать в строй
```
root@instance-2:~# kubectl uncordon worker-1
node/worker-1 uncordoned
root@instance-2:~# kubectl get node
NAME         STATUS   ROLES    AGE   VERSION
instance-2   Ready    master   57m   v1.18.0
worker-1     Ready    <none>   32m   v1.18.0
worker-2     Ready    <none>   32m   v1.17.4
worker-3     Ready    <none>   32m   v1.17.4
```
...аналогично оставшиеся...
```
root@instance-2:~# kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
instance-2   Ready    master   61m   v1.18.0
worker-1     Ready    <none>   36m   v1.18.0
worker-2     Ready    <none>   36m   v1.18.0
worker-3     Ready    <none>   36m   v1.18.0
```
## Автоматическое развертывание кластеров

по инструкции, пришлось повозиться с GKP обходя ограничения бесплатного
использования... после работы kubesprey-а чуть более одного часа, результат:
```
root@node1:~# kubectl get nodes -o wide
NAME    STATUS   ROLES                  AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
node1   Ready    control-plane,master   121m   v1.23.5   10.128.0.28   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   containerd://1.6.2
node2   Ready    <none>                 119m   v1.23.5   10.128.0.29   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   containerd://1.6.2
node3   Ready    <none>                 119m   v1.23.5   10.128.0.30   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   containerd://1.6.2
node4   Ready    <none>                 119m   v1.23.5   10.128.0.31   <none>        Ubuntu 18.04.6 LTS   5.4.0-1069-gcp   containerd://1.6.2
```
