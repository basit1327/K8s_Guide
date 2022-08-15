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
