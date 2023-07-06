# twoge_k8s

The purpose of this repository is to gain experience using Kubernetes by deploying the Twoge application through Kubernetes. The Twoge application will first be deployed on a local Kubernetes cluster through Minikube, then deployed utilizing AWS Elastic Kubernetes Service (EKS).
Source files for the Twoge application can be found at https://github.com/chandradeoarya/twoge/tree/k8s

Architecture Diagram:



***Step 1: Create a Docker Image of the Twoge Application***

The first step in deploying the Twoge application through Kubernetes is to first dockerize the application by defining and building a Dockerfile to run the application within a container

Clone the application's repository onto the local system then change over to the new directory and create the Dockerfile. 
```
git clone https://github.com/chandradeoarya/twoge.git
cd twoge
touch Dockerfile
```
Build a Postgres database using Amazon RDS for initial testing of the Twoge Application. Configure attached security group to allow inbound traffic over port 5432 and 22. Take note of the database server's endpoint URL and the specified user/password/database information for use later within the environment variable. 

<img width="679" alt="image" src="https://github.com/perryb3693/twoge_k8s/assets/129805541/80dda0bd-a714-4aa4-a150-89f2cfc5f5ef">

Create the Dockerfile using database information:
```
vim Dockerfile
```
```
FROM python:alpine                                                                                                                               #using pyython alpine as the base image 

ENV SQLALCHEMY_DATABASE_URI=postgresql://postgres:password@twoge-database-1.ctfbo8wbotzl.ca-central-1.rds.amazonaws.com/postgres                 #environment variable contains sensitive information and will be removed after initial testing
RUN apk update && \                                                                                                                              #updates the indexes from all configured package repos
    apk add --no-cache build-base libffi-dev openssl-dev                                                                                         #adds the requsted packages to the image and installs/upgrades them 
COPY . /app                                                                                                                                      #copy the working directory on the local machine to the /app directory
WORKDIR /app
RUN pip install -r requirements.txt                                                                                                              #installs the dependencies listed within the requirements.txt file
EXPOSE 80                                                                                                                                        #specify the open port
CMD python app.py                                                                                                                                #command to run the python application
```
Build the image then push to DockerHub for later use for containers built within the Kubernetes Cluster. 
```
docker build -t perryb3693/twoge .
docker push perryb3693/twoge
```
***Step 2: Deploy Twoge on Minikube***

Next, write the Kubernetes Deployment and Service YAML configuration files using the created Docker Image to deploy the Twoge application and expose it to network traffic within the Minikube cluster. 
```
vim twoge_dep.yml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: twoge-dep                                           #name of the deployment. this name will become the basis for the replicasets and pods which are created later
spec:
  selector:                                                 #defines how the created replicaset finds which Pods to manage
    matchLabels:
      app: twoge-web                                        #specify what pods to apply deployment to
  replicas: 1                                               #the deployment creates a replicaset that creates the specified number of replicated pods in the background
  template: 
    metadata:
      labels:                                               #attach labels to replicaset pods for organization 
        app: twoge-web       
    spec:
      containers:                                           #indicates that the pods will run one container with the specified name and image from dockerhub 
        - name: twoge-webserver                             #name of the replicaset containers
          image:  perryb3693/twoge                          #image used by the replicaset contaienrs
          ports:
            - containerPort: 80                             #opens the specified port on the container to allow traffic
```
```
vim twoge_service.yml
```
```
#a serverice is a method of exposing a network application that is running as one or more Pods in your cluster. You use a Service to make a set of Pods available on the network so that clients can interact with it. 

apiVersion: v1
kind: Service
metadata:
  name: twoge-service
spec:
  selector:
    app: twoge-web
  type: NodePort   
  ports:
    - protocol: TCP
      port: 80                   #the internal port number on which the Service is accessible within the cluster             
      targetPort: 80             # the port number to which the network traffic is forwarded to the pods
      nodePort: 30333            #static port number on each cluster node, allows access to the Service through the port specified
```
```
minikube start
minikube kubectl -- apply -f .
minikube service twoge-service --url
```
Navigate to the service URL and post a Twoge blog

![ezgif com-video-to-gif (1)](https://github.com/perryb3693/twoge_k8s/assets/129805541/1d662e3a-2769-4a59-a524-5ac1125b5432)

***Step 3: Configure Database***

Next, redploy a database within the Kubernetes Cluster to use within the Twoge Application instead and use ConfigMap and Secrets YAML configurations to pass database configurations to the application. 

Update the docker image with removing the environment variable and push to DockerHub using a new tag (2.0.0).
```
FROM python:alpine

RUN apk update && \
    apk add --no-cache build-base libffi-dev openssl-dev
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 80
CMD python app.py
```
```
docker build -t perryb3693/twoge:2.0.0 .
docker push perryb3693/twoge:2.0.0
```

configure secrets yaml file
vim twoge_secrets.yml
```
apiVersion: v1
kind: Secret
metadata:
  name: twoge-secrets
type: Opaque
stringData:
  postgres-uri: postgresql://postgres:password@postgres-service/postgres
  postgres-username: postgres
  postgres-password: password

#URI Format - <database_engine>://<username>:<password>@<host>:<port>/<database_name>
```

configure configmap file 
vim postgres-configmap.yml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-configmap
data:
  postgres-database: postgres
```

configure postgres deployment yaml
vim postgres-dep.yml
```
#Postgres Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres                                  #Sets the deployment's name
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres                            
        image: postgres                         #uses the latest Postgres image
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: postgres-configmap
              key: postgres-database
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: twoge-secrets
              key: postgres-username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: twoge-secrets
              key: postgres-password
```

configure postgres service yaml
vim postgres-service.yml
```
# MySQL Service
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  labels:
    app: postgres
spec:
  selector:
    app: postgres
  type: ClusterIP
  ports:
    - port: 5432                #Sets port to run the postgres application
```

edit twoge application deployment yaml file to include database environment variable and use new image
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: twoge-dep       #name of the deployment. this name will become the basis for the replicasets and pods which are created later
spec:
  selector:               #defines how the created replicaset finds which Pods to manage
    matchLabels:
      app: twoge-web
  replicas: 3                   #the deployment creates a replicaset that creates the specified number of replicated pods in the background
  template: 
    metadata:
      labels:                #attach labels to replicaset pods for organization
        app: twoge-web       
    spec:
      containers:                           #indicates that the pods will run one container with the specified name and image from dockerhub 
        - name: twoge-webserver           #name of the replicaset containers
          image:  perryb3693/twoge:2.0.0 #image used by the replicaset contaienrs
          ports:
            - containerPort: 80
          livenessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
          env:
            - name: SQLALCHEMY_DATABASE_URI
              valueFrom:
                secretKeyRef:
                  name: twoge-secrets
                  key: postgres-uri
```

***Step 4: Namespace and Quotas***

Create a new namespace and define a resource quota to apply to the kubernetes cluster

vim twoge-namespace.yml
```
apiVersion: v1
kind: Namespace
metadata:
  name: twoge-development
  labels:
    name: twoge-development
```
Apply the namespace yaml file to create the new namespace then save the namespace configuration for all subsequent kubectl commands
minikube kubectl -- apply -f twoge-namespace.yml
minikube kubectl -- config set-context --current --namespace=twoge-development
minikube kubectl -- config view --minify | grep namespace:

Create a ResourceQuota and apply it to the new namespace

vim quota-pod.yml
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-quota
spec:
  hard:
    pods: "3"
```

minikube kubectl -- apply -f quota-pod.yml

Redeploy all pods under the new namespace and check current namespace configurations
```
minikube kubectl -- apply -f .
minikube kubectl -- describe namespaces twoge-development
```
```
Name:         twoge-development
Labels:       kubernetes.io/metadata.name=twoge-development
              name=twoge-development
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:     pod-quota
  Resource  Used  Hard
  --------  ---   ---
  pods      3     3

No LimitRange resource.
```



View detailed information about the ResourceQuota
```
minikube kubectl -- get resourcequota pod-quota --namespace=twoge-development --output=yaml
minikube kubectl -- get deployment twoge-dep --namespace=twoge-development --output=yaml
```
The output shows that even though the number of replicas specified within the Deployment is three, only two Pods were created due to the emplaced quota.
status:
```
  availableReplicas: 2
  conditions:
  - lastTransitionTime: "2023-07-01T20:34:13Z"
    lastUpdateTime: "2023-07-01T20:34:13Z"
    message: Deployment does not have minimum availability.
    reason: MinimumReplicasUnavailable
    status: "False"
    type: Available
  - lastTransitionTime: "2023-07-01T20:34:14Z"
    lastUpdateTime: "2023-07-01T20:34:14Z"
    message: 'pods "twoge-dep-9d4465966-5qttw" is forbidden: exceeded quota: pod-quota,
      requested: pods=1, used: pods=3, limited: pods=3'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  - lastTransitionTime: "2023-07-01T20:34:13Z"
    lastUpdateTime: "2023-07-01T20:34:32Z"
    message: ReplicaSet "twoge-dep-9d4465966" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 2
  replicas: 2
  unavailableReplicas: 1
  updatedReplicas: 2
```


***Step 5: Probes***

Implement a liveness probe within the YAML files of each of the deployments. The kublet will run the first liveness probe 15 seconds after the container starts. It will attempt to connect to the twoge-service container on port 80 and the postgres container on port 5432. If the liveness prove  fails, the container is restarted by the kublet.

```
livenessProbe:
    tcpSocket:
        port: 80
    initialDelaySeconds: 5
    periodSeconds: 5
```
kubectl describe -pod-

***Step 6: Deploy on AWS EKS***

Migrate the deployments to EKS. First, create an EKS cluster using the command line interface:
```
eksctl create cluster --region ca-central-1 --node-type t2.small --nodes 1 --nodes-min 1 --nodes-max 1 --name twoge-cluster
```
Navigate to the directory storing the previously configured YAML files and apply the configuration to build the deployments and services within the EKS cluster
```
kubectl apply -f .
```
Use `kubectl get all` to verify the successful deployment of all nodes 
Output:
```
NAME                            READY   STATUS    RESTARTS        AGE
pod/postgres-5996fb64f5-vw2hc   1/1     Running   0               4m36s
pod/twoge-dep-8f55967f-k57b6    1/1     Running   1 (4m14s ago)   4m35s
pod/twoge-dep-8f55967f-lhz8t    1/1     Running   2 (4m12s ago)   4m36s

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP                                                                 PORT(S)        AGE
service/kubernetes         ClusterIP      10.100.0.1      <none>                                                                      443/TCP        86m
service/postgres-service   ClusterIP      10.100.64.101   <none>                                                                      5432/TCP       4m36s
service/twoge-service      LoadBalancer   10.100.46.81    a6cd4f493af494e0c8b32fa0d576c163-897229701.ca-central-1.elb.amazonaws.com   80:30333/TCP   4m36s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/postgres    1/1     1            1           4m36s
deployment.apps/twoge-dep   2/3     2            2           4m36s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/postgres-5996fb64f5   1         1         1       4m36s
replicaset.apps/twoge-dep-8f55967f    3         2         2       4m36s
```
Navigate to the specified external-ip for the twoge-service and post a blog on Twoge!
<img width="910" alt="image" src="https://github.com/perryb3693/twoge_k8s/assets/129805541/fe23d2af-1eb9-41ed-9d9d-29fc8646b635">

