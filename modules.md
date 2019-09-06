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