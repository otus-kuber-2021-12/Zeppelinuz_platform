## kubernetes-operators

на виртуалке под VMware Player, minikube v1.25.1 on Centos 8.5.2111

по методичке, до страницы момента создания CRD, при попытке ошибка:
```
$ kubectl apply -f bacis_crd.yml
error: unable to recognize "bacis_crd.yml": no matches for kind "CustomResourceDefinition" in version "apiextensions.k8s.io/v1beta1"
```
поменял версию, получил другую ошибку:
```
$ kubectl apply -f bacis_crd.yml
The CustomResourceDefinition "mysqls.otus.homework" is invalid: spec.versions[0].schema.openAPIV3Schema: Required value: schemas are required
```
дорисовал что требуется, создалось (закоментировав usless_data в cr.yaml, а то ругался: error: error validating "./cr.yaml": error validating data: ValidationError(MySQL): unknown field "usless_data" in homework.otus.v1.MySQL):
```
$ kubectl apply -f ./crd.yaml
customresourcedefinition.apiextensions.k8s.io/mysqls.otus.homework created
$
$ kubectl get crd -A
NAME                   CREATED AT
mysqls.otus.homework   2022-02-12T10:29:16Z
$
$ kubectl apply -f ./cr.yaml
mysql.otus.homework/mysql-instance created
```
чего там есть:
```
$ kubectl get mysqls.otus.homework
NAME             AGE
mysql-instance   2m39s
$
$ kubectl describe mysqls.otus.homework mysql-instance
Name:         mysql-instance
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  otus.homework/v1
Kind:         MySQL
Metadata:
  Creation Timestamp:  2022-02-12T10:30:54Z
  Generation:          1
  Managed Fields:
    API Version:  otus.homework/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
    Manager:         kubectl-client-side-apply
    Operation:       Update
    Time:            2022-02-12T10:30:54Z
  Resource Version:  13828
  UID:               cfeaa7ab-0eba-4b83-9b49-794930aa88d5
Spec:
Events:  <none>
```
#### Validation
```
$ kubectl delete mysqls.otus.homework mysql-instance
mysql.otus.homework "mysql-instance" deleted
```
добавил Validation по методичке, применяю - ошибка:
```
$ kubectl apply -f ./crd.yml
error: error validating "./crd.yml": error validating data: ValidationError(CustomResourceDefinition.spec): unknown field "validation" in io.k8s.apiextensions-apiserver.pkg.apis.apiextensions.v1.CustomResourceDefinitionSpec; if you choose to ignore these errors, turn validation off with --validate=false
```
походу теперь в Schema надо это всё описывать,
перерисовал и добавил ещё что требовалось, завелось:
```
$ kubectl apply -f ./crd.yml
customresourcedefinition.apiextensions.k8s.io/mysqls.otus.homework created
$
$ kubectl apply -f ./cr.yaml
mysql.otus.homework/mysql-instance created
```
добавил пару строчек описания обязательных полей, применил, всё завелось
#### Деплой оператора
по методичке
```
$ kubectl apply -f ./cr.yaml
mysql.otus.homework/mysql-instance unchanged
```
проверяем pvc:
```
$ kubectl get pvc
No resources found in default namespace.
```
а где pvc ?
```
$ kubectl get pvc --all-namespaces
No resources found
$
$ kubectl get pod -A
NAMESPACE     NAME                               READY   STATUS              RESTARTS       AGE
default       mysql-operator-b9f7f9b48-hlj8z     0/1     ContainerCreating   0              4m34s
......
```
ясно, пошёл курить...
```
$ kubectl get pvc
NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
backup-mysql-instance-pvc   Bound    pvc-6ad85382-1ecd-4a64-8eaa-56d7c649cc9d   1Gi        RWO            standard       13m
mysql-instance-pvc          Bound    pvc-169694da-3262-44f0-a9fb-8d096ff7f67a   1Gi        RWO            standard       13m
```
проверяю дальше
```
$ export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
$

$ kubectl exec -it $MYSQLPOD -- mysql -u root -potuspassword -e "CREATE TABLE test ( id smallint unsigned not null auto_increment, name varchar(20) not null, constraint pk_example primary key (id) );" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
$
$ kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES (null, 'some data' );" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
$
$ kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES (null, 'some data-2' );" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
$
$ kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
```
Удалил mysql-instance:
```
$ kubectl delete mysqls.otus.homework mysql-instance
mysql.otus.homework "mysql-instance" deleted
$
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                               STORAGECLASS   REASON   AGE
backup-mysql-instance-pv                   1Gi        RWO            Retain           Available                                                               18m
pvc-169694da-3262-44f0-a9fb-8d096ff7f67a   1Gi        RWO            Delete           Released    default/mysql-instance-pvc          standard                18m
pvc-6ad85382-1ecd-4a64-8eaa-56d7c649cc9d   1Gi        RWO            Delete           Bound       default/backup-mysql-instance-pvc   standard                18m
$
$ kubectl get jobs.batch
NAME                        COMPLETIONS   DURATION   AGE
backup-mysql-instance-job   1/1           2s         57s
```
заводим, проверяем:
```
$ kubectl apply -f ./cr.yaml
mysql.otus.homework/mysql-instance created
$
$ export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
$
$ kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
```
работает
в дополнение, переименовал расширения у всех манифестов, как требовали (yml вместо yaml)
только, подозреваю, автотесты всё равно не пройдут, слишком синтаксис изменился в новых версиях
...
дополнительно для проверки ДЗ:
```
$ kubectl get jobs
NAME                         COMPLETIONS   DURATION   AGE
backup-mysql-instance-job    1/1           2s         9m7s
restore-mysql-instance-job   1/1           44s        6m36s
```
&
```
$ export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
$ kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
```

