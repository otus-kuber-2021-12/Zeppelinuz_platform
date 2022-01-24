## HW-5 kubernetes-volumes

Создал новый кластер в kind
по инструкции применил два манифеста
результат:
```
# kubectl get statefulsets
NAME    READY   AGE
minio   1/1     5m22s


# kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
minio-0   1/1     Running   0          5m34s


# kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-minio-0   Bound    pvc-b7ef9849-e990-415c-a804-68a94d05876a   10Gi       RWO            standard       5m49s


# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pvc-b7ef9849-e990-415c-a804-68a94d05876a   10Gi       RWO            Delete           Bound    default/data-minio-0   standard                6m12s


# kubectl describe pv/pvc-b7ef9849-e990-415c-a804-68a94d05876a
Name:              pvc-b7ef9849-e990-415c-a804-68a94d05876a
Labels:            <none>
Annotations:       pv.kubernetes.io/provisioned-by: rancher.io/local-path
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      standard
Status:            Bound
Claim:             default/data-minio-0
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          10Gi
Node Affinity:
  Required Terms:
    Term 0:        kubernetes.io/hostname in [kind-control-plane]
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /var/local-path-provisioner/pvc-b7ef9849-e990-415c-a804-68a94d05876a_default_data-minio-0
    HostPathType:  DirectoryOrCreate
Events:            <none>

```
# Задание со *
зашифровать
```
- name: MINIO_ACCESS_KEY
    гalue: "minio"
- name: MINIO_SECRET_KEY
    value: "minio123"
```
encoding:
```
# echo -n minio | base64
bWluaW8=
# echo -n minio123 | base64
bWluaW8xMjM=
```
создал и применил манифест kind: Secret (secret.yaml)
```
# kubectl apply -f ./secret.yaml
secret/minio created
```
внёс изменения в манифест StatefulSet, применил, проверка:
```
# kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-xgxtn   kubernetes.io/service-account-token   3      43m
minio                 Opaque                                2      8m26s


# kubectl describe secret minio
Name:         minio
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  8 bytes
username:  5 bytes


# kubectl get secret minio -o yaml
apiVersion: v1
data:
  password: bWluaW8xMjM=
  username: bWluaW8=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"bWluaW8xMjM=","username":"bWluaW8="},"kind":"Secret","metadata":{"annotations":{},"name":"minio","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2022-01-24T11:13:51Z"
  name: minio
  namespace: default
  resourceVersion: "4178"
  uid: d7b121bf-c3fa-4c69-947e-71a594c5bf88
type: Opaque


# kubectl describe statefulsets minio
Name:               minio
Namespace:          default
CreationTimestamp:  Mon, 24 Jan 2022 16:39:35 +0600
Selector:           app=minio
Labels:             <none>
Annotations:        <none>
Replicas:           1 desired | 1 total
Update Strategy:    RollingUpdate
  Partition:        0
Pods Status:        1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=minio
  Containers:
   minio:
    Image:      minio/minio:RELEASE.2019-07-10T00-34-56Z
    Port:       9000/TCP
    Host Port:  0/TCP
    Args:
      server
      /data
    Liveness:  http-get http://:9000/minio/health/live delay=120s timeout=1s period=20s #success=1 #failure=3
    Environment:
      MINIO_ACCESS_KEY:  <set to the key 'username' in secret 'minio'>  Optional: false
      MINIO_SECRET_KEY:  <set to the key 'password' in secret 'minio'>  Optional: false
    Mounts:
      /data from data (rw)
  Volumes:  <none>
Volume Claims:
  Name:          data
  StorageClass:
  Labels:        <none>
  Annotations:   <none>
  Capacity:      10Gi
  Access Modes:  [ReadWriteOnce]
Events:
  Type    Reason            Age                From                    Message
  ----    ------            ----               ----                    -------
  Normal  SuccessfulCreate  46m                statefulset-controller  create Claim data-minio-0 Pod minio-0 in StatefulSet minio success
  Normal  SuccessfulDelete  11m                statefulset-controller  delete Pod minio-0 in StatefulSet minio successful
  Normal  SuccessfulCreate  11m (x2 over 46m)  statefulset-controller  create Pod minio-0 in StatefulSet minio successful
```






















