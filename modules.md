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
