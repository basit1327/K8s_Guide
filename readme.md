# Deploying Application to Kubernetes Cluster

- Create Azure Container Registory (ACR)
- Create Azure Resource Group
- Create K8 Cluster using AKS

#### Login to AZ 
```$ az login``` This will open azure portal for authenticating user
#### Creating Resource Group
```az group create --name myResourceGroup --location eastus```

#### Creating Cluster
You can use AKS dashboard or CLI to create Cluster
```az aks create -g myResourceGroup -n myAKSCluster --enable-managed-identity --node-count 1 --enable-addons monitoring --enable-msi-auth-for-monitoring  --generate-ssh-keys```
#### Connect to the cluster
```az aks install-cli```

#### Downloads credentials and configures the Kubernetes CLI to use them
```az aks get-credentials --resource-group myResourceGroup --name myAKSCluster```

#### Verify the connection to your cluster
```kubectl get```

#### Getting Cluster Nodes
````kubectl get nodes````
## Deploy the Application (Angular Frontend)

https://docs.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli

Docker Image will be build/push by Azure Pipeline Code Build
````
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loader-design
spec:
  replicas: 2
  selector:
    matchLabels:
      app: loader-design
  template:
    metadata:
      labels:
        app: loader-design
    spec:
      containers:
      - name: loader-design
        image: basitraza88.azurecr.io/loaderdesign:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 250m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 256Mi
      # If image is not public and require authentication for pulling image    
      # Authentication is discussed below  
      imagePullSecrets:
      - name: <SECRET_NAME_FROM_K8>      
---
apiVersion: v1
kind: Service
metadata:
  name: loader-design
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: loader-design
````
## Authenticate with an Azure container registry
There are several ways to authenticate with an Azure container registry, each of which is applicable to one or more registry usage scenarios.

https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication?tabs=azure-cli

### Individual AD identity 
```az acr login --name <acrName> --expose-token```

Output displays the access token, abbreviated here:
```
{
  "accessToken": "eyJhbGciOiJSUzI1NiIs[...]24V7wA",
  "loginServer": "myregistry.azurecr.io"
}
```

Store the token credential in a safe location and follow recommended practices to manage docker loginServer
```
TOKEN=$(az acr login --name <acrName> --expose-token --output tsv --query accessToken)
```
Then, run docker login, passing 00000000-0000-0000-0000-000000000000 as the username and using the access token as password
```
docker login myregistry.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password $TOKEN
```


## How to authenticate in K8 to pull private images
#### Pull an Image from a Private Registry
https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/

This section shows how to create a Pod that uses a Secret to pull an image from a private container image registry or repository

When login to docker by above method ```docker login .... ```.

The login process creates or updates a config.json file that holds an authorization token.
View the config.json file: ```cat ~/.docker/config.json```

The output contains a section similar to this:
```
{
    "auths": {
        "https://index.docker.io/v1/": {
            "auth": "c3R...zE2"
        }
    }
}
```

### Create a Secret based on existing credentials
A Kubernetes cluster uses the Secret of kubernetes.io/dockerconfigjson type to authenticate with a container registry to pull a private image.

If you already ran docker login, you can copy that credential into Kubernetes:
```
kubectl create secret generic <SECRET_NAME_FROM_K8> \
    --from-file=.dockerconfigjson=<path/to/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson
```


### (I use this) Create a Secret by providing credentials on the command line  
Create this Secret, naming it <SECRET_NAME_FROM_K8> :

``` ```
```
kubectl create secret docker-registry <SECRET_NAME_FROM_K8>  
--docker-server=<your-registry-server> 
--docker-username=<your-name> 
--docker-password=<your-pword> 
--docker-email=<your-email> 
```


Combining it with docker login --expose-token
```
TOKEN=$(az acr login --name <acrName> --expose-token --output tsv --query accessToken)

kubectl create secret docker-registry <SECRET_NAME_FROM_K8>  
--docker-server=<your-registry-server> 
--docker-username=00000000-0000-0000-0000-000000000000
--docker-password=$TOKEN 
--docker-email=<your-email> 
```
## Updating Secrets in Kubectl
https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/

We create secret from above step. If secrets are expire by timeout then you will get Image pull error
So we have to update secret in kubectl. 

### Delete the Secret you created:
```kubectl delete secret<SECRET_NAME_FROM_K8>```

Then Create again, Or other option is to edit the Secrets

### Check that the Secret was created:
```kubectl get secrets```



### Rolling Update 
For updating image version we can use below command
```kubectl set image deployment/<DEPLOYMENT_NAME> <DEPLOYMENT_NAME>=<REGISTERY_URL>/<IMAGE_NAME>:<VERSION>```
