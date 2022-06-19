# jgraux
Kube manifests and instructions to deploy grafana with influxdb in cluster. And use in jmeter to test your apps.

### In this short tutorial, I will show you how to quickly deploy grafana and influxdb to your kubernetes cluster. Get access to them from outside the cluster and use them for load tests of your web applications with JMeter.

### I am using k3s cluster, you can find out how to install it here: https://rancher.com/docs/k3s/latest/en/installation/

### I will use `jmeter-test` namespace, you can create it:
```
kubectl create namespace jmeter-test
```

**Hint: set alias kj to kubectl -n jmeter-test**
```
alias kj="kubectl -n jmeter-test"
```

# Deploy influxdb.

## Step 1. Create enviromental variables use secret.

```
kubectl -n jmeter-test create secret generic influxdb-creds \
  --from-literal=INFLUXDB_DATABASE=jmeter \
  --from-literal=INFLUXDB_USERNAME=root \
  --from-literal=INFLUXDB_PASSWORD=root \
  --from-literal=INFLUXDB_HOST=influxdb
```

You can save secret to local yaml file:
```
kubectl -n jmeter-test get secret
kubectl -n jmeter-test get secret influxdb-creds -o yaml > influxdb-secret.yaml
```

## Step 2. Create PV and PVC for influxdb
We need to store influxdb data somewhere.
Create two files: `influxdb-pv.yaml` and `influxdb-pvc.yaml`.
Open `influxdb-pv.yaml` and paste in this:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: influxdb
spec:
  storageClassName: local-path
  volumeMode: Filesystem
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /var/influxdb/
```
capacity you can change and path in hostPath too.
**Do not use hostPath in production. This can serve as a very good security hole.**

Open `influxdb-pvc.yaml` and paste in this:
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: influxdb-claim
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 8Gi
```

## Step 3. Create influxdb deployment.
Create file `influxdb-deployment.yaml`, open it and paste in:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: influxdb
spec:
  selector:
    matchLabels:
      app: influxdb
  template:
    metadata:
      labels:
        app: influxdb
    spec:
      containers:
      - name: influxdb
        image: docker.io/influxdb:latest
        envFrom:
        - secretRef:
           name: influxdb-creds
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
      volumes:
      - name: var-lib-influxdb
        persistentVolumeClaim:
          claimName: influxdb-claim
```

## Step 4. Create service for influxdb

Create one more file with name `influxdb-service.yaml`.
Open it and paste in:

```
apiVersion: v1
kind: Service
metadata:
  name: influxdb-service
spec:
  selector:
    app: influxdb
  ports:
  - port: 8086
    protocol: TCP
    targetPort: 8086
```

## Step 5. Create ingress for influxdb.

Create file `influxdb-ingress.yaml`, open and paste in.
**Rename host to your domain**
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: influx-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
  - host: <host>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: influxdb
            port:
              number: 8086
```

## Step 6 (Optional). Create nodePort service for influx.
Create nodePort service for influx.
Create file `influxdb-np.yaml`
```
apiVersion: v1
kind: Service
metadata:
  name: influxdb-np
spec:
  type: NodePort
  selector:
    app: influxdb
  ports:
  - port: 8086
    targetPort: 8086
    nodePort: 30001
```

## Step 7. Deploy influxdb

Go to path where all created files located.
`kubectl -n jmeter-test apply -f .`
You will see:
```
deployment.apps/influxdb created
persistentvolume/influxdb created
persistentvolumeclaim/influxdb-claim create
service/influxdb created
service/influxdb-np created
```

Then run `kubectl -n jmeter-test get pod` and
make sure influxdb is running.
```
NAME                       READY   STATUS    RESTARTS   AGE
influxdb-74df469dd-bwqng   1/1     Running   0          7s
```

# Deploy grafana

## Step 1. Create secret for grafana

Create secret for grafana.
```
k create secret generic grafana-creds \
  --from-literal=GF_SECURITY_ADMIN_USER=admin \
  --from-literal=GF_SECURITY_ADMIN_PASSWORD=grafana \
```
### **Attention**.
If you want to change root url for grafana, add these:
```
--from-literal=GF_SERVER_ROOT_URL=<protocol>://<your domain>:<port>/grafana \
--from-literal=GF_SERVER_SERVE_FROM_SUB_PATH='true'
```
Because of these envs you will access grafana with url, for example: `http://<your domain>:<port>/grafana`

You can save secret to local yaml file:
```
kubectl -n jmeter-test get secret
kubectl -n jmeter-test get secret grafana-creds -o yaml > grafana-secret.yaml
```

## Step 2. Create grafana deployment.

Create file `grafana-deployment.yaml`. Open it and paste in:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: docker.io/grafana/grafana:latest
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        envFrom:
        - secretRef:
            name: grafana-creds
      restartPolicy: Always
```

## Step 3. Create service for grafana

Create file `grafana-service.yaml` and paste in:
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: grafana
  name: grafana
spec:
  selector:
    app: grafana
  ports:
    - port: 3000
      protocol: TCP
      targetPort: 3000
```

## Step 4. Create ingress for grafana

Create file `grafana-service.yaml` and paste in.
**Rename host to your host.**
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  labels:
    name: grafana
spec:
  rules:
  - host: <Host>
    http:
      paths:
      - pathType: Prefix
        path: /grafana
        backend:
          service:
            name: grafana
            port:
              number: 3000

```

## Step 5. Apply.

Go to path where all created files located.
`kubectl -n jmeter-test apply -f .`


# Finalize deploy.

### Now you can access `http://<domain>/grafana` to visit grafana and `http://<domain>/` for influxdb admin panel.

# Install and configure JMeter

### You can use this tool to make load test for you endpoint/endpoint.

## Step 1. Install

Go to https://jmeter.apache.org/download_jmeter.cgi and install jmeter

## Step 2. Get required template.

Go to `templates` --> select `JDBC Load Test` --> `Select`.

## Step 3. Install plugin for Flux query language.

Go to https://grafana.com/grafana/dashboards/13644 and install plugin `jmeter-plugins-influxdb2-listener-<>.jar`

## Step 4. Configure JMeter

Go to the `Thread group` --> enter name and set as in image. 
![image](https://user-images.githubusercontent.com/62915291/174488618-02338437-c73d-4093-953b-2c26a06490b3.png)

Now right click on the left side on thread group --> `Add` --> `Listener` --> `Backend Listener`.  
Go to `Backend Listener implementation` and select ![image](https://user-images.githubusercontent.com/62915291/174488697-fa517645-4748-4540-a50b-e0215edcba39.png)  

Go to `Backend Listener implementation` and select ![image](https://user-images.githubusercontent.com/62915291/174488697-fa517645-4748-4540-a50b-e0215edcba39.png)  

Then you have to change  
1) `influxDBHost` set you host
2) `influxDBPort` (we created 30001) if not changed dont change  
3) `influxDB` (we created jmeter)  if not changed dont change  
4) `influxDBOrganization` --> go to influxdb admin panel --> `Me` --> OrganizationID  
5) `influxDBToken` --> go to influxdb admin panel --> API tokens --> copy and paste
![image](https://user-images.githubusercontent.com/62915291/174488723-ad808b6b-06a0-441c-a09f-194ed5243c03.png)

Also you have go to the HTTP request page in JMeter and apply you http request. I think you can do it by yourself.  

That's all with JMeter.

# Configure Grafana.

## Part 1. Create datasource.

Just look at the screen and paste your data  
![image](https://user-images.githubusercontent.com/62915291/174489044-8afc186c-3b08-4ff6-ad35-9c164fdff8fd.png)

## Part 2. Add dashboard.  

Download json here - https://grafana.com/grafana/dashboards/13644.

Go to ![image](https://user-images.githubusercontent.com/62915291/174489065-15e64d32-12d0-407f-b1fb-b754a223a386.png)

And import json file. After select our datasource.

## That's all.
