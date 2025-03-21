# EasyShop Deployment Guide


This guide will walk you through the step-by-step process of deploying the EasyShop application on a Kubernetes cluster.

## Prerequisites
- A running Kubernetes cluster (Minikube, Kind, or a cloud provider like GKE, EKS, AKS)
- `kubectl` installed and configured
- `helm` installed (if using Helm charts)
- Docker installed and configured
- Ingress controller installed (NGINX Ingress recommended)

---

## Project dir structure:
This is just a preview want more details, then use `tree` inside the repo.
```bash
.
├── Dockerfile
├── Dockerfile.dev
├── JENKINS.md
├── Jenkinsfile
├── LICENSE
├── README.md
├── components.json
├── docker-compose.yml
├── ecosystem.config.cjs
├── kubernetes
│   ├── 00-kind-config.yaml
│   ├── 01-namespace.yaml
│   ├── 02-mongodb-pv.yaml
│   ├── 03-mongodb-pvc.yaml
│   ├── 04-configmap.yaml
│   ├── 05-secrets.yaml
│   ├── 06-mongodb-service.yaml
│   ├── 07-mongodb-statefulset.yaml
│   ├── 08-easyshop-deployment.yaml
│   ├── 09-easyshop-service.yaml
│   ├── 10-ingress.yaml
│   ├── 11-hpa.yaml
│   └── 12-migration-job.yaml
├── next.config.cjs
├── next.config.js
├── package-lock.json
├── package.json
├── postcss.config.js
├── public
├── scripts
│   ├── Dockerfile.migration
│   ├── migrate-data.ts
│   └── tsconfig.json
├── src
│   ├── app
│   ├── data
│   ├── lib
│   ├── middleware.ts
│   ├── styles
│   ├── types
│   └── types.d.ts
├── tailwind.config.ts
├── tsconfig.json
└── yarn.lock
```
## 🛠️ Environment Setup

### 1. Clone the Repository
```sh
 git clone https://github.com/mukeshchaudhary14/easyshop.git
 cd easyshop/
```
### 2. Build and Push Docker Images
> 
> #### 2.1 Login to Docker Hub
> First, login to Docker Hub (create an account at [hub.docker.com](https://hub.docker.com) if you haven't):
>    ```bash
>    docker login
>    ```
>
> #### 2.2 Build Application Image
>    ```bash
>    # Build the application image
>    docker build -t your-dockerhub-username/easyshop:latest .
> 
>    # Push to Docker Hub
>    docker push your-dockerhub-username/easyshop:latest
>    ```
>
> #### 2.3 Build Migration Image
>    ```bash
>    # Build the migration image
>    docker build -t your-dockerhub-username/easyshop-migration:latest -f Dockerfile.migration .
> 
>    # Push to Docker Hub
>    docker push your-dockerhub-username/easyshop-migration:latest
>    ```
## Kind Cluster Setup

### 3. Make a directory in EasyShop 
>    ```sh
>    mkdir k8s
>    cd k8s
>    ```
> #### 3.1 Create new cluster
> Create `00-kind-config.yaml` with the following content:
>```zsh
>  vim 00-kind-config.yaml
>```
```yaml
  kind: Cluster
  apiVersion: kind.x-k8s.io/v1alpha4
  nodes:
    - role: control-plane
      kubeadmConfigPatches:
        - |
          kind: InitConfiguration
          nodeRegistration:
            kubeletExtraArgs:
              node-labels: "ingress-ready=true"
      extraPortMappings:
        - containerPort: 80
          hostPort: 80
          protocol: TCP
        - containerPort: 443
          hostPort: 443
          protocol: TCP
 ```


 ```bash
  kind create cluster --name easyshop --config kubernetes/00-kind-config.yaml
 ```
##### This command creates a new Kind cluster using our custom configuration with one control plane and two worker nodes.
### 4.  Create namespace
Create `01-namespace.yaml ` with the following content:
 ```zsh
  vim 01-namespace.yaml 
  ```
 ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: easyshop
    labels:
      name: easyshop
      environment: production
```
##### Apply a separate namespace for the application:
  ```zsh
  kubectl apply -f kubernetes/01-namespace.yaml
   ```
   
 ### 5.  Create & Setup storage
   ##### 5.1  Create `02-mongodb-pv.yaml ` with the following content:
```zsh
  vim 02-mongodb-pv.yaml
 ```
```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: mongodb-pv
  spec:
    capacity:
      storage: 5Gi
    accessModes:
      - ReadWriteOnce
    hostPath:
      path: /data/mongodb
    persistentVolumeReclaimPolicy: Retain
```
##### Apply PersistentVolume for the application:
```zsh
  kubectl apply -f  02-mongodb-pv.yaml
```
##### 5.2  Create `03-mongodb-pvc.yaml ` with the following content:
```zsh
  vim 03-mongodb-pvc.yaml 
```
```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: mongodb-pvc
    namespace: easyshop
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi
```

##### Apply PersistentVolumeClaim for the application:
```zsh
  kubectl apply -f  03-mongodb-pvc.yaml  
```
### Generating Secure Secrets for Deployment
####  When deploying to EC2, replace your-ec2-ip with your actual EC2 instance address.
####  To generate secure secret keys, use these commands in your terminal:
```zsh
# For NEXTAUTH_SECRET
openssl rand -base64 32

# For JWT_SECRET
openssl rand -hex 32
 ```
### 6. Create ConfigMap
Create `04-configmap.yaml `  with the following content:
   ```zsh
  vim 04-configmap.yaml 
   ```
```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: easyshop-config
    namespace: easyshop
  data:
    MONGODB_URI: "mongodb://mongodb-service:27017/easyshop"
    NODE_ENV: "production"
    NEXT_PUBLIC_API_URL: "http://YOUR_EC2_PUBLIC_IP/api"  # Replace with your YOUR_EC2_PUBLIC_IP
    NEXTAUTH_URL: "http://YOUR_EC2_PUBLIC_IP"             # Replace with your YOUR_EC2_PUBLIC_IP
    NEXTAUTH_SECRET: "KU4WleadP0sQE+PWkTxUeWN3zBQNZ4suxJnO1davpOo="
    JWT_SECRET: "e2e358d3a115c97e6c2d7df1dc8d0fa5dc6a14f82ecceac0e947eb9f5bea6056"
```
  ##### Apply ConfigMap for the application:
   ```zsh
   kubectl apply -f  04-configmap.yaml 
   ```

 ### 7. Create secrets
 Create `05-secrets.yaml  `  with the following content:
   ```zsh
   vim 05-secrets.yaml 
   ```
```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: easyshop-secret
    namespace: easyshop
  type: Opaque
  stringData:
    JWT_SECRET: "change-this-in-production"
    NEXTAUTH_SECRET: "change-this-in-production"
```
 ##### Apply Secret for the application:
   ```zsh
 kubectl apply -f 05-secrets.yaml 
   ```
   
 ### 8. Create mongodb-service.yaml
 Create `06-mongodb-service.yaml  `  with the following content:
   ```zsh
   vim 06-mongodb-service.yaml
   ```
```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: mongodb-service
    namespace: easyshop
  spec:
    ports:
      - port: 27017
        targetPort: 27017
    selector:
      app: mongodb
```
 ##### Apply mongodb-service.yaml for the application:
   ```zsh
 kubectl apply -f 06-mongodb-service.yaml
   ```

  ### 9. Create mongodb-statefulset.yaml
Create ` 07-mongodb-statefulset.yaml  `  with the following content:
   ```zsh
   vim  07-mongodb-statefulset.yaml
   ```
```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: mongodb
    namespace: easyshop
  spec:
    serviceName: mongodb-service
    replicas: 1
    selector:
      matchLabels:
        app: mongodb
    template:
      metadata:
        labels:
          app: mongodb
      spec:
        containers:
          - name: mongodb
            image: mongo:latest
            ports:
              - containerPort: 27017
            volumeMounts:
              - name: mongodb-storage
                mountPath: /data/db
            resources:
              requests:
                memory: "256Mi"
                cpu: "250m"
              limits:
                memory: "512Mi"
                cpu: "500m"
        volumes:
          - name: mongodb-storage
            persistentVolumeClaim:
              claimName: mongodb-pvc
```
 ##### Apply  mongodb-statefulset.yaml for the application:
   ```zsh
 kubectl apply -f  07-mongodb-statefulset.yaml
  ```

 ### 10. Create easyshop-deployment.yaml
 Create ` 08-easyshop-deployment.yaml  `  with the following content:
   ```zsh
   vim  08-easyshop-deployment.yaml
   ```
```yaml
apiVersion: v1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: easyshop
  namespace: easyshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: easyshop
  template:
    metadata:
      labels:
        app: easyshop
    spec:
      containers:
        - name: easyshop
          image: makug/easyshop:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: easyshop-config
            - secretRef:
                name: easyshop-secrets
          env:
            - name: NEXTAUTH_URL
              valueFrom:
                configMapKeyRef:
                  name: easyshop-config
                  key: NEXTAUTH_URL
            - name: NEXTAUTH_SECRET
              valueFrom:
                secretKeyRef:
                  name: easyshop-secrets
                  key: NEXTAUTH_SECRET
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: easyshop-secrets
                  key: JWT_SECRET
          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          startupProbe:
            httpGet:
              path: /
              port: 3000
            failureThreshold: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 20
            periodSeconds: 15
          livenessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 25
            periodSeconds: 20

```
 ##### Apply  easyshop-deployment.yaml for the application:
   ```zsh
 kubectl apply -f  08-easyshop-deployment.yaml
   ```

### 11. Create  easyshop-service.yaml
 Create ` 09-easyshop-service.yaml  `  with the following content:
   ```zsh
   vim  09-easyshop-service.yaml
   ```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: easyshop-service
  namespace: easyshop
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30000
  selector:
    app: easyshop
```
 ##### Apply  easyshop-service.yaml for the application:
   ```zsh
 kubectl apply -f  09-easyshop-service.yaml
   ```

### 12. Create  ingress.yaml
 Create ` 10-ingress.yaml `  with the following content:
   ```zsh
   vim  10-ingress.yaml
   ```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: easyshop-ingress
  namespace: easyshop
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  rules:
    - host: "192.168.142.132.nip.io"      #Replace with your YOUR_EC2_PUBLIC_IP and add extension od .nip.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: easyshop-service
                port:
                  number: 80
```
##### Apply  ingress.yaml for the application:
   ```zsh
 kubectl apply -f  10-ingress.yaml
   ```

### 13. Create  hpa.yaml
 Create ` 11-hpa.yaml  `  with the following content:
   ```zsh
   vim  11-hpa.yaml
   ```
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: easyshop-hpa
  namespace: easyshop
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: easyshop
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```
 ##### Apply  easyshop-service.yaml for the application:
   ```zsh
 kubectl apply -f  11-hpa.yaml
   ```

### 14. Create  migration-job.yaml
 Create ` 12-migration-job.yaml  `  with the following content:
   ```zsh
   vim  12-migration-job.yaml
   ```
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: easyshop
spec:
  template:
    spec:
      containers:
        - name: migration
          image: makug/easyshop-migration:latest     #replace with your dockerhub image
          imagePullPolicy: Always
          env:
            - name: MONGODB_URI
              value: "mongodb://mongodb-service:27017/easyshop"
      restartPolicy: OnFailure
```
 ##### Apply  easyshop-service.yaml for the application:
   ```zsh
 kubectl apply -f  12-migration-job.yaml
   ```

### 15.Verify that the pods are running:
```sh
kubectl get pods -n easyshop
```
 ### Expected output:
 ```sh
    NAME                      READY   STATUS    RESTARTS   AGE
    easyshop-7478769d-h5qqg   1/1     Running   0          13s
    easyshop-7478769d-jprc7   1/1     Running   0          13s
    mongodb-0                 1/1     Running   0          13s
 ```

### 16.Expose Application Using Ingress
   ##### Check if the Ingress is configured properly:
```sh
kubectl get ingress -n easyshop
```
### Expected output:
```sh
NAME               CLASS    HOSTS                    ADDRESS   PORTS   AGE
easyshop-ingress   <none>   192.168.142.132.nip.io             80      2m
```
### 17.Verify the Deployment:
  #### To access the application, open your browser and visit:
```sh
  http://192.168.142.132.nip.io/
```
### If the application does not load, check:
```sh
kubectl describe ingress easyshop-ingress -n easyshop
kubectl get pods -n easyshop
kubectl logs <pod-name> -n easyshop
```

### 18.Port Forwarding (Optional)
  #### If the Ingress is not working, try port forwarding:
  ```sh
    kubectl port-forward svc/easyshop-service 8080:80 -n easyshop
  ```
  #### Then open:
  ```sh
    http://localhost:8080/
  ```


## Troubleshooting
1. If pods are not starting, check logs:
   
   ```bash
   kubectl logs -n easyshop <pod-name>
    ```
2. For MongoDB connection issues:
   
   ```bash
   kubectl exec -it -n easyshop mongodb-0 -- mongosh
    ```
3. To restart deployments:
   
   ```bash
   kubectl rollout restart deployment/easyshop -n easyshop 
    ```
## Cleanup
To delete the cluster:

```bash
kind delete cluster --name easyshop
 ```

## Conclusion

#### Your EasyShop application is now successfully deployed on Kubernetes. 🎉 If you face any issues, refer to the troubleshooting steps or check the logs.
#### Happy Coding! 🚀  
