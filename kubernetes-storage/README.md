#### kubernetes-storage
создал кластер
```
# kind create cluster --config ./cluster.yaml
enabling experimental podman provider
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.23.4) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
```
установил csi-driver-host-path
```
# git clone https://github.com/kubernetes-csi/csi-driver-host-path
...
# cd ./csi-driver-host-path/
# ./deploy/kubernetes-latest/deploy.sh
```
далее применил манифесты для создания sc, pvc с именем storage-pvc и pod с именем storage-pod
##### проверяем

```
# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
pvc-ac88b40e-c2b1-49db-a3db-9b2694c617a9   100Mi      RWO            Delete           Bound    default/storage-pvc   csi-hw-sc               3m27s
```
```
# kubectl get pvc
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
storage-pvc   Bound    pvc-ac88b40e-c2b1-49db-a3db-9b2694c617a9   100Mi      RWO            csi-hw-sc      3m54s
```
```
# kubectl describe pod storage-pod
Name:         storage-pod
Namespace:    default
...
    Mounts:
      /data from csi-hw-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-b8lqf (ro)
...
Volumes:
  csi-hw-volume:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  storage-pvc
    ReadOnly:   false
...
Events:
  Type     Reason                  Age                    From                     Message
  ----     ------                  ----                   ----                     -------
  Normal   SuccessfulAttachVolume  4m30s                  attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-ac88b40e-c2b1-49db-a3db-9b2694c617a9"
  ```
  

