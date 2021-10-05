# Kubernetes (K8)
![overview](https://csharpcorner-mindcrackerinc.netdna-ssl.com/article/getting-started-with-kubernetes-part2/Images/1.png)

## What is Kubernetes
Kubernetes or K8 is an orchestration tool used to manage Containers and Micro-servies. It is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem. Kubernetes services, support, and tools are widely available.

## Advantages
### Service discovery and load balancing 
Kubernetes can expose a container using the DNS name or using their own IP address. If traffic to a container is high, Kubernetes is able to load balance and distribute the network traffic so that the deployment is stable.
### Storage orchestration 
Kubernetes allows you to automatically mount a storage system of your choice, such as local storages, public cloud providers, and more.
### Automated rollouts and rollbacks 
You can describe the desired state for your deployed containers using Kubernetes, and it can change the actual state to the desired state at a controlled rate. For example, you can automate Kubernetes to create new containers for your deployment, remove existing containers and adopt all their resources to the new container.
### Automatic bin packing 
You provide Kubernetes with a cluster of nodes that it can use to run containerized tasks. You tell Kubernetes how much CPU and memory (RAM) each container needs. Kubernetes can fit containers onto your nodes to make the best use of your resources.
### Self-healing 
Kubernetes restarts containers that fail, replaces containers, kills containers that don't respond to your user-defined health check, and doesn't advertise them to clients until they are ready to serve.
### Secret and configuration management 
Kubernetes lets you store and manage sensitive information, such as passwords, OAuth tokens, and SSH keys. You can deploy and update secrets and application configuration without rebuilding your container images, and without exposing secrets in your stack configuration.

## First steps in using K8
1. Having a service broken down into using 1 or 2 microservices
2. Using docker to containerise the microservices
3. Orchestrate the containers using K8

## K8 Cheatsheet
- `kubectl`
- `kubectl --version`
- `kubectl get pods`
- `kubectl get srv/service`
- `kubectl get deploy/deployment`
- `kubectl describe pod {pod_ID}`
- `kubectl delete pod {pod_ID]`
- `kubectl delete svc/service {service_name}`
- `kubectl create -f yaml_file.yaml`
- `kubectl edit deploy {service_name}`
- `kubectl rollout restart deployment`

## Enabling K8
1. Head over to the `Docker Desktop`
2. Press the `Settings` button
3. Navigate to the `kubernetes` section on the left menu bar
4. Tick the `enable kubernetes` tick box and then click `apply and restart`
5. A pop-up should appear, click `install` and now wait for kubernetes to load this may take a while

## Types of Service
- `Cluster IP`: Exposes a service which is only accessible from within the cluster.
- `NodePort`: Exposes a service via a static port on each node's IP.
- `LoadBalancer`:  Exposes the service via the cloud provider's load balancer.
- `ExternalName`: Maps a service to a predefined externalName field by returning a value for the CNAME record.


## Creating a Deployment with K8
```
# KB works with API versions to declare the resources
# We have to declare the apiVersion and the kind of service/component

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment # naming the deployment


spec:
  selector:
    matchLabels:
      app: nginx # look for this label to match with k8 service

  # Lets create a replica set of this with 2 instance/pod
  replicas: 2

  # template to use its labal for K8 service to launch in browser
  template:
    metadata:
      labels:
        app: nginx # This label connects to the service or any other

    # Lets define the container spec
    spec:
      containers:
      - name: nginx
        image: sc18kg/sre_customised_nginx:latest # name of image pulls from dockerhub
        ports:
        - containerPort: 80

```

## Creating a Service in K8
```
---
# Select the type of API version and the type of service/object
apiVersion: v1
kind: Service
# Metadata for name
metadata:
  name: nginx-deployment
  namespace: default

  # Specification to include ports Selector to connector
spec:
  ports:
  - nodePort: 30442
    port: 8080
    protocol: TCP
    targetPort: 8080

# Lets define the selector and label to connect to nginx
  selector:
    app: nginx # this label connects this service to deployment

  # Creating LoadBalancer type of deployment
  type: LoadBalancer

```
## AWS to Run image - minikube

### Install minikube and start the server
1. `minikube start --vm-driver=none`
2. `minikube status`

### Run the images as services
3. `kubectl create -f nginx-deployment.yaml` - This will create the pods and the number of replicas set in the `.yaml` file
4. `kubectl run nginx-deployment --image=sc18kg/sre_customised_nginx --port=8080`
5. `kubectl expose deployment nginx-deployment --type=NodePort` - This will create a service for the pods to be exposed

### Check the service is running 
6. `kubectl get services`

### Check security of AWS EC2
The `Port` will be shown next to the `Port 8080` so add this port to your inbound rules on the EC2 instance 
`Type`: Custom TCP Rule
`Protocol`: TCP
`Port Range`: 30000-32767 (the port given to you by the kubectl get services command)
`Source`: Custom 0.0.0.0/0 (Accessible via the internet)

### Check the browser
Head to the address `<ipv4_public_ip>:<ec2_port>`.

## NodeApp and Database
![Structure](https://amlanscloud.com/static/bbc5a55e5f99dd01781ba3fd3c2ac32c/88ed5/kubernetes_archi.png)

## Creating a node-deploy.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node

spec:
  selector:
    matchLabels:
      app: node
  replicas: 3

  template:
    metadata:
      labels:
        app: node
    spec:
      containers:
        - name: node
          image: ahskhan/eng89_node_prod

          ports:
            - containerPort: 3000

```
To run this use `kubectl create -f node_deploy.yaml`

## Create Horizontal Scaling

```
# Create a file to create an auto scaling group
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler

metadata:
  name: sparta-node-app-deploy
  namespace: default

spec:
  maxReplicas: 9
  minReplicas: 3
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: node

  targetCPUUtilizationPercentage: 50

```
Run `kubectl create -f node_hpa.yaml` then to check use `kubectl get hpa` to check its live

## Create deploy and service for Mongo
### Service
```
# Create the Service
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec: 
  selector:
    app: mongo
  ports:
    - port: 27017
      targetPort: 27017
```
### Deployment
```
# Create the Deployment
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
spec:
  selector:
    matchLabels:
      app: mongo
  replicas: 1
  template:
    metadata:
      labels:
        app: mongo
        
    spec:
      containers:
        - name: mongo
          image: mongo:latest
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: storage
              mountPath: /data/db
      volumes:
        - name: storage
          persistantVolumeClaim:
            claimName: mongo-db
```
## Create Persistant Volume and PV Claim
```
---
apiVersion: v1
kind: PersistantVolumeClaim
metadata:
  name: mongo-db
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
```
## Connect the NodeApp and DB

### Once connected
Run the command `kubectl exec node env node seeds/seed.js` which will seed the database on the `/posts` page
