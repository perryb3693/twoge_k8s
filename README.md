# twoge_k8s

The purpose of this repository is to gain experience using Kubernetes by deploying the Twoge application through Kubernetes. The Twoge application will first be deployed on a local Kubernetes cluster through Minikube, then deployed utilizing AWS Elastic Kubernetes Service (EKS).
Source files for the Twoge application can be found at https://github.com/chandradeoarya/twoge/tree/k8s


***Step 1: Create a Docker Image of the Twoge Application***
The first step in deploying the Twoge application through Kubernetes is to first dockerize the application by defining the application's environment and its dependencies within a Dockerfile to build a docker image later.

Clone the application's repository onto the local system then change over to the new directory and create the Dockerfile. 

```
git clone https://github.com/chandradeoarya/twoge.git
cd twoge
```
Build a Postgres database using Amazon RDS for initial testing of the Twoge Application. Configure attached security group to allow inbound traffic over port 5432 and 22. Take note of the database server's endpoint URL and the specified user/password/database information for use later within the environment variable. 

<img width="679" alt="image" src="https://github.com/perryb3693/twoge_k8s/assets/129805541/80dda0bd-a714-4aa4-a150-89f2cfc5f5ef">

Create the Dockerfile using database information:
```
vim Dockerfile
```
```
FROM python:alpine

ENV SQLALCHEMY_DATABASE_URI=postgresql://postgres:password@twoge-database-1.ctfbo8wbotzl.ca-central-1.rds.amazonaws.com/postgres
RUN apk update && \
    apk add --no-cache build-base libffi-dev openssl-dev
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 80
CMD python app.py
```
Build the image then push to DockerHub for later use for containers built within the Kubernetes Cluster. 
```
docker build -t perryb3693/twoge .
docker push perryb3693/twoge
```
***Deploy Twoge on Minikube***
Next, write the Kubernetes Deployment and Service YAML configuration files using the created Docker Image to deploy the Twoge application and expose it within the Minikube cluster. 
```
vim twoge_dep.yml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: twoge-dep       #name of the deployment. this name will become the basis for the replicasets and pods which are created later
spec:
  selector:               #defines how the created replicaset finds which Pods to manage
    matchLabels:
      app: twoge-web
  replicas: 1                   #the deployment creates a replicaset that creates the specified number of replicated pods in the background
  template: 
    metadata:
      labels:                #attach labels to replicaset pods for organization
        app: twoge-web       
    spec:
      containers:                           #indicates that the pods will run one container with the specified name and image from dockerhub 
        - name: twoge-webserver           #name of the replicaset containers
          image:  perryb3693/twoge #image used by the replicaset contaienrs
          ports:
            - containerPort: 80
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

Remove the environment variable from the Dockerfile and build/push to DockerHub using a new tag (2.0.0).
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



