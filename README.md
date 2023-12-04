# Kubernetes  Lab

The aim of this lab is to provide an understanding of advanced Kubernetes features. It covers deployment strategies, service management, and autoscaling within Google Kubernetes Engine (GKE). 


### 1. Initialize a Google Kubernetes Engine (GKE) Cluster

- Launch the Google Cloud Console.
- Select the project to work on. 
- From the side menu, proceed to "Kubernetes Engine" and then to "Clusters."
- Select the "Create Cluster" option. 
- Assign and adjust the cluster parameters such as the cluster name, zone, node configuration, etc. 
- Press "Create" to finalize the creation of your cluster.

### 2. Containerizing a Python Application with Docker
1. Write a Dockerfile to containerize the python code and its corresponding requirements file.

  ```
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8080

CMD ["python", "app.py"]

```


- `FROM python:3.9-slim`: Use the Python 3.9 slim image as a base to create a leaner image that contains only the essential packages.

- `WORKDIR /app`: Establish `/app` as the directory for any subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, and `ADD` Dockerfile commands.

- `COPY requirements.txt .`: Copy the `requirements.txt` file from the local directory to the current working directory in the container (denoted by `.`).

- `RUN pip install --no-cache-dir -r requirements.txt`: Run pip install commands to install the dependencies listed in `requirements.txt`, using the `--no-cache-dir` option to prevent caching and reduce image size.

- `COPY app.py .`: Transfer the `app.py` file from the local directory to the container's working directory.

- `EXPOSE 8080`: Indicate that the container will listen to port 8080 at runtime. Note that this doesn't automatically expose the port.

- `CMD ["python", "app.py"]`: Define the default command to execute `app.py` with Python when the container starts up.

### 3. Build and Push Docker Image to Google Container Registry (GCR)
- Authenticate with the Google Cloud SDK to gain access to GCP services. 
- Use gcloud auth login to authenticate your local machine with your Google Cloud account, opening up access to your GCP resources via the command line.
```
gcloud auth login
```
- Build your Docker image and tag it for the Google Container Registry: Execute the below command to create a Docker image from your current directory's Dockerfile and tag it with a version and project path suitable for upload to GCR.
```
docker buildx build --platform linux/amd64 -t gcr.io/airflow-github/mlops:firstbuild -f Dockerfile .
```
- Upload the Docker image to your Google Container Registry: Use the below command to push the tagged image to GCR, making it available for deployment within your GCP projects.
```
docker push gcr.io/airflow-github/mlops:firstbuild
```
### 4. Kubernetes Deployment

Create a Kubernetes Deployment manifest to leverage Rolling Updates for enhanced deployment approaches.

Manifest for Rolling Updates Deployment:

- Employ a Rolling Update deployment tactic to facilitate a seamless transition to new application versions. 
- This strategy incrementally replaces the old version with the new one, thereby minimizing service interruptions and ensuring uninterrupted availability during updates.


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlops-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mlops
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: mlops
    spec:
      containers:
      - name: mlops
        image: gcr.io/airflow-github/mlops:firstbuild
```

- `apiVersion: apps/v1`: Specifies the API version of Kubernetes used for the Deployment manifest.
- `kind: Deployment`: Declares that this manifest is for creating a Kubernetes Deployment.
- `metadata: name: mlops-deployment`: Defines the name of the deployment as `mlops-deployment`.
- `spec: replicas: 3`: Sets the desired number of replica pods for the deployment to 3.
- `selector: matchLabels: app: mlops`: Uses label selectors to find which pods to manage and update with this deployment strategy.
- `strategy: type: RollingUpdate`: Specifies the deployment strategy as `RollingUpdate`.
- `rollingUpdate: maxUnavailable: 1`: During the update, ensures that at least two replicas of the application are available at all times.
- `rollingUpdate: maxSurge: 1`: Allows one extra pod to be created above the desired number of replicas during the update.
- `template: metadata: labels: app: mlops`: Describes the pod templates used by the deployment, labeling the pods with `app: mlops`.
- `spec: containers: - name: mlops`: Defines the container specifications within the pod, setting the name for the container as `mlops`.
- `image: gcr.io/airflow-github/mlops:firstbuild`: Specifies the container image to use for the pod, pointing to the `firstbuild` tag of the `mlops` image in Google Container Registry.


### 5. Create Horizontal Pod Autoscaler (HPA)
The Horizontal Pod Autoscaler (HPA) is a feature in Kubernetes that automatically scales the number of pod replicas in a deployment, replication controller, stateful set, or replica set based on observed CPU utilization or other select metrics. The HPA adjusts the number of replicas dynamically to match the workload demands, with the goal of maintaining performance and resource efficiency.
    - Automatic Scaling: HPA adjusts pod counts based on demand, automating scalability for fluctuating traffic.
- `Resource Efficiency`: It optimizes resource use, preventing wastage through dynamic scaling, which can reduce costs.
- `Performance Management`: HPA scales resources to maintain service quality during traffic surges.
- `Downtime Reduction`: It proactively manages scaling to prevent outages from overloading resources.
- `Ease of Management`: HPA simplifies operations by replacing manual scaling with automated, policy-driven adjustments.
- `Flexibility`: Offers custom scaling options through user-defined metrics for tailored resource management.
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: mlops-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mlops-deployment
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

- `apiVersion: autoscaling/v1`: Indicates the version of the Kubernetes autoscaling API being used for the HPA resource.
- `kind: HorizontalPodAutoscaler`: Specifies the resource type as HPA, which automatically scales the number of pod replicas.
- `metadata: name: mlops-hpa`: Sets the name of the Horizontal Pod Autoscaler to `mlops-hpa`.
- `spec: scaleTargetRef: ...`: Points to the target resource (like a Deployment) that HPA will scale, in this case, `mlops-deployment`.
- `minReplicas: 1`: Configures HPA to maintain a minimum of one replica of the pod.
- `maxReplicas: 10`: Allows HPA to scale up to a maximum of ten pod replicas.
- `targetCPUUtilizationPercentage: 50`: Tells HPA to initiate scaling actions to maintain an average CPU utilization across all pods at 50%.

### 6. Configure Kubernetes Services
Kubernetes Service manifests enable a reliable network interface for a dynamic set of Pods, facilitating stable application exposure by managing traffic routing and service discovery as Pods change, scale, or update. They provide an essential linkage between deployed applications and users or other services, maintaining accessibility through internal and external endpoints.

```
apiVersion: v1
kind: Service
metadata:
  name: mlops-service
spec:
  selector:
    app: mlops
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
```
- `apiVersion: v1`: Specifies the version of the Kubernetes API this Service is using.
- `kind: Service`: Defines the resource as a Service, which manages network access to a set of Pods.
- `metadata: name: mlops-service`: Sets the name of the Service to `mlops-service` for reference within the Kubernetes cluster.
- `spec: selector: app: mlops`: Determines which Pods will be part of this Service based on the `app: mlops` label.
- `ports: - protocol: TCP`: Configures the Service to use TCP for the ports that are being exposed.
- `port: 80`: The Service will be exposed on port 80 to the outside of the cluster.
- `targetPort: 8080`: Service will route to `targetPort` 8080 on the Pods, which is the port on which the Pods' application is actually running.
- `type: LoadBalancer`: Specifies the Service type as LoadBalancer, which will provision an external IP to handle incoming traffic and route it to the Pods.

### 7.  Implementing Persistent Storage

`Persistent Storage`: Pods in Kubernetes are ephemeral, which means they can be stopped and started at any time. When a pod dies, the data stored on its file system is lost. PVs provide a way to use durable storage that remains consistent regardless of pod lifecycle events, ensuring data persistence.

`Decoupling Storage Configuration`: PVs and PVCs separate storage configuration from the actual use in pods, allowing developers to consume storage resources without knowing the details of the underlying storage infrastructure.

`Storage Abstraction`: Using PVCs and PVs allows you to abstract the details of how storage is provided. The same Kubernetes manifests can be used across development, staging, and production without changes, even if the underlying storage implementations differ.

`Dynamic Provisioning`: Dynamically provisioning PVs through storage classes simplifies the process of deploying and managing applications that require persistent storage. It automates the process of storage creation, allowing you to manage storage with Kubernetes API calls rather than manual storage operations.


```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd

```

- `StorageClass`: Defines a type of storage provided by the cloud. By specifying type: pd-ssd, you're requesting SSD-backed storage, which offers better performance than standard hard disk drives.
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-storage
```
- `PersistentVolumeClaim`: Acts as a request for storage by a user. You specify the size (10Gi) and the access modes (like ReadWriteOnce, which means the volume can be mounted as read-write by a single node). The storageClassName: fast-storage tells Kubernetes to provision this storage using the fast-storage StorageClass, which is configured to create an SSD disk.
```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - mountPath: "/var/www/html"
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc

```
- `Pod Configuration Using PVC`: Shows how to mount the provisioned storage inside a container in a pod. The volumeMounts section of the pod definition specifies the path where the volume should be mounted inside the container, allowing the application to read and write data to the persistent storage.

### Sequence of Execution

1. **Set up GKE Cluster:**

2. **Create and Dockerize Application:**

3. **Build and Push Docker Image to GCR:**
   - Use the Docker CLI to build your Docker image and then push it to Google Container Registry (GCR).

4. **Create Kubernetes Manifests:**
   - Write your Kubernetes manifests, including the deployment, service, HPA, PV, and PVC files. Save these as `.yaml` files.

5. **Deploy Persistent Volume (PV) and PersistentVolumeClaim (PVC):**
   - Apply the PV and PVC before deploying your application to ensure the storage is available when the application starts.
   ```bash
   kubectl apply -f storage.yaml  
   kubectl apply -f pvc.yaml
   kubectl apply -f pod.yaml
   ```

6. **Deploy Application:**
   ```bash
   kubectl apply -f deployment.yaml
   ```

7. **Expose Application with a Service:**
   ```bash
   kubectl apply -f service.yaml
   ```

8. **Implement Autoscaling:**
   ```bash
   kubectl apply -f hpa.yaml
   ```

9. **Monitor Application and Autoscaler:**
   - Monitor your application's deployment, service status, and HPA behavior.
   ```bash
   kubectl get deployment
   kubectl get services
   kubectl get hpa
   ```

10. **Test Application and Autoscaling:**
    - Once everything is deployed and exposed, you can test the application's functionality and autoscaling behavior by generating load.
```bash
for i in {1..10000}; do
  curl http://<your-service-external-ip>/ &
done
wait
```
