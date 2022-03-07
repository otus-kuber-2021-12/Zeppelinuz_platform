### kubernetes-gitops

поставил istio
```
$ helm repo add istio https://istio-release.storage.googleapis.com/charts
"istio" has been added to your repositories
$
$ kubectl create namespace istio-system
namespace/istio-system created
$
$ helm install istio-base istio/base -n istio-system
NAME: istio-base
LAST DEPLOYED: Tue Mar  1 15:22:32 2022
NAMESPACE: istio-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Istio base successfully installed!
$
$ helm install istiod istio/istiod -n istio-system --wait
W0301 15:23:21.243410  415274 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
W0301 15:23:27.486801  415274 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
NAME: istiod
LAST DEPLOYED: Tue Mar  1 15:23:19 2022
NAMESPACE: istio-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
"istiod" successfully installed!
$
$ helm install istio-ingress istio/gateway -n istio-ingress --wait
NAME: istio-ingress
LAST DEPLOYED: Tue Mar  1 15:25:34 2022
NAMESPACE: istio-ingress
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
"istio-ingress" successfully installed!



```
собрал и добавил в dockerhub микросервисы
готовимся дальше
```
$ helm repo add fluxcd https://charts.fluxcd.io
"fluxcd" has been added to your repositories
$
$ helm upgrade --install flux fluxcd/flux -f ./flux.values.yaml -n flux --create-namespace
Release "flux" does not exist. Installing it now.
NAME: flux
LAST DEPLOYED: Wed Mar  2 15:17:43 2022
NAMESPACE: flux
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Get the Git deploy key by either (a) running
$
$ helm upgrade --install helm-operator fluxcd/helm-operator -f ./helm-operator.values.yaml -n flux
Release "helm-operator" does not exist. Installing it now.
NAME: helm-operator
LAST DEPLOYED: Wed Mar  2 15:25:15 2022
NAMESPACE: flux
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Flux Helm Operator docs https://fluxcd.io/legacy/helm-operator
$
```
fluxctl
```
$ snap install fluxctl --classic
fluxctl 1.24.3 от Flux CD developers (weaveflux) установлен
$
$ export FLUX_FORWARD_NAMESPACE=flux
$ fluxctl list-workloads
WORKLOAD  CONTAINER  IMAGE  RELEASE  POLICY
$ fluxctl identity
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC14kaype/e00ujR5tmQRUB+4je8/98llgOlMFa0s8bpYrQyu9qgHvKmSKPHMWkIgEIlWxVuZVxwy92jLi7Xe5T0MjJRuNgKD3CDnBrz+o4F6kvzcgcOjbm/NuSukRK3xK4BQRRjGPb2...
```
добавил ключ в gitlab SSH Keys
```
curl -s https://fluxcd.io/install.sh | sudo bash
flux bootstrap gitlab --owner=otushwo --repository=namesace-test --token-auth
```
запускаем, проверяем
```
$ kubectl get helmreleases.helm.fluxcd.io -A
NAMESPACE            NAME       RELEASE    PHASE       RELEASESTATUS   MESSAGE                                                                       AGE
microservices-demo   frontend   frontend   Succeeded   deployed        Release was successful for Helm release 'frontend' in 'microservices-demo'.   86m
$
$ helm list -A
NAME            NAMESPACE               REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
flux            flux                    2               2022-03-02 16:37:58.988989012 +0600 +06 deployed        flux-1.11.4             1.24.3     
frontend        microservices-demo      1               2022-03-03 11:12:30.816047416 +0000 UTC deployed        frontend-0.21.0         1.16.0     
helm-operator   flux                    1               2022-03-02 15:25:15.693849032 +0600 +06 deployed        helm-operator-1.4.2     1.4.2      
```
обновил image в dockerhub, pod обновился самостоятельно
```
$ helm list -A
NAME            NAMESPACE               REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
flux            flux                    2               2022-03-02 16:37:58.988989012 +0600 +06 deployed        flux-1.11.4             1.24.3     
frontend        microservices-demo      2               2022-03-03 11:38:25.839054254 +0000 UTC deployed        frontend-0.21.0         1.16.0     
helm-operator   flux                    1               2022-03-02 15:25:15.693849032 +0600 +06 deployed        helm-operator-1.4.2     1.4.2 
```
часть лога ( $ kubectl logs -f helm-operator-f7b7544c4-q5d98 -n flux )
```
ts=2022-03-03T11:38:25.164889497Z caller=release.go:79 component=release release=frontend targetNamespace=microservices-demo resource=microservices-demo:helmrelease/frontend helmVersion=v3 info="starting sync run"
ts=2022-03-03T11:38:25.376497726Z caller=release.go:353 component=release release=frontend targetNamespace=microservices-demo resource=microservices-demo:helmrelease/frontend helmVersion=v3 info="running upgrade" action=upgrade
ts=2022-03-03T11:38:25.398932981Z caller=helm.go:69 component=helm version=v3 info="preparing upgrade for frontend" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:25.405473336Z caller=helm.go:69 component=helm version=v3 info="resetting values to the chart's original version" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:26.034385296Z caller=helm.go:69 component=helm version=v3 info="performing update for frontend" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:26.052944165Z caller=helm.go:69 component=helm version=v3 info="creating upgraded release for frontend" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:26.067964685Z caller=helm.go:69 component=helm version=v3 info="checking 4 resources for changes" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:26.080723935Z caller=helm.go:69 component=helm version=v3 info="Looks like there are no changes for Service \"frontend\"" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:26.291464967Z caller=helm.go:69 component=helm version=v3 info="updating status for upgraded release for frontend" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:26.33056986Z caller=release.go:364 component=release release=frontend targetNamespace=microservices-demo resource=microservices-demo:helmrelease/frontend helmVersion=v3 info="upgrade succeeded" revision=eedf23ac4086fb334d9e21a4c8076212b9f085cd phase=upgrade
ts=2022-03-03T11:38:27.801254489Z caller=release.go:79 component=release release=frontend targetNamespace=microservices-demo resource=microservices-demo:helmrelease/frontend helmVersion=v3 info="starting sync run"
ts=2022-03-03T11:38:28.00170512Z caller=release.go:289 component=release release=frontend targetNamespace=microservices-demo resource=microservices-demo:helmrelease/frontend helmVersion=v3 info="running dry-run upgrade to compare with release version '2'" action=dry-run-compare
ts=2022-03-03T11:38:28.007815206Z caller=helm.go:69 component=helm version=v3 info="preparing upgrade for frontend" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:28.014183376Z caller=helm.go:69 component=helm version=v3 info="resetting values to the chart's original version" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:28.679527154Z caller=helm.go:69 component=helm version=v3 info="performing update for frontend" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:28.697194975Z caller=helm.go:69 component=helm version=v3 info="dry run for frontend" targetNamespace=microservices-demo release=frontend
ts=2022-03-03T11:38:28.802861661Z caller=release.go:311 component=release release=frontend targetNamespace=microservices-demo resource=microservices-demo:helmrelease/frontend helmVersion=v3 info="no changes" phase=dry-run-compare
```
и ещё
```
$ helm history frontend -n microservices-demo
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
1               Thu Mar  3 11:12:30 2022        superseded      frontend-0.21.0 1.16.0          Install complete
2               Thu Mar  3 11:38:25 2022        deployed        frontend-0.21.0 1.16.0          Upgrade complete
```
Манифест в gitlab обновился.
Аналогично растолкал остальные сервисы.
```
$ kubectl get pod -n microservices-demo
NAME                                     READY   STATUS     RESTARTS   AGE
adservice-6dc97c5c96-bc2rk               1/1     Running    0          33m
cartservice-7485464bf7-x4ptx             1/1     Running    0          14m
cartservice-redis-master-0               1/1     Running    0          14m
checkoutservice-564d4ff8c6-kh99c         1/1     Running    0          41m
currencyservice-f7565ff4d-27vqc          1/1     Running    0          41m
emailservice-6cd7b74b8b-v5788            1/1     Running    0          45m
frontend-57b9977d4-rl65c                 1/1     Running    0          70m
loadgenerator-79d4dd6894-zwrfz           0/1     Init:0/1   0          74s
paymentservice-9955c8cc8-4j8md           1/1     Running    0          26m
productcatalogservice-5748496564-ldgwq   1/1     Running    0          26m
recommendationservice-59f76976bd-rmbmq   1/1     Running    0          23m
shippingservice-694bfdb7bd-wnf5h         1/1     Running    0          23m
$
$ helm list -n microservices-demo
NAME                    NAMESPACE               REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
adservice               microservices-demo      1               2022-03-03 12:14:49.108418563 +0000 UTC deployed        adservice-0.5.0                 1.16.0     
cartservice             microservices-demo      1               2022-03-03 12:34:24.444286349 +0000 UTC deployed        cartservice-0.4.1               1.16.0     
checkoutservice         microservices-demo      1               2022-03-03 12:07:45.401145524 +0000 UTC deployed        checkoutservice-0.4.0           1.16.0     
currencyservice         microservices-demo      1               2022-03-03 12:07:45.500756968 +0000 UTC deployed        currencyservice-0.4.0           1.16.0     
emailservice            microservices-demo      1               2022-03-03 12:03:42.262259799 +0000 UTC deployed        emailservice-0.4.0              1.16.0     
frontend                microservices-demo      2               2022-03-03 11:38:25.839054254 +0000 UTC deployed        frontend-0.21.0                 1.16.0     
loadgenerator           microservices-demo      1               2022-03-03 12:47:34.223565333 +0000 UTC deployed        loadgenerator-0.4.0             1.16.0     
paymentservice          microservices-demo      1               2022-03-03 12:21:53.874228495 +0000 UTC deployed        paymentservice-0.3.0            1.16.0     
productcatalogservice   microservices-demo      1               2022-03-03 12:21:54.003921451 +0000 UTC deployed        productcatalogservice-0.3.0     1.16.0     
recommendationservice   microservices-demo      1               2022-03-03 12:24:55.848982156 +0000 UTC deployed        recommendationservice-0.3.0     1.16.0     
shippingservice         microservices-demo      1               2022-03-03 12:24:55.827895808 +0000 UTC deployed        shippingservice-0.3.0           1.16.0     
```
#### Flagger & Istio
```
$ istioctl install -y
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Installation complete                                                                                                                                                         Making this installation the default for injection and validation.
Making this installation the default for injection and validation.

Thank you for installing Istio 1.13.
$
$ kubectl label namespace microservices-demo istio-injection=enabled
namespace/microservices-demo labeled
$ kubectl get pod -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-5957d56d94-s6q8l   1/1     Running   0          2m14s
istiod-68fb4f5c54-zfsz8                 1/1     Running   0          2m27s
```
flagger
```
$ helm repo add flagger https://flagger.app
"flagger" has been added to your repositories
$
$ kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/artifacts/flagger/crd.yaml
customresourcedefinition.apiextensions.k8s.io/canaries.flagger.app created
customresourcedefinition.apiextensions.k8s.io/metrictemplates.flagger.app created
customresourcedefinition.apiextensions.k8s.io/alertproviders.flagger.app created
```
далее по методичке
```
$ helm upgrade --install flagger flagger/flagger --namespace=istio-system --set crd.create=false --set=meshProvider=istio --set metricsServer=http://prometheus:9090
Release "flagger" does not exist. Installing it now.
NAME: flagger
LAST DEPLOYED: Fri Mar  4 14:16:08 2022
NAMESPACE: istio-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Flagger installed
```
какой ещё prometheus?
едем дальше по методичке
```
$ kubectl describe namespaces microservices-demo | grep istio
              istio-injection=enabled
```
сайдкар запускается, далее Gateway & VirtualService в istio для frontend-а
```
$ kubectl get gateway -n microservices-demo
NAME               AGE
frontend           41s
frontend-gateway   21h
```
попытаемся открыть http://104.196.216.196/ (боюсь к окончанию работы над ДЗ баланс бесплатного GKE закончится, 17$ осталось)
вуа-ля - Free shipping with $75 purchase!
папку deploy/istio удалил, перезапустил, работает вроде
#### Flagger | Canary
... по методичке, ну и куда же без этого
```
kubectl logs helm-operator-f7b7544c4-j8l2j -n flux -f | grep Сanary
...
ts=2022-03-06T11:36:20.913939765Z caller=release.go:357 component=release release=frontend targetNamespace=microservices-demo resource=microservices-demo:helmrelease/frontend helmVersion=v3 error="upgrade failed: unable to recognize \"\": no matches for kind \"Canary\" in version \"flagger.app/v1alpha3\"" action=upgrade
```
изменил манифест под требования новой версии, завелось
```
$ kubectl get canary -n microservices-demo
NAME       STATUS        WEIGHT   LASTTRANSITIONTIME
frontend   Initialized   0        2022-03-06T12:51:52Z
$
$ kubectl get pods -n microservices-demo -l app=frontend-primary
NAME                              READY   STATUS    RESTARTS   AGE
frontend-primary-874f58bc-prw54   2/2     Running   0          10m
$
```
после обновления образа
```
$ kubectl get canaries -n microservices-demo
NAME               STATUS      WEIGHT   LASTTRANSITIONTIME
frontend           Succeeded   0        2022-03-26T12:59:09Z
$
```
###### для ДЗ
ссылка на репозитарий - https://gitlab.com/otushwo/microservices-demo
```
$ kubectl get canaries -n microservices-demo
NAME               STATUS      WEIGHT   LASTTRANSITIONTIME
frontend           Succeeded   0        2022-03-26T12:59:09Z
```
```
kubectl describe canary frontend -n microservices-demo
...
Events:
  Type     Reason  Age                    From     Message
  ----     ------  ----                   ----     -------
  Normal   Synced  9m28s (x2 over 22m)    flagger  New revision detected! Scaling up frontend.microservices-demo
  Normal   Synced  7m58s                  flagger  New revision detected! Restarting analysis for frontend.microservices-demo
  Normal   Synced  7m15s (x3 over 22m)    flagger  Advance frontend.microservices-demo canary weight 5
  Normal   Synced  7m15s (x3 over 22m)    flagger  Starting canary analysis for frontend.microservices-demo
  Normal   Synced  6m44s (x2 over 7m13s)  flagger  Advance frontend.microservices-demo canary weight 10
  Normal   Synced  6m15s                  flagger  Advance frontend.microservices-demo canary weight 15
  Normal   Synced  4m43s                  flagger  Advance frontend.microservices-demo canary weight 20
  Normal   Synced  4m15s                  flagger  Advance frontend.microservices-demo canary weight 25
  Normal   Synced  3m43s                  flagger  Advance frontend.microservices-demo canary weight 30
  Normal   Synced  98s (x3 over 2m24s)    flagger  (combined from similar events): Promotion completed! Scaling down frontend.microservices-demo
```

