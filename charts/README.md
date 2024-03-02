# Quickstart: Deploy an nginx demo application

This quickstart guide will show you how to deploy an nginx demo application and expose it to the internet using the self-hosted gateway chart.

## Create a Deployment

First, create a namespace for the nginx demo application:
```bash
kubectl create namespace nginx-demo
```
    
Then, deploy the nginx demo application:
```bash
kubectl create deployment nginx-demo \
    --image=nginx:latest \
    --namespace=nginx-demo
```

## Create a Service

Next, create a service to expose the nginx demo application:
```bash
kubectl expose deployment nginx-demo \
    --port=80 --target-port=80 \
    --namespace=nginx-demo
```

## Configure access to the Gateway

Define some variables:
```bash
export GATEWAY_HOST=<gateway-ipaddress>
export GATEWAY_USER=<gateway-username>
```

Generate a SSH key pair:
```bash
ssh-keygen -t rsa -b 4096 -f ./nginx-demo-id -N "" -C "nginx-demo"
```

Copy the public key to the Gateway:
```bash
# start a ssh-agent
eval `ssh-agent`
# add a key allowed to access the gateway
ssh-add ~/.ssh/id_rsa
# copy the created key to the gateway
ssh-copy-id -i ./nginx-demo-id.pub $GATEWAY_USER@$GATEWAY_HOST
```

Create a secret with the private key:
```bash
kubectl create secret generic nginx-demo-ssh-key \
    --from-file=private_key=./nginx-demo-id \
    --namespace=nginx-demo
```

## Set up the Gateway

SSH into the Gateway and run the following commands:
```bash
# create a docker network
docker network create gateway
# start the gateway server
docker run -it -d \
    --network gateway \
    --restart unless-stopped \
    -p 80:80 -p 443:443 \
    -e NGINX_ENVSUBST_OUTPUT_DIR=/etc/nginx \
    ghcr.io/rpersee/selfhosted-gateway:latest
```

## Expose the nginx demo application

Define the access FQDN:
```bash
EXPOSE_FQDN="nginx-demo-$(printf "%02x" ${GATEWAY_HOST//./ }).nip.io"
```

Define the values file:
```bash
cat <<EOF > nginx-demo-gateway-values.yaml
expose:
  hostFqdn: $EXPOSE_FQDN

  service:
    name: nginx-demo
    namespace: nginx-demo
    port: 80

gateway:
  host: $GATEWAY_HOST
  user: $GATEWAY_USER

  sshIdentity:
    secretName: nginx-demo-ssh-key
    secretKey: private_key
EOF
```

Deploy the self-hosted gateway chart:
```bash
helm install nginx-demo-gateway \
    --namespace nginx-demo \
    --values nginx-demo-gateway-values.yaml \
    charts/selfhosted-gateway
```

Example output:
```bash
$ kubectl -n nginx-demo get all
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/nginx-demo-5996b5bfd4-8nsmz                      1/1     Running   0          45m
pod/nginx-demo-selfhosted-gateway-5cccfd5664-s24vl   1/1     Running   0          6m10s

NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/nginx-demo   ClusterIP   10.96.94.16   <none>        80/TCP    44m

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-demo                      1/1     1            1           45m
deployment.apps/nginx-demo-selfhosted-gateway   1/1     1            1           6m10s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-demo-5996b5bfd4                      1         1         1       45m
replicaset.apps/nginx-demo-selfhosted-gateway-5cccfd5664   1         1         1       6m10s
```
