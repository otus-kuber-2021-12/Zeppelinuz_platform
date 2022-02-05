## kubernetes-monitoring

попробую в minikube на виртуалке под VMware Player

```
$ helm version
version.BuildInfo{Version:"v3.8.0", GitCommit:"d14138609b01886f544b2025f5000351c9eb092e", GitTreeState:"clean", GoVersion:"go1.17.5"}
$
$ minikube start
* minikube v1.25.1 on Centos 8.5.2111
* Using the podman driver based on existing profile
* Starting control plane node minikube in cluster minikube
* Pulling base image ...
E0205 16:29:32.717070    1717 cache.go:203] Error downloading kic artifacts:  no                                                             t yet implemented, see issue #8426
* Restarting existing podman container for "minikube" ...
* Preparing Kubernetes v1.23.1 on Docker 20.10.12 ...
  - kubelet.housekeeping-interval=5m
  - Generating certificates and keys ...
  - Booting up control plane ...
  - Configuring RBAC rules ...
* Verifying Kubernetes components...
  - Using image gcr.io/k8s-minikube/storage-provisioner:v5
* Enabled addons: storage-provisioner, default-storageclass
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```
создал ns и helm-ом развернул prometheus
```
$ kubectl create namespace monitoring
namespace/monitoring created
$ 
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories
$ 
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
$
$ helm install prome-1 prometheus-community/kube-prometheus-stack -n monitoring
NAME: prome-1
LAST DEPLOYED: Sat Feb  5 16:30:53 2022
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
```
проверил, что там
```
$ kubectl get pod -n monitoring
NAME                                                   READY   STATUS              RESTARTS   AGE
prome-1-grafana-b64bf88d4-x47c9                        0/3     ContainerCreating   0          57s
prome-1-kube-prometheus-st-operator-8669687f45-w7gcj   0/1     ContainerCreating   0          57s
prome-1-kube-state-metrics-544946b867-272lh            0/1     ContainerCreating   0          57s
prome-1-prometheus-node-exporter-ghz59                 1/1     Running             0          57s
```
нарисовал четыре манифеста и применил их
```
$ kubectl apply -f ./nginx-ConfigMap.yaml
configmap/nginx-config created
$ kubectl apply -f ./nginx-Deployment.yaml
deployment.apps/nginx created
$ kubectl apply -f ./nginx-nginx-Service.yaml
service/nginx created
$ kubectl apply -f ./nginx-ServiceMonitor.yaml
servicemonitor.monitoring.coreos.com/nginx-sm created
$
$
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-54cc5ddf85-lx6qq   2/2     Running   0          14m
nginx-54cc5ddf85-mj55v   2/2     Running   0          14m
nginx-54cc5ddf85-vgf7k   2/2     Running   0          14m
```
чего там по сервисам:
```
$ kubectl get services -n monitoring
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   22m
prome-1-grafana                           ClusterIP   10.105.224.241   <none>        80/TCP                       23m
prome-1-kube-prometheus-st-alertmanager   ClusterIP   10.107.25.16     <none>        9093/TCP                     23m
prome-1-kube-prometheus-st-operator       ClusterIP   10.96.254.135    <none>        443/TCP                      23m
prome-1-kube-prometheus-st-prometheus     ClusterIP   10.109.99.161    <none>        9090/TCP                     23m
prome-1-kube-state-metrics                ClusterIP   10.110.64.89     <none>        8080/TCP                     23m
prome-1-prometheus-node-exporter          ClusterIP   10.99.168.107    <none>        9100/TCP                     23m
prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     22m
```
проверим, глянем
> в отдельном окне
$ kubectl port-forward service/prome-1-kube-prometheus-st-prometheus -n monitoring 9090:9090

```
$ curl http://127.0.0.1:9090/targets
<!doctype html><html lang="en"><head><meta charset="utf-8"/><link rel="shortcut icon" href="./favicon.ico"/><meta name="viewport" content="width=device-width,initial-scale=1,shrink-to-fit=no"/><meta name="theme-color" content="#000000"/><script>const GLOBAL_CONSOLES_LINK="",GLOBAL_AGENT_MODE="false"</script><link rel="manifest" href="./manifest.json" crossorigin="use-credentials"/><title>Prometheus Time Series Collection and Processing Server</title><link href="./static/css/2.cede384b.chunk.css" rel="stylesheet"><link href="./static/css/main.ac88b532.chunk.css" rel="stylesheet"></head><body class="bootstrap"><noscript>You need to enable JavaScript to run this app.</noscript><div id="root"></div><script>!function(e){function r(r){for(var n,l,a=r[0],p=r[1],f=r[2],c=0,s=[];c<a.length;c++)l=a[c],Object.prototype.hasOwnProperty.call(o,l)&&o[l]&&s.push(o[l][0]),o[l]=0;for(n in p)Object.prototype.hasOwnProperty.call(p,n)&&(e[n]=p[n]);for(i&&i(r);s.length;)s.shift()();return u.push.apply(u,f||[]),t()}function t(){for(var e,r=0;r<u.length;r++){for(var t=u[r],n=!0,a=1;a<t.length;a++){var p=t[a];0!==o[p]&&(n=!1)}n&&(u.splice(r--,1),e=l(l.s=t[0]))}return e}var n={},o={1:0},u=[];function l(r){if(n[r])return n[r].exports;var t=n[r]={i:r,l:!1,exports:{}};return e[r].call(t.exports,t,t.exports,l),t.l=!0,t.exports}l.m=e,l.c=n,l.d=function(e,r,t){l.o(e,r)||Object.defineProperty(e,r,{enumerable:!0,get:t})},l.r=function(e){"undefined"!=typeof Symbol&&Symbol.toStringTag&&Object.defineProperty(e,Symbol.toStringTag,{value:"Module"}),Object.defineProperty(e,"__esModule",{value:!0})},l.t=function(e,r){if(1&r&&(e=l(e)),8&r)return e;if(4&r&&"object"==typeof e&&e&&e.__esModule)return e;var t=Object.create(null);if(l.r(t),Object.defineProperty(t,"default",{enumerable:!0,value:e}),2&r&&"string"!=typeof e)for(var n in e)l.d(t,n,function(r){return e[r]}.bind(null,n));return t},l.n=function(e){var r=e&&e.__esModule?function(){return e.default}:function(){return e};return l.d(r,"a",r),r},l.o=function(e,r){return Object.prototype.hasOwnProperty.call(e,r)},l.p="./";var a=this.webpackJsonpgraph=this.webpackJsonpgraph||[],p=a.push.bind(a);a.push=r,a=a.slice();for(var f=0;f<a.length;f++)r(a[f]);var i=p;t()}([])</script><script src="./static/js/2.75f1d0f1.chunk.js"></script><script src="./static/js/main.a00f3aa4.chunk.js"></script></body></html>
```
и grafana
```
$ kubectl port-forward service/prome-1-grafana -n monitoring 8000:80
Forwarding from 127.0.0.1:8000 -> 3000
Forwarding from [::1]:8000 -> 3000
Handling connection for 8000
```
```
$ curl http://127.0.0.1:8000/
<a href="/login">Found</a>.
```

p.s. Wolfenstein 3D 4eva
