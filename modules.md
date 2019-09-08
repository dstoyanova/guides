# Setup the modules

## Setup helm tiller
```
kubectl -n kube-system create serviceaccount tiller

kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

helm init --service-account tiller
```
Note: Requires kubectl and helm to be installed.

## Setup private registry
Use a command like the following to start the registry container:
```
$ docker run -d -p 5000:5000 --restart=always --name registry registry:2
```
Then, you are ready to setup the registry:
```
docker run --entrypoint htpasswd registry:2 -Bbn admin ~/auth/htpasswd
```
Copy the password for admin and paste it in values.yaml under secrets (htpasswd). Give a name for the hosts.
```
kubectl create namespace docker-registry

kubectl create -f tls-secret.yaml -n docker-registry

helm install stable/docker-registry -f values.yml --name=docker-registry --namespace=docker-registry

kubectl create secret docker-registry regcred3 --docker-server=https://docker-registry/ --docker-username=user --docker-password=pass
```

## Setup minio
Create a file values-minio.yaml:
```yaml
ingress:
  enabled: true
  path: /
  # Used to create an Ingress record.
  hosts:
    - docker-registry
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/add-base-url: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/secure-backends: “true”
    nginx.ingress.kubernetes.io/proxy-body-size: "5500m"
  labels: {}
  tls:
  - hosts:
    - docker-registry
    secretName: tls-secret

accessKey: user
secretKey: pass

persistence:
  enabled: true
  accessMode: ReadWriteOnce
  size: 60Gi
```
Then, run the following commands:
```
kubectl create namespace minio

kubectl create -f tls-secret.yaml -n minio

helm install --name minio-store --namespace store stable/minio -f values-minio.yml --namespace=minio
```
Create a file create_secret.sh:
```bash
#!/bin/bash
PASSWD=`echo -n user|base64`
USERNAME=`echo -n pass|base64`


echo "apiVersion: v1
kind: Secret
metadata:
  name: s3-credentials
type: Opaque
data:
  accesskey: ${USERNAME}
  secretkey: ${PASSWD}"
```
You can now run the following:
```
. create_secret.sh | kubectl create -f-
```

## Setup argo workflow
```
kubectl create namespace argo-events

kubectl create -f tls-secret.yaml -n argo-events

kubectl create secret docker-registry regcred3 --docker-server=https://docker-registry/ --docker-username=user --docker-password=pass -n argo-events

kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo/stable/manifests/install.yaml

kubectl create sa argo-events-sa -n argo-events && kubectl create sa argo-events-sa

kubectl apply -f argo-ingress.yml
```

## Setup scalar database
```
kubectl create namespace submit-scaler

kubectl create -f tls-secret.yaml -n submit-scaler

kubectl create secret docker-registry regcred3 --docker-server=https://docker-registry/ --docker-username=user --docker-password=pass -n submit-scaler
```

Create a file named values-db:
```yaml
mysqlRootPassword: "pass"
mysqlUser: "user"
mysqlPassword: "pass"
## Security context
securityContext:
 enabled: true
 runAsUser: 999
 fsGroup: 999
```

```
helm install --name scaler-db stable/mysql --namespace=submit-scaler -f values-db.yaml
```

### Seed the database
```
kubectl get pods -n submit-scaler

kubectl exec -it -n submit-scaler scaler-db-mysql-85996f7dbf-hnwkr /bin/bash

mysql -u root -p
```

Copy and paste the content from DB_Def.sql

```sql
insert into ClusterInfo (machineId, unitId, nodeName, removable, active) VALUES  ("17","kubernetes-worker/6", "	juju-37883b-default-17", False, True)
```

Exit and create a file called create_secret-db.sh:

```bash
#!/bin/bash
PASSWD=`echo -n pass|base64`
USERNAME=`echo -n root|base64`

echo "apiVersion: v1
kind: Secret
metadata:
  name: scaler-db-secret
type: Opaque
data:
  mysqlUser: ${USERNAME}
  mysqlPassword: ${PASSWD}"
```

```
. create_secret-db.sh | kubectl create -f- -n submit-scaler
```

## Setup Python Scaler
1. Navigate to modules/PythonScaler
2. Navigate to modules/PythonScaler/configs and run:
```
cp -r ~/.local/share/juju/
```
3. Edit and create secrets by:
    1. Edit create_secret.sh 
    ```bash
    #!/bin/bash
    JUJU_CONTROLLER_ENDPOINT=`echo -n 89.45.233.240:17070 | base64`
    JUJU_USERNAME=`echo -n admin | base64`
    JUJU_PASSWORD=`echo -n 37fc3807e59c4ea9f450d1c00d41611c | base64`
    JUJU_CACERT=`cat tls.ca | base64`

    echo "apiVersion: v1
    kind: Secret
    metadata:
      name: python-scaler-secret
    type: Opaque
    data:
      controller_endpoint: ${JUJU_CONTROLLER_ENDPOINT}
      username: ${JUJU_USERNAME}
      password: ${JUJU_PASSWORD}
      cacert: ${JUJU_CACERT}"
    ```
    2. Edit tls.ca
    ```
    juju show-controller
    ```
    Extract the certificate data and put it in 'tls.ca':

    ```
    -----BEGIN CERTIFICATE-----
    MIIDrDCCApSgAwIBAgIUGlx2FSZDDemI6CieiwoKESyKN/kwDQYJKoZIhvcNAQEL
    BQAwbjENMAsGA1UEChMEanVqdTEuMCwGA1UEAwwlanVqdS1nZW5lcmF0ZWQgQ0Eg
    Zm9yIG1vZGVsICJqdWp1LWNhIjEtMCsGA1UEBRMkZDhjNjE5YWYtNzRiMy00ZjY5
    LTg2NWUtZDk2MDE1MWJmZGUyMB4XDTE5MDgyOTE0NDAxNFoXDTI5MDkwNTE0NDAx
    NFowbjENMAsGA1UEChMEanVqdTEuMCwGA1UEAwwlanVqdS1nZW5lcmF0ZWQgQ0Eg
    Zm9yIG1vZGVsICJqdWp1LWNhIjEtMCsGA1UEBRMkZDhjNjE5YWYtNzRiMy00ZjY5
    LTg2NWUtZDk2MDE1MWJmZGUyMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC
    AQEAwQNUHHbIeOl2Oy8cn2dyUakhRt5R6hEdg0SoEAzSQT3YY30kgyRTP6j43rYp
    akQ2qkoiidtYHAOZe//mCQFUCqwYRFX/jA6e9MB0Lx6F17GabTd/IBXO6DEJCyW+
    occi0mPJ2xmnj+EpRMNZfPdCz3DMaty132ZgEkrmrIm5i9Av0OV8VEviz1ywRyBN
    Pq1guDwep9tDZIjjCZUy7b/24Y86c30nHFKhROCIWuIVLRMFj3o39NvqEoBAlOW3
    lf0HfRaBNoiQUdjoOzEg6p4VSXJiUD1BBEMYpVAdMcFpDlb9uSiYtYIY9tBt2mdH
    fqrAcwlbqQDLV/Fk8svTMyHCuQIDAQABo0IwQDAOBgNVHQ8BAf8EBAMCAqQwDwYD
    VR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUFXe/l2Gc768P7vHfC5D7pcdnSkEwDQYJ
    KoZIhvcNAQELBQADggEBAE/jFjB+fefCA+mg+dLQWR0qaN8pHQNnRpYddaDh1FEj
    sAj4APbgIsxxemwcxRZHxcPyY0QgMZZ04S0mZkcxVaOigAH4LtsYdRkxRTHlWlP5
    crS645uz34su1tgU+DZj2ulpr/SQnY2QEXbSvo9RbXb3iZvknrBJs2ZGXGHCJFzw
    VGee/jAiebHSwZIcNyZxlNRSJrTiWPs9QkrODiCDNNgZJBt6/lOs0Bz3cI5uyjMW
    Mz8BGbMRqCen92CScrtxvBMS7n7W3prNsobRgMIhvqfwW9XhEW+zH7CLsFyaJCdk
    53Qqd9Px90Hw7+emsaTZyTCYWpxNrDgKyFNMlWeCQbk=
    -----END CERTIFICATE-----
    ```
    3. Then create the secret by:
    ```
    . create_secret.sh | kubectl create -f- -n submit-scaler
    ```
4. Make sure you have the Dockerfile and run:
```
docker login

docker build . -t dstoyanova/resource-scaler:latest

docker push dstoyanova/resource-scaler:latest
```
5. Edit values.yaml under .charts and run:
```
helm install .charts --name python-scaler -f .charts/values.yaml --namespace=submit-scaler
```

## Setup work queue
1. Navigate to modules/argo-submit-server/workqueue
2. Edit values.yaml
3. Run the following
```
helm install --name rabbit-mq stable/rabbitmq -f values.yaml --namespace=submit-scaler
```
4. Enter username/password from values.yaml in the create_secret.sh and run:
```
. create_secret.sh | kubectl create -f- -n submit-scaler
```

## Setup argo submit server
1. Navigate to: modules/argo-submit-server/go-client
2. Edit argo-cli.go constant
3. Copy your kube config to argo
```
cp ~/.kube/config config
```
