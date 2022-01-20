## kubernetes-security

task01

сначала создал манифест для сервисного аккаунта dave, затем админский sa для bob-а
just-dave.yaml
admin-bob.yaml

task02

создал ns prometheus, затем sa “carol” в созданном ns, потом роль и манифест для биндинга
step-1-prometheus-ns.yaml
step-2-sa-carol.yaml
step-3-prometheus-read-role.yaml
step-4-prometheus-role-binding.yaml

task03

сначала ns, потом пользователя и бинднг админом в ns, опять пользовать и биндинг как view в ns
step-1-dev-ns.yaml
step-2-sa-jane.yaml
step-3-jane-dev-admin.yaml
step-4-sa-ken.yaml
step-5-ken-dev-view.yaml

