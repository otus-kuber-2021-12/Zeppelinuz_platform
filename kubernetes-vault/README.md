### kubernetes-vault

GKE
```
$ kubectl get nodes
NAME                                       STATUS   ROLES    AGE   VERSION
gke-cluster-1-default-pool-7f4da77c-jr41   Ready    <none>   21m   v1.21.10-gke.2000
gke-cluster-1-default-pool-7f4da77c-kmfv   Ready    <none>   21m   v1.21.10-gke.2000
gke-cluster-1-default-pool-7f4da77c-tdck   Ready    <none>   21m   v1.21.10-gke.2000
```

$ git clone https://github.com/hashicorp/consul-helm.git
$ helm install consul-helm --generate-name
$ git clone https://github.com/hashicorp/vault-helm.git

исправил values.yaml

```
$ helm install --generate-name .
$
$ helm status chart-1650945137
NAME: chart-1650945137
LAST DEPLOYED: Tue Apr 26 09:52:19 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named chart-1650945137. To learn more about the release, try:

  $ helm status chart-1650945137
  $ helm get manifest chart-1650945137
```

а в логах
```
$ kubectl logs chart-1650945137-vault-0
==> Vault server configuration:

             Api Address: http://10.4.0.10:8200
                     Cgo: disabled
         Cluster Address: https://chart-1650945137-vault-0.chart-1650945137-vault-internal:8201
              Go Version: go1.17.5
              Listener 1: tcp (addr: "[::]:8200", cluster address: "[::]:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: consul (HA available)
                 Version: Vault v1.9.3
             Version Sha: 7dbdd57243a0d8d9d9e07cd01eb657369f8e1b8a

==> Vault server started! Log data will stream in below:

2022-04-26T03:52:39.802Z [INFO]  proxy environment: http_proxy="\"\"" https_proxy="\"\"" no_proxy="\"\""
2022-04-26T03:52:39.802Z [WARN]  storage.consul: appending trailing forward slash to path
2022-04-26T03:52:39.911Z [INFO]  core: Initializing VersionTimestamps for core
2022-04-26T03:52:45.103Z [INFO]  core: security barrier not initialized
2022-04-26T03:52:45.104Z [INFO]  core: seal configuration missing, not initialized
2022-04-26T03:52:50.047Z [INFO]  core: security barrier not initialized
2022-04-26T03:52:50.049Z [INFO]  core: seal configuration missing, not initialized
2022-04-26T03:52:55.040Z [INFO]  core: security barrier not initialized
2022-04-26T03:52:55.041Z [INFO]  core: seal configuration missing, not initialized
2022-04-26T03:53:00.213Z [INFO]  core: security barrier not initialized
2022-04-26T03:53:00.215Z [INFO]  core: seal configuration missing, not initialized
2022-04-26T03:53:05.095Z [INFO]  core: security barrier not initialized
2022-04-26T03:53:05.097Z [INFO]  core: seal configuration missing, not initialized
2022-04-26T03:53:10.044Z [INFO]  core: security barrier not initialized
2022-04-26T03:53:10.045Z [INFO]  core: seal configuration missing, not initialized
2022-04-26T03:53:15.172Z [INFO]  core: security barrier not initialized
2022-04-26T03:53:15.173Z [INFO]  core: seal configuration missing, not initialized
2022-04-26T03:53:20.049Z [INFO]  core: security barrier not initialized
2022-04-26T03:53:20.050Z [INFO]  core: seal configuration missing, not initialized
и т.д.
```
инициировал через любой под
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault operator init --key-shares=1 --key-threshold=1
Unseal Key 1: P8ECeYkFseoNCFaqka7GnA6FTCr6RtWd/GDw6utG9ZU=

Initial Root Token: s.ifnPD4slFptizkgWS5gxfYtb

Vault initialized with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 1 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 1 keys to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

в логах при этом
```
2022-04-26T03:58:15.376Z [INFO]  core: security barrier initialized: stored=1 shares=1 threshold=1
2022-04-26T03:58:15.471Z [INFO]  core: post-unseal setup starting
2022-04-26T03:58:15.526Z [INFO]  core: loaded wrapping token key
2022-04-26T03:58:15.535Z [INFO]  core: Recorded vault version: vault version=1.9.3 upgrade time="2022-04-26 03:58:15.526046718 +0000 UTC m=+335.890586144"
2022-04-26T03:58:15.537Z [INFO]  core: successfully setup plugin catalog: plugin-directory="\"\""
2022-04-26T03:58:15.540Z [INFO]  core: no mounts; adding default mount table
2022-04-26T03:58:15.610Z [INFO]  core: successfully mounted backend: type=cubbyhole path=cubbyhole/
2022-04-26T03:58:15.611Z [INFO]  core: successfully mounted backend: type=system path=sys/
2022-04-26T03:58:15.613Z [INFO]  core: successfully mounted backend: type=identity path=identity/
2022-04-26T03:58:15.809Z [INFO]  core: successfully enabled credential backend: type=token path=token/
2022-04-26T03:58:15.818Z [INFO]  core: restoring leases
2022-04-26T03:58:15.818Z [INFO]  rollback: starting rollback manager
2022-04-26T03:58:15.822Z [INFO]  expiration: lease restore complete
2022-04-26T03:58:15.842Z [INFO]  identity: entities restored
2022-04-26T03:58:15.843Z [INFO]  identity: groups restored
2022-04-26T03:58:15.918Z [INFO]  core: usage gauge collection is disabled
2022-04-26T03:58:15.927Z [INFO]  core: post-unseal setup complete
2022-04-26T03:58:16.021Z [INFO]  core: root token generated
2022-04-26T03:58:16.021Z [INFO]  core: pre-seal teardown starting
2022-04-26T03:58:16.021Z [INFO]  rollback: stopping rollback manager
2022-04-26T03:58:16.021Z [INFO]  core: pre-seal teardown complete
```
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       1
Threshold          1
Unseal Progress    0/1
Unseal Nonce       n/a
Version            1.9.3
Storage Type       consul
HA Enabled         true
command terminated with exit code 2
```
статусы
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault operator unseal
Unseal Key (will be hidden): 
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.9.3
Storage Type    consul
Cluster Name    vault-cluster-95ee2207
Cluster ID      830aa6ce-4696-db04-b200-8e0927317a85
HA Enabled      true
HA Cluster      https://chart-1650945137-vault-0.chart-1650945137-vault-internal:8201
HA Mode         active
Active Since    2022-04-26T05:48:19.446407407Z
$ 
$ kubectl exec -it chart-1650945137-vault-1 -- vault operator unseal
Unseal Key (will be hidden): 
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.9.3
Storage Type           consul
Cluster Name           vault-cluster-95ee2207
Cluster ID             830aa6ce-4696-db04-b200-8e0927317a85
HA Enabled             true
HA Cluster             https://chart-1650945137-vault-0.chart-1650945137-vault-internal:8201
HA Mode                standby
Active Node Address    http://10.4.0.10:8200
$ 
$ kubectl exec -it chart-1650945137-vault-2 -- vault operator unseal
Unseal Key (will be hidden): 
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.9.3
Storage Type           consul
Cluster Name           vault-cluster-95ee2207
Cluster ID             830aa6ce-4696-db04-b200-8e0927317a85
HA Enabled             true
HA Cluster             https://chart-1650945137-vault-0.chart-1650945137-vault-internal:8201
HA Mode                standby
Active Node Address    http://10.4.0.10:8200
$
```
далее по методичке получаем ошибку:
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault auth list
Error listing enabled authentications: Error making API request.

URL: GET http://127.0.0.1:8200/v1/sys/auth
Code: 400. Errors:

* missing client token
command terminated with exit code 2
```
логин в вольт c рутовым токеном
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault login
Token (will be hidden): 
Error authenticating: error looking up token: Error making API request.

URL: GET http://127.0.0.1:8200/v1/auth/token/lookup-self
Code: 403. Errors:

* permission denied
```
ой, промахнулся
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault login
Token (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.ifnPD4slFptizkgWS5gxfYtb
token_accessor       ThWj1EUq5pYOXfJXdYCKkqUq
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```
список авторизаций:
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault auth list
Path      Type     Accessor               Description
----      ----     --------               -----------
token/    token    auth_token_b539f953    token based credentials
$
```
добавил секреты
```
$ 
$ kubectl exec -it chart-1650945137-vault-0 -- vault secrets enable --path=otus kv
Success! Enabled the kv secrets engine at: otus/
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault secrets list --detailed
Path          Plugin       Accessor              Default TTL    Max TTL    Force No Cache    Replication    Seal Wrap    External Entropy Access    Options    Description                                                UUID
----          ------       --------              -----------    -------    --------------    -----------    ---------    -----------------------    -------    -----------                                                ----
cubbyhole/    cubbyhole    cubbyhole_304b10c4    n/a            n/a        false             local          false        false                      map[]      per-token private secret storage                           9f829ca0-38c9-c860-33b9-62bdf713024f
identity/     identity     identity_20a66fd3     system         system     false             replicated     false        false                      map[]      identity store                                             92016cd5-f203-0d6a-1ea9-897465387839
otus/         kv           kv_772d5cd8           system         system     false             replicated     false        false                      map[]      n/a                                                        80cf4f5e-8d33-67c2-006c-8896ae88a984
sys/          system       system_d3fbd7c8       n/a            n/a        false             replicated     false        false                      map[]      system endpoints used for control, policy and debugging    027741ac-a6d4-1fea-041e-cf9016f52264
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault kv put otus/otus-ro/config username='otus' password='asajkjkahs'
Success! Data written to: otus/otus-ro/config
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault kv put otus/otus-rw/config username='otus' password='asajkjkahs'
Success! Data written to: otus/otus-rw/config
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault read otus/otus-ro/config
Key                 Value
---                 -----
refresh_interval    768h
password            asajkjkahs
username            otus
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault kv get otus/otus-rw/config
====== Data ======
Key         Value
---         -----
password    asajkjkahs
username    otus
$
```
далее авторизацию черерз k8s
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault auth list
Path           Type          Accessor                    Description
----           ----          --------                    -----------
kubernetes/    kubernetes    auth_kubernetes_1db75c61    n/a
token/         token         auth_token_b539f953         token based credentials
```
двигаемся дальше
```
$ kubectl create serviceaccount vault-auth
serviceaccount/vault-auth created
$ kubectl apply -f ./vault-auth-service-account.yaml 
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRoleBinding
clusterrolebinding.rbac.authorization.k8s.io/role-tokenreview-binding created
```
по методичке, записал конфиг в vault
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault write auth/kubernetes/config token_reviewer_jwt="$SA_JWT_TOKEN" kubernetes_host="$K8S_HOST" kubernetes_ca_cert="$SA_CA_CRT"
Success! Data written to: auth/kubernetes/config
```
далее политику и роль, как обычно не обошлось без танцев с бубном,
получил ошибку при поытке копироать файл в под:
```
$ kubectl cp otus-policy.hcl chart-1650945137-vault-0:./
tar: can't open 'otus-policy.hcl': Permission denied
command terminated with exit code 1
```
пришлось скопировать в другое место, задем залогиниться в под и переместить
```
$ kubectl cp otus-policy.hcl chart-1650945137-vault-0:/tmp/
$ 
$ kubectl exec -i -t chart-1650945137-vault-0 -- sh -c "clear; (bash || ash || sh)"
+++++++++++++++++++++++++
/ $ cd
~ $ ls -la
total 20
drwxrwsrwx    3 root     vault         4096 Apr 26 07:19 .
drwxr-xr-x    1 root     root          4096 Jan 24 22:22 ..
-rw-------    1 vault    vault           17 Apr 26 07:19 .ash_history
drwxr-sr-x    3 vault    vault         4096 Apr 26 03:52 .cache
-rw-------    1 vault    vault           26 Apr 26 05:54 .vault-token
~ $ pwd
/home/vault
~ $ mv /tmp/otus-policy.hcl ./
~ $ ls -la
total 24
drwxrwsrwx    3 root     vault         4096 Apr 26 07:19 .
drwxr-xr-x    1 root     root          4096 Jan 24 22:22 ..
-rw-------    1 vault    vault           58 Apr 26 07:19 .ash_history
drwxr-sr-x    3 vault    vault         4096 Apr 26 03:52 .cache
-rw-------    1 vault    vault           26 Apr 26 05:54 .vault-token
-rw-r--r--    1 vault    vault          133 Apr 26 07:17 otus-policy.hcl
~ $ 
+++++++++++++++++++++++++
```
потом то просто пути ставил правельнее и работало... двигаем дальше
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault policy write otus-policy /home/vault/otus-policy.hcl
Success! Uploaded policy: otus-policy
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault write auth/kubernetes/role/otus \
> bound_service_account_names=vault-auth \
> bound_service_account_namespaces=default policies=otus-policy ttl=24h
Success! Data written to: auth/kubernetes/role/otus
```
далее по методичке, записать не смогли otus-rw/config потому что прав не
хватает, необхоидимо в otus-policy.hcl добавить "update"
едем дальше
```
$ kubectl apply -f ./config/configmap.yaml 
configmap/example-vault-agent-config created
$
$ kubectl apply -f ./config/example-k8s-spec.yaml 
pod/vault-agent-example created
$ kubectl exec -ti vault-agent-example -c nginx-container  -- cat /usr/share/nginx/html/index.html
<html>
<body>
<p>Some secrets:</p>
<ul>
<li><pre>username: otus</pre></li>
<li><pre>password: asajkjkahs</pre></li>
</ul>

</body>
</html>
```
pki
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault secrets tune -max-lease-ttl=87600h pki
Success! Tuned the secrets engine at: pki/
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault write -field=certificate pki/root/generate/internal common_name="exmaple.ru" ttl=87600h > CA_cert.crt
$
```
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault write pki/config/urls issuing_certificates="http://vault:8200/v1/pki/ca" crl_distribution_points="http://vault:8200/v1/pki/crl"
Success! Data written to: pki/config/urls
```
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault secrets enable --path=pki_int pki
Success! Enabled the pki secrets engine at: pki_int/
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault secrets tune -max-lease-ttl=87600h pki_int
Success! Tuned the secrets engine at: pki_int/
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault write -format=json pki_int/intermediate/generate/internal common_name="example.ru Intermediate Authority" | jq -r '.data.csr' > pki_intermediate.csr
$
```
```
$ kubectl cp pki_intermediate.csr chart-1650945137-vault-0:/home/vault/
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault write -format=json pki/root/sign-intermediate csr=@/home/vault/pki_intermediate.csr format=pem_bundle ttl="43800h" | jq -r '.data.certificate' > intermediate.cert.pem
$
$ kubectl cp intermediate.cert.pem chart-1650945137-vault-0:/home/vault/
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault write pki_int/intermediate/set-signed certificate=@/home/vault/intermediate.cert.pem
Success! Data written to: pki_int/intermediate/set-signed
```
далее новые, всё по методичке
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault write pki_int/roles/example-dot-ru allowed_domains="example.ru" allow_subdomains=true max_ttl="720h"
Success! Data written to: pki_int/roles/example-dot-ru
$
$ kubectl exec -it chart-1650945137-vault-0 -- vault write pki_int/issue/example-dot-ru common_name="test.example.ru" ttl="24h"
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUdBAC5aIeiZ2ceaCHdDuoFvPneKQwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMjA0MjcwMzMyNTZaFw0yNzA0
MjYwMzMzMjZaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALSv/4oIv/cM
+kJFvuoHjFzBlgbUEQnFtM/ewZr6dqBfOx6MeDkFIaSy+MbOtLPtZElWElOmYA3Y
64LeEeDayKAgPln8FfTd14QZe/+YuoE7e2U5AaPO2Itt8SfnL+iRt8sHrT1At4RK
OC0+vPHMc8sOCNTjD4a6LotaGTrlxR5mn1pFp+iDIpmfL/4QMKqwco4IMMiysoPZ
f3RgBjswv6sVtFoRM0EbOozaHSVCpqLGl0QE7css4MriUnJqI4q4U7t/eD5DlTPY
jLWMnrmTr9AOPVZyVz866rz8fEa3zJY6/yTuHyF+KS3MbjFCGugXghKMnc46PVw8
c3nTdOxmBu8CAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQU+EPFEZAmP9BXsGBP2XbaXWj3SmcwHwYDVR0jBBgwFoAU
gZhx06LTC2vpFIRIsX85Jeb5PtowNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
RkuDkXZcK2Dxyqi13IwsEiR79+pNHmbhayb595WkjyMF0vFJ5Ixu26x2g8z2zx+c
boZsusofJXhcn0odlrb1Y/z3RCuPo8vz/GCj6QKV2DSTGTpzOmJlZ9ZZBvYNS7sY
gsKndZim2yWAsBgCjySLibQiekbPOJ7zq6tpSM4Cg7ELAYB8nNToZIMu3pxx5CKk
k6P7UEjcCXoIEAYHEAp5YGBBy28jBFVrI3jQqwxLt77inFHX9vlfw4Yxv6Elp20w
2KDqtu4ALpLBcFZmAitoKOdW+5M/szzJsXJyUhqh5Ql8ChFD1sCFFMu09OncvEq7
76cZJ9Nh6YcbZ3BYpe9+XQ==
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
MIIDYzCCAkugAwIBAgIUSTAcp++f/3iAofqSuWa9HsvA7XYwDQYJKoZIhvcNAQEL
BQAwLDEqMCgGA1UEAxMhZXhhbXBsZS5ydSBJbnRlcm1lZGlhdGUgQXV0aG9yaXR5
MB4XDTIyMDQyNzAzNDA0M1oXDTIyMDQyODAzNDExM1owGjEYMBYGA1UEAxMPdGVz
dC5leGFtcGxlLnJ1MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAv5Nx
7N8KUXAeDbZHU3hAkMZZSTyNYExLYRFW8kE32Kqw8U4knqUIB0uzIZuZjCa6UPns
rW1CaG7Vh4zlL5yBhbraBgJDOwo+K38q3c8m72KwwhNAFimOcdy1cMaW/MV9gQAn
o1ij3p7WRRqj/Tcb1K1Y6ad+37fqEtHXPRFBKi6X/sQuu46A6yqVnLSL1T4Bv8Qr
+qSg5jOBZIYRsBsssj3uyID8bUhpoJ0E9OQNma94BBbooIfod//athxnrhqz3T6N
kviLQNhzovO5G8/a5kdxONlccnDxNByRn/3wQF+JuC5tOnRJ/Ib79wj/RvsrPKb4
6FfGibEmnX6C6lvg7wIDAQABo4GOMIGLMA4GA1UdDwEB/wQEAwIDqDAdBgNVHSUE
FjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwHQYDVR0OBBYEFJaLg4xB/NewE5lj8l5M
mlg7Q8MRMB8GA1UdIwQYMBaAFPhDxRGQJj/QV7BgT9l22l1o90pnMBoGA1UdEQQT
MBGCD3Rlc3QuZXhhbXBsZS5ydTANBgkqhkiG9w0BAQsFAAOCAQEApm8EH9OffNcf
wjOC50pVvlv0aRBVMaO/4lLOUgFO3deC101mCXzh1cCOlKNKZLDJyQZWt9Q1m0F0
reoDTUKpRnUR+FFTRdOIhahVPcMw3J/K0u80EcFPnOtiXu1C2L8bFPwwalxWc02L
0hyTt8QPPkCM58cD0HGhvfQNNkrtExmUd6UeQZarDTOy+0hrkVvOeS25PrnmFDHr
vuBOxUKKzwP5p8tkXlObwXDSaohWoGHuD3Jxh2eq4CNrXjtiLqSWbKEbIcrWRI1P
oJmu4B8dc9l2Rmv7MTyJ5lE4/60EfcdokSrUxoiXDgGiV6Utse2u/DJk2fBRY42i
meAMX6Fv4A==
-----END CERTIFICATE-----
expiration          1651117273
issuing_ca          -----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUdBAC5aIeiZ2ceaCHdDuoFvPneKQwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMjA0MjcwMzMyNTZaFw0yNzA0
MjYwMzMzMjZaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALSv/4oIv/cM
+kJFvuoHjFzBlgbUEQnFtM/ewZr6dqBfOx6MeDkFIaSy+MbOtLPtZElWElOmYA3Y
64LeEeDayKAgPln8FfTd14QZe/+YuoE7e2U5AaPO2Itt8SfnL+iRt8sHrT1At4RK
OC0+vPHMc8sOCNTjD4a6LotaGTrlxR5mn1pFp+iDIpmfL/4QMKqwco4IMMiysoPZ
f3RgBjswv6sVtFoRM0EbOozaHSVCpqLGl0QE7css4MriUnJqI4q4U7t/eD5DlTPY
jLWMnrmTr9AOPVZyVz866rz8fEa3zJY6/yTuHyF+KS3MbjFCGugXghKMnc46PVw8
c3nTdOxmBu8CAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQU+EPFEZAmP9BXsGBP2XbaXWj3SmcwHwYDVR0jBBgwFoAU
gZhx06LTC2vpFIRIsX85Jeb5PtowNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
RkuDkXZcK2Dxyqi13IwsEiR79+pNHmbhayb595WkjyMF0vFJ5Ixu26x2g8z2zx+c
boZsusofJXhcn0odlrb1Y/z3RCuPo8vz/GCj6QKV2DSTGTpzOmJlZ9ZZBvYNS7sY
gsKndZim2yWAsBgCjySLibQiekbPOJ7zq6tpSM4Cg7ELAYB8nNToZIMu3pxx5CKk
k6P7UEjcCXoIEAYHEAp5YGBBy28jBFVrI3jQqwxLt77inFHX9vlfw4Yxv6Elp20w
2KDqtu4ALpLBcFZmAitoKOdW+5M/szzJsXJyUhqh5Ql8ChFD1sCFFMu09OncvEq7
76cZJ9Nh6YcbZ3BYpe9+XQ==
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAv5Nx7N8KUXAeDbZHU3hAkMZZSTyNYExLYRFW8kE32Kqw8U4k
nqUIB0uzIZuZjCa6UPnsrW1CaG7Vh4zlL5yBhbraBgJDOwo+K38q3c8m72KwwhNA
FimOcdy1cMaW/MV9gQAno1ij3p7WRRqj/Tcb1K1Y6ad+37fqEtHXPRFBKi6X/sQu
u46A6yqVnLSL1T4Bv8Qr+qSg5jOBZIYRsBsssj3uyID8bUhpoJ0E9OQNma94BBbo
oIfod//athxnrhqz3T6NkviLQNhzovO5G8/a5kdxONlccnDxNByRn/3wQF+JuC5t
OnRJ/Ib79wj/RvsrPKb46FfGibEmnX6C6lvg7wIDAQABAoIBAAwKYVOo5QYfTNRB
y5PUcAJpZP00YBJYWTh9lYBeVvs4JyzTY3vRFYMX3+dR10G2wWkLfDOeNVlI9gSx
90mZxY45IzDTfZQ9XZDwSipstZ7ADin0eceqzvgbDhBLevviEbRE5Tjf/lSkmQT4
2qu0hfxE9NyimVfIQF70b1m4NudGsIqk6ShGSOb63pG2EvUteGMdNoB7p/rr9tEl
rf8LSUnqgpRHEJZaL0kxHyD7zEl0rF6Qwcl3VNRvaHbRVfpH6vRVErWtrMcPzTX0
UmbeSgIHP82flnwuR0D1HisVj/5QfTfbxkck1Z3M6+L4Ushjy2GV93HXdJIqauDj
PWTgdzECgYEA6BI+nAqOOhpjBqBqs1BxEi51uX0Y3+KI7/vNbA2yY76JvQuC67v+
7AjCSZ04wTsHUeIRh13elHXFay11H03fet3BOD7tuMOyLJ6Zsz6Uuu6rZxlUGR9a
4Qv8I5QZDtDO4j0Kzo0HRsfhXSAMS0GpiTaDCDvk70Czguwf+aEmyzkCgYEA01RJ
LLCpsHSvmKUc3ng4LGCK8auXCWJjTAqn62KQCzE+8lt/6DTQEeyaYrVlvAoSYDqv
5n+P49c0ebZotZlax94c31GoGRvY4emX9x9g+4bp7NugmknHOodxd4sCdaxxa9tJ
hgIxXMm2uz/ijZEvVvvtDxPP2vm/u/mU1rgIBWcCgYEAvq7nBN3jeThfL321znqF
PbQxBOUWADep3s4ePu+OKUjQ8iU4QKvqzVRxF314ucTfwdcoIfruPTv7p5HlT4Bz
5Qe6kJWcTJl3mBQFJHOCT4p2CbOVF0NdL9biKPWyFStbIieX7pmQZgcsVJFVqKxe
OiExTx2vgSq/lQ6hQ0K3lnkCgYBgf65iT9FMmBvO0iaal77e1L7dmAMB8AFzqbH/
1CP+WGBr/sgrWmJgrO/afwaTlO3LL0E/OaSU36JAqcCqm/pOJeh9OSZPQN4KWsZf
u95nPLX4yFlP2ry0x0BS3BEldrbcD2hFXx73RczBOGzVRCSfza30IpHZZg3dYhxK
6AIRpQKBgQCk0PcDyjZXmro1xVoVIHS1cTf9ZGXaCDdpHGQq7XpWQYozd4K7RdRq
nJ7ArE8A+HytukAdUC0fhWE6jJHdYFegUIwPKcxAdw8Cud7pfwChpw7PvYJExVGX
dw25XEGjbcMKVeSbffvynhfbO+M+G1AagpcaEPIfMEf6t32WuRI7bA==
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       49:30:1c:a7:ef:9f:ff:78:80:a1:fa:92:b9:66:bd:1e:cb:c0:ed:76
```
и отзыв
```
$ kubectl exec -it chart-1650945137-vault-0 -- vault write pki_int/revoke serial_number="49:30:1c:a7:ef:9f:ff:78:80:a1:fa:92:b9:66:bd:1e:cb:c0:ed:76"
Key                        Value
---                        -----
revocation_time            1651030970
revocation_time_rfc3339    2022-04-27T03:42:50.31500196Z
$
```
