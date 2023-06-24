# Quickstart: Develop on Azure Kubernetes Service (AKS) with Helm

Helm is an open-source packaging tool that helps you install and manage the lifecycle of Kubernetes applications. Similar to Linux package managers like APT and Yum, Helm manages Kubernetes charts, which are packages of pre-configured Kubernetes resources.

In this quickstart, you'll use Helm to package and run an application on AKS.

## Prerequisites
An Azure subscription. If you don't have an Azure subscription, you can create a free account.

Azure CLI.

Helm v3 installed.

### Create an Azure Container Registry

You'll need to store your container images in an Azure Container Registry (ACR) to run your application in your AKS cluster using Helm. 

>az group create --name myResourceGroup --location brazil

Pick unique name for ACR
>az acr create --resource-group MyResourceGroup --name myhelmacrmr --sku Basic

### Create an AKS cluster. 

If quota erro shows up, one way to address it is to pass the ```--node-vm-size``` with a VM of appropriate type. i.e Standard_DS2_v2, that uses 2 cores and you can limit node count to 1 to ensure only one node will be created

Use the az aks create command to create an AKS cluster called myAKSCluster and the ```--attach-acr``` parameter to grant the cluster access to the myhelmacr ACR.

>az aks create --resource-group myResourceGroup --name myAKSCluster --location eastus --attach-acr myhelmacr --generate-ssh-keys

### Connect to your AKS cluster

Install kubectl locally
>az aks install-cli

Configure kubectl to connect to your Kubernetes cluster using the az aks get-credentials command. The following command gets credentials for the AKS cluster named myAKSCluster in myResourceGroup:
>az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

### Download the sample application

>git clone https://github.com/Azure-Samples/azure-voting-app-redis.git
>cd azure-voting-app-redis/azure-vote/

### Build and push the sample application to the ACR

The . at the end of the command provides the location of the source code directory path (in this case, the current directory). The --file parameter takes in the path of the Dockerfile relative to this source code directory path.
>az acr build --image azure-vote-front:v1 --registry myhelmacr --file Dockerfile .

### Create your Helm chart

Generate your Helm chart
>helm create azure-vote-front

Update azure-vote-front/Chart.yaml to add a dependency for the redis chart from the https://charts.bitnami.com/bitnami chart repository and update appVersion to v1. For example:
```
apiVersion: v2
name: azure-vote-front
description: A Helm chart for Kubernetes

dependencies:
  - name: redis
    version: 17.3.17
    repository: https://charts.bitnami.com/bitnami

...
# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application.
appVersion: v1
```

Update your helm chart dependencies using the helm dependency update command.
>helm dependency update azure-vote-front


Update azure-vote-front/values.yaml with the following changes:

>Add a redis section to set the image details, container port, and deployment name.
>Add a backendName for connecting the frontend portion to the redis deployment.
>Change image.repository to <loginServer>/azure-vote-front.
>Change image.tag to v1.
>Change service.type to LoadBalancer.
```
# Default values for azure-vote-front.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1
backendName: azure-vote-backend-master
redis:
  image:
    registry: mcr.microsoft.com
    repository: oss/bitnami/redis
    tag: 6.0.8
  fullnameOverride: azure-vote-backend
  auth:
    enabled: false

image:
  repository: myhelmacr.azurecr.io/azure-vote-front
  pullPolicy: IfNotPresent
  tag: "v1"
...
service:
  type: LoadBalancer
  port: 80
...
```

Add an env section to azure-vote-front/templates/deployment.yaml for passing the name of the redis deployment.
```
...
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
          - name: REDIS
            value: {{ .Values.backendName }}
...
```

### Run your Helm chart

Install your application using your Helm chart using the helm install command.
>helm install azure-vote-front azure-vote-front/

Monitor progress using the kubectl get service command with the --watch argument. Navigate to your application's load balancer in a browser using the <EXTERNAL-IP> to see the sample application.

>kubectl get service azure-vote-front --watch

#### Source
@Azure Docs