# Flask and MongoDB on Kubernetes (Minikube)

A Python Flask application connecting to a MongoDB database, hosted on a local Minikube cluster with authentication, persistent volume, autoscaling and resource limits.

## Endpoints

| End Point | Method | Description                                               |
| --------- | ------ | --------------------------------------------------------- |
| `/`       | GET    | Returns "Welcome to the Flask app! The current time is: " |
| `/data`   | POST   | Inserts information from JSON body into MongoDB           |
| `/data`   | GET    | Returns all documents stored in MongoDB                   |

## Project Structure

```

```

flask-mongodb-app/
├── app.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── README.md
└── k8s/
├── mongo-secret.yaml
├── mongo-pv-pvc.yaml
├── mongo-statefulset.yaml
├── mongo-service.yaml
├── flask-deployment.yaml
├── flask-service.yaml
└── flask-hpa.yaml

```

```

## Requirements

- Docker Desktop (active)

- Minikube
- kubectl
- Docker Hub (this project used vedantshrivastav)

---

## 1. Building and pushing Docker Image

````
```bash
# Build
docker build -t vedantshrivastav/flask-mongo-app:v1 .
# Log in to Docker Hub
docker login
# Push
docker push vedantshrivastav/flask-mongo-app:v1
````

```
### Dockerfile
```

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

```
This will install dependencies, this layer will be cached by Docker speeding up re-building.
---
## 2. Deployment on Minikube
### Start cluster
```

```bash
minikube start --driver=docker --cpus=2 --memory=4096
minikube addons enable metrics-server
kubectl get nodes
```

```
This is needed for Horizontal Pod Autoscaler to GET CPU usage.
### Apply all Manifests
```

```bash
kubectl apply -f k8s/
```

```
### Check
```

```bash
kubectl get pods
kubectl get pv,pvc
kubectl get svc
kubectl get hpa
```

```
Should show `mongodb-0` and `flask-app` pods (2 of them) in `1/1 Running`, the PVC should be `Bound` and the HPA should show a CPU %.
### Connecting to Application
Option 1: Using Minikube Service URL
```

```bash
minikube service flask-service --url
```

```
Option 2: Using Port Forwarding
```

```bash
kubectl port-forward service/flask-service 5000:5000
```

```
Testing (Windows uses `curl.exe` in PowerShell):
```

```bash
curl.exe http://localhost:5000/
curl.exe -X POST -H "Content-Type: application/json" -d '{\"sampleKey\":\"sampleValue\"}' http://localhost:5000/data
curl.exe http://localhost:5000/data
```

```
---
## 3. Kubernetes Resources
| File | Resource | Description |
| --- | --- | --- |
| `mongo-secret.yaml` | Secret | Stores MongoDB Username and Password |
| `mongo-pv-pvc.yaml` | PV & PVC | 1Gi storage for MongoDB |
| `mongo-statefulset.yaml` | Statefulset | MongoDB with authentication enabled |
| `mongo-service.yaml` | Service (ClusterIP) | Internal access to MongoDB |
| `flask-deployment.yaml` | Deployment (2 replicas) | Flask Application |
| `flask-service.yaml` | Service (NodePort) | Expose Flask Application |
| `flask-hpa.yaml` | HPA | Scale Flask Application from 2 to 5 replicas based on 70% CPU usage |
### Enabling Authentication
Using the official MongoDB image, if `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD` are set as environment variables authentication will be enabled and a root user will be created. These are obtained from `mongo-secret` Secret. The Flask application connects using the same Secret, specifying `username`, `password` and `authSource="admin"`.
### Persistent Volume
The PVC `mongo-pvc` is mounted to PV `mongo-pv` (Storage Class: `manual`, `hostPath`: `/data/mongo`). This means that MongoDB will store data in `/data/db`. The data will remain even if the pod gets deleted/restarted.
---
## 4. DNS
Every cluster will have a DNS service running (CoreDNS). Every time a Service is created a DNS entry will automatically be created:
```

```
..svc.cluster.local
```

```
For the MongoDB Service in the `default` namespace:
```

```
mongodb-service.default.svc.cluster.local
```

```
How it works:
1. All pods will have a `/etc/resolv.conf` pointing to the DNS server in the cluster. This will also include search domains such as `default.svc.cluster.local`.
2. When the Flask pod attempts to resolve `mongodb-service`, it will automatically append the search domains resulting in the short name being resolved to the long name above.
3. CoreDNS will return the ClusterIP of the Service.
4. `kube-proxy` will route traffic to a healthy pod matching the Service selector (`app: mongodb`).
Pods will have different IPs every time they are recreated. However, the Service will always have the same name and ClusterIP. We therefore connect using the name.
Flask is connected to MongoDB using the Deployment `MONGO_HOST=mongodb-service`, this is passed to `MongoClient(host=...)` in `app.py`. No IP addresses are hardcoded. As the Service type is `ClusterIP`, MongoDB can only be accessed from within the cluster.
---
## 5. Resource Limits and Requests
Resources are what Kubernetes will guarantee to be available to the container. The scheduler will use this to place pods onto nodes. The HPA will also use this to calculate CPU usage as a % of resources.
Limits are the maximum amount of resources a container can use.
If the container uses more than the CPU limit it will be throttled (slowed down). If it uses more than the memory limit it will be killed (`OOMKilled`) and restarted.
This stops one pod starving the others, provides predictable scheduling, provides stability on the node and enables autoscaling based on CPU.
Both Flask and MongoDB have the following values:
| | CPU | Memory |
| --- | --- | --- |
| Request | 200m (0.2 CPU) | 250M |
| Limit | 500m (0.5 CPU) | 500M |
Mongo is started with `--wiredTigerCacheSizeGB 0.25` to ensure the cache doesn’t exceed the memory limit and cause OOMKills.
---
## 6. Design Decisions
| Design Decision | Reason | Alternatives Considered |
| --- | --- | --- |
| Minikube (Docker Driver) | Free, simple and can run locally | kind, k3d, Docker Desktop Kubernetes. Minikube also has simple add-ons such as metrics-server |
| Statefulset for MongoDB | Provides stable pod name (`mongodb-0`) and stable storage. Ideal for databases | Would work with a Deployment if only 1 replica |
| 1 MongoDB Replica | Simple with only 1 PVC | Can provide high availability with a replica set (3 pods). However this requires additional configuration outside the scope of this assignment |
| ClusterIP for MongoDB | Ensures database remains private within the cluster | NodePort/LoadBalancer would expose the DB (not secure) |
| NodePort for Flask | Works with Minikube (no cloud load balancer) | LoadBalancer (requires `minikube tunnel`), Ingress |
| Secret for credentials | Means credentials aren’t stored in the image or in the deployment | Hardcoded in code or as env variables (not secure) |
| Static PV (hostPath) | Meets requirement explicitly | Dynamic provision using default StorageClass (easier but less explicit) |
| HPA (CPU) (autoscaling/v2) | Required by assignment. 70% target, 2-5 replicas | Memory or custom metric scaling |
| `python:3.9-slim` | Small image size and compatible with pinned Flask 2.0.2 | Full Python image (Much bigger), Alpine (Potential issues with building) |
Note: Credentials in `mongo-secret.yaml` are just example values. In production you should use a secret manager or sealed secrets. Never commit passwords to Git. By default, Kubernetes Secrets are not encrypted, just base64 encoded.
---
## 7. Testing
### 7.1 Authentication
No credentials (Fail):
```

```bash
kubectl exec -it mongodb-0 -- mongosh --eval "db.adminCommand('listDatabases')"
```

````
Should return
fails without authentication
c:\Users\lenovo\Pictures\Screenshots\Screenshot 2026-11-12 211331.png
With credentials (Success):

c:\Users\lenovo\Pictures\Screenshots\Screenshot 2026-11-12 211356.png

```bash
kubectl exec -it mongodb-0 -- mongosh -u admin -p password123 --authenticationDatabase admin --eval "db.adminCommand('listDatabases')"
````

```
the list of databases
c:\Users\lenovo\Pictures\Screenshots\Screenshot 2026-11-12 214042.png
### 7.2 Interacting with Database and Persistence
Inserted some records using `POST /data`. Then retrieved them using `GET /data`.
c:\Users\lenovo\Pictures\Screenshots\Screenshot 2026-11-12 223755.png
Deleted MongoDB pod using `kubectl delete pod mongodb-0`.
Once the Statefulset had recreated `mongodb-0`, `GET /data` was run again.
c:\Users\lenovo\Pictures\Screenshots\Screenshot 2026-11-12 224028.png
. This proves that the PV/PVC is keeping the data.
### 7.3 Autoscaling (Simulate High Traffic)
Busybox pods were created in the cluster to continually send requests to the Flask Service:
```

```bash
kubectl run load-generator --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://flask-service:5000/ > /dev/null; done"
```

```
Extra generators were created (`load-generator-2`, `load-generator-3`). This sends requests to the Service so traffic is distributed evenly across all Flask pods. Progress was monitored using:
```

```bash
kubectl get hpa flask-hpa -w
kubectl get pods -w
```

```
Results:
| Stage | CPU (Target: 70%) | Replicas |
| --- | --- | --- |
| Before Load | 1%-8% | 2 |
| Whilst Loading | 144% | 2 -> 4 -> 5 (Max) |
| After Load has finished | Drops to near 0% | 2 (After ~5min stabilisation window) |
HPA scaled up to max amount of 5 replicas. New pods went through `Pending`, `ContainerCreating` and `Running`. Once generators were deleted, HPA scaled back down to min amount of 2 replicas.
(Please add screenshots of HPA before, during and after load and pod list with 5 replicas.)
HPA before the load test

c:\Users\lenovo\Pictures\Screenshots\Screenshot 2026-11-12 212453.png

HPA during the load test
replicas reached to 5

c:\Users\lenovo\Pictures\Screenshots\Screenshot 2026-11-12 212815.png

flask pods  - 5 pods

c:\Users\lenovo\Pictures\Screenshots\Screenshot 2026-11-12 212938.png

HPA after the load
back to 2 replicas

c:\Users\lenovo\Pictures\Screenshots\Screenshot 2026-11-12 213621.png
### 7.4 Issues
| Issue | Cause | Fix |
| --- | --- | --- |
| `curl -X POST ...` failed in PowerShell. Error: Headers parameter | `curl` is an alias for `Invoke-WebRequest` in PowerShell | Use `curl.exe` (or `Invoke-RestMethod`) |
| `mongodb-0` stuck in `Pending`. Error: `persistentvolumeclaim "mongo-pvc" not found` | `mongo-pv-pvc.yaml` was blank (0 bytes). PV/PVC not created | Copied content, saved and reapplied. Checked file sizes using `dir k8s` |
| `mongodb-0` stuck in `ContainerCreating` for a few minutes | Slow initial pull of `mongo:6.0` image | Waited for image to be pulled (Optionally: `minikube image load mongo:6.0`) |
| `minikube` not recognised in terminal | PATH not updated in currently open terminal | Open new terminal or use `kubectl port-forward` to access application |
| HPA showing `` CPU initially | metrics-server requires a short amount of time to initialise | Enable add-on `metrics-server` and wait ~1 minute. |
---
## 8. Clean Up
```

```bash
kubectl delete -f k8s/
minikube stop
# minikube delete   (removes the cluster completely)
```

```

```
