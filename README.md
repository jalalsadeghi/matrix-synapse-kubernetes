# matrix-synapse

```
cd matrix-synapse
```
set a secure password for the user in postgres/configMap.yaml
```
nano postgres/configMap.yaml
```

```
#Deploys the ConfigMap
sta configMap.yaml
#Deploys the PersistentVolume
kubectl apply -f pv.yaml
#Deploys the StatefulSet
kubectl apply -f statefulSet.yaml
#Deploys the Service
kubectl apply -f service.yaml
```

```
kubectl apply -f pv.yaml
kubectl apply -f deployment.yaml
```


```
kubectl get Configmap
kubectl get statefulset
kubectl get pv
kubectl get service
kubectl get deployment
kubectl get ingress
```
```
kubectl delete Configmap
kubectl delete statefulset
kubectl delete pv
kubectl delete service
kubectl delete deployment
kubectl delete ingress
```

Next delete the section database: and remove the beginning # for the example Postgres section. Here we now fill in the Database connection details. The section should look like
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

Now before starting Synapse again the Deployment has to be adjusted. To do so edit the deployment.yaml and delete the following
```
args: ["generate"]
env:
- name: SYNAPSE_SERVER_NAME
    value: "example.com"
- name: SYNAPSE_REPORT_STATS
    value: "yes"
```

```
sudo certbot certonly --standalone --preferred-challenges http -d min-wave.com

kubectl create secret tls min-wave-tls-secret --cert=/etc/letsencrypt/live/min-wave.com/fullchain.pem --key=/etc/letsencrypt/live/min-wave.com/privkey.pem
```