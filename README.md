# Matrix Synapse server on Kubernetes
Synapse is the reference implementation of the Matrix protocol, a decentralized communication standard for instant messaging and VoIP. It serves as the server software for the Matrix communication network, allowing users to communicate seamlessly across different platforms and services in a federated and open manner. Synapse supports end-to-end encryption, user authentication, and the storage and retrieval of messages in a distributed and secure way.

In this repository we want to run a synapse matrix on kubernetes in the following part.
- [Requirements](#requirements)
- [Install some tools](#install-some-tools)
    * [Install helm](#install-helm)
    * [Install NGINX Ingress Controller](#install-nginx-ingress-controller)
- [Deploy a Matrix](#deploy-a-matrix) 
    * [Namespace](#namespace)
    * [Setting up PostgreSQL](#setting-up-postgresql)
    * [Setting up Synapse](#setting-up-synapse)
    * [Setting up the Ingress](#setting-up-the-ingress)
- [Federation](#federation)
    * [Setting up delegation](#setting-up-delegation)
- [Enable VoIP calls](#enable-voip-calls)

### Requirements
- Before starting, I have deployed a [Kubespray](https://github.com/kubernetes-sigs/kubespray) on two servers as master and worker. (You can use the Kubernetes you want)
- we assuming, we've set up a domain and two subdomains for the Synapse service.
    * domain: example.com
    * subdomains: matrix, turn
- Basic Knowledge about Kubernetes and Linux itself

## Install some tools

### Install helm 
```
curl -o /tmp/helm.tar.gz -LO https://get.helm.sh/helm-v3.10.1-linux-amd64.tar.gz
tar -C /tmp/ -zxvf /tmp/helm.tar.gz
mv /tmp/linux-amd64/helm /usr/local/bin/helm
chmod +x /usr/local/bin/helm
```
### Install NGINX Ingress Controller
The controller ships as a helm chart, I grab version 1.9.5 as per the compatibility matrix.
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm search repo ingress-nginx --versions
```
From the app version I select the version that matches the compatibility matrix.
```
NAME                       	CHART VERSION	APP VERSION	DESCRIPTION                                       
ingress-nginx/ingress-nginx	4.9.0        	1.9.5      	Ingress controller for Kubernetes using NGINX a...
```

Now we can use helm to install the chart directly
```
CHART_VERSION="4.4.0"
APP_VERSION="1.5.1"

mkdir ./kubernetes/ingress/controller/nginx/manifests/

helm template ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--version ${CHART_VERSION} \
--namespace ingress-nginx \
> ./kubernetes/ingress/controller/nginx/manifests/nginx-ingress.${APP_VERSION}.yaml

# Deploy the Ingress controller
kubectl create namespace ingress-nginx
kubectl apply -f ./kubernetes/ingress/controller/nginx/manifests/nginx-ingress.${APP_VERSION}.yaml
```
### Deploy the Ingress controller
```
kubectl create namespace ingress-nginx
kubectl apply -f ./kubernetes/ingress/controller/nginx/manifests/nginx-ingress.${APP_VERSION}.yaml
```

```
kubectl -n ingress-nginx get pods,svc

NAME                                            READY   STATUS      RESTARTS        AGE
pod/ingress-nginx-admission-create-q4b2f        0/1     Completed   0               21m
pod/ingress-nginx-admission-patch-j67vn         0/1     Completed   0               21m
pod/ingress-nginx-controller-76df688779-4ph6z   1/1     Running     0               21m

NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.233.7.145    <pending>     80:30818/TCP,443:31379/TCP   21m
service/ingress-nginx-controller-admission   ClusterIP      10.233.25.188   <none>        443/TCP                      21m
```

## Deploy a Matrix
Now we are ready to implement the matrix synapse.

```
cd matrix-synapse-kubernetes
```
### Namespace
Initial we need to create a namespace
```
kubectl apply -f initial/namespace.yaml
```
### Setting up PostgreSQL
Synapse does support PostgreSQL and SQLite as database. We want to use postgreSQl for this project.

befor apply configMap.yaml, Don't forget to set a secure password for the user.
```
kubectl apply -f postgres/configMap.yaml
```
Next a PersistentVolume as well as a PersistentVolumeClaim.
```
kubectl apply -f postgres/pv.yaml
```
Now we will configure a StatefulSet that will deploy one PostgreSQL POD.
```
kubectl apply -f postgres/statefulSet.yaml
```
Lastly we need a Service so Synapse can connect to PostgreSQL.
```
kubectl apply -f postgres/service.yaml
```

### Setting up Synapse

First create another PersistentVolume and PersistentVolumeClaim for Synapse to store configuration and media data.
```
kubectl apply -f synapse/pv.yaml
```
We want Synapse to run as a normal Deployment.
```
kubectl apply -f synapse/deployment.yaml
```
After applying synapse/deployment.yaml, you will find a homeserver.yaml in /etc/matrix/synapse/data.
If so, you can delete the deployment.
```
kubectl -n synapse delete deployment.apps synapse
```
Now open homeserver.yaml with an editor and configure Synapse itself.
- Replace `server_name: "example.org"` with your `server_name`
- Change the section `database:` with the following code.
- Set database password.
```
database:
  name: psycopg2
  args:
    user: synapse-db
    password: {CHANGEME}
    database: synapse
    host: postgres-service
    cp_min: 5
    cp_max: 10
```

We apply Synapse again with Deployment, before that we need to remove or comment out the following.
```
args: ["generate"]
env:
- name: SYNAPSE_SERVER_NAME
    value: "example.com"
- name: SYNAPSE_REPORT_STATS
    value: "yes"
```
The deployment can now be configured on the cluster again.
```
kubectl apply -f synapse/deployment.yaml
```
You can check the Synapse
```
kubectl -n synapse get deployments.apps synapse 

NAME      READY   UP-TO-DATE   AVAILABLE   AGE
synapse   1/1     1            1           2m

```

### Setting up the Ingress
Now we need to make the synapse accessible from the outside.
First we apply the service. 
```
kubectl apply -f synapse/service.yaml
```
Then apply the ingress. befor that set your domain name.
```
kubectl apply -f synapse/ingress.yaml
```
You can navigate to `https://matrix.example.com`. You receive something like this.

![Matrix server](https://raw.githubusercontent.com/jalalsadeghi/matrix-synapse-kubernetes/main/2021-01-19_18-21.png)

## Federation
### Setting up delegation
We want to use `.well-known` to achieve delegation.
When using .well-known another Homeserver will make an HTTPS request to `https://{synapse_server_name}/.well-known/matrix/server.` Which will return the actual domain and port of your homeserver. The same goes for clients except they request `https://{synapse_server_name}/.well-known/matrix/client.`

Note: Before apply 'default.conf', adjust the domain matrix.example.com to the domain Synapse is served through.
```
kubectl apply -f delegation/defaultconf.yaml
```
Apply deployment.
```
kubectl apply -f delegation/deployment.yaml
```
Apply service.
```
kubectl apply -f delegation/service.yaml
```
Apply ingress.
```
kubectl apply -f delegation/ingress.yaml
```
If federation is correctly setup, can be tested with the [Federation Tester](https://federationtester.matrix.org/). 

## Enable VoIP calls
for using voice / video calls on our Synapse server we need to Turnserver ([coturn](https://github.com/coturn/coturn) in this case) in Kubernetes, behind a dynamic IP.
First apply ConfigMap
```
kubectl apply -f coturn/configMap.yaml
```
### sources
my sources for this repository are:
- [Introduction to NGINX Ingress Controller](https://github.com/marcel-dempers/docker-development-youtube-series/blob/master/kubernetes/ingress/controller/nginx/README.md)
- [Installing a Matrix Synapse server on Kubernetes](https://nonedhudla.xyz/installing-synapse-on-kubernetes/)
- [Enable VoIP calls for Matrix Synapse on Kubernetes](https://nonedhudla.xyz/enabling-voip-calls-in-synapse/)
