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
kubectl apply -f configMap.yaml
#Deploys the PersistentVolume
kubectl apply -f pv.yaml
#Deploys the StatefulSet
kubectl apply -f statefulSet.yaml
#Deploys the Service
kubectl apply -f service.yaml
```

```
kubectl get Configmap
kubectl get statefulset
kubectl get pv
kubectl get service
```

```
kubectl apply -f pv.yaml
kubectl apply -f deployment.yaml
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
