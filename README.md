# twoge_k8s

The purpose of this repository is to gain experience using Kubernetes by deploying the Twoge application through Kubernetes. The Twoge application will first be deployed on a local Kubernetes cluster through Minikube, then deployed utilizing AWS Elastic Kubernetes Service (EKS).
Source files for the Twoge application can be found at https://github.com/chandradeoarya/twoge/tree/k8s


***Step 1: Create a Docker Image of the Twoge Application***
The first step in deploying the Twoge application through Kubernetes is to first dockerize the application by compiling the application and its dependencies into a Dockerfile to build a docker image for later use. Clone the application's repository onto the local system then change over to the new directory and create the Dockerfile. 

```
git clone https://github.com/chandradeoarya/twoge.git
cd twoge
code Dockerfile
```
```



```
docker build -t perryb3693/twoge .
docker push perryb3693/twoge
```
Create Postgres Database using Amazon RDS
Configure security group

```
minikube start
minikube kubectl -- apply -f .
minikube kubectl -- port-forward deployment/twoge-dep :80
