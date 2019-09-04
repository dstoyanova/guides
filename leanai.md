# Lean AI Stack

## Setup the infrastructure

### Add Cloud
```
juju add-cloud test-cloud -f cloud_def_example.yaml 
```
File content:
```yaml
clouds:
  test-cloud:
    type: openstack
    auth-types: [userpass]
    endpoint: https://keystone.api.cloud.ipnett.se/v3
    regions:
      se-east-1:
        endpoint: https://keystone.api.cloud.ipnett.se/v3
```

### Add credentials to cloud
```
juju add-credential test-cloud -f credential_def_example.yaml 
```
File content:
```yaml
credentials:
  test-cloud:
    test-credential:
      auth-type: userpass
      tenant-id: da485ad604b544a88c4e05b28c6*****
      user-domain-name: scaleoutsystems.com
      project-domain-name: test.scaleoutsystems.com
      username: desislava.stoyanova@scaleoutsystems.com
      password: S%g3m=CVZh*****
```

### Generate image metadata
```
juju metadata generate-image -d simplestreams -i 9ddfcfd5-78bf-41c3-acb3-7f87216***** -s bionic -r se-east-1 -u https://keystone.api.cloud.ipnett.se/v3
```
This will create __simplestreams__ folder in the current directory.

### Create a bootstrap controller
```
juju bootstrap test-cloud test-controller --config network=d6ccc707-cacb-42a5-b547-f1f1518***** --config external-network=71b10496-2617-47ae-abbc-36239f0***** --config use-floating-ip=true --metadata-source simplestreams --bootstrap-constraints "mem=4G cores=2"
```

### Fix juju GUI
Once the previous step is completed, you need to fix a juju GUI. Run the following:
```
juju gui
```
You will get a link and credentials for logging into your GUI.

### Fix juju bunde.yaml
There is an example file in the examples folder.

### Deploy the bundle
```
juju model-config enable-os-refresh-update=false

juju model-config enable-os-upgrade=false

juju set-default-credential test-cloud test-credential

juju deploy -f bundle.yaml 

juju add-storage ceph-osd/0 osd-devices=cinder,100G,1
juju add-storage ceph-osd/1 osd-devices=cinder,100G,1
juju add-storage ceph-osd/2 osd-devices=cinder,100G,1

juju status --color
```

### Download kubeconfig
Once the previous step is completed, wait for 15 minutes and execute the following:
```
juju scp kubernetes-master/0:config kube/config

export KUBECONFIG=kube/config
```
This will create a __kube__ folder in the current directory.

### Fix DNS and SSL certificate
You need to have certbot installed first:
```
brew install certbot
``` 
Also, you need a DNS that supports TXT records. Then you can run the following command:
```
certbot certonly --manual --preferred-challenges=dns --email desislava@scaleoutsystems.com --server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d *.dev-001.scaleout.se
```
This will generate a TXT record that you need to add to the DNS. After this is done, press enter to continue. If everything has gone well, you have the certificate generated.

### Create a secret.yaml
Create a yaml file with the following structure:
```yaml
apiVersion: v1
data:
  tls.crt: 
  tls.key: 
kind: Secret
metadata:
  name: tls-secret
type: Opaque
```
To generate tls.crt:
```
certbot certificates

cat .../fullchain.pem | base64
```
To generate tls.key:
```
cat .../privkey.pem | base64
```
Then, you need to apply to kubernetes:
```
kubectl apply -f secret.yaml
```

### Modify the deployment.yaml file in the examples folder
Replace tls.crt and tls.key with the values you generated in the previous step. Also, replace the host and hosts (under rules and tls respectively) with ingress.

### Activate the ingress controller
```
juju config kubernetes-worker ingress=true

juju expose kubernetes-worker
```

### Set allow-privileged to true
```
juju config kubernetes-master allow-privileged=true
```

### Deploy rancher
```
kubectl apply -f deployment.yaml
```