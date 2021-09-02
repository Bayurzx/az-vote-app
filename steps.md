## Run the application locally with Docker
- Navigate to the downloaded directory
``` bash
cd voting-app
```
- Use docker-compose.yaml file to create images, and run the application locally using Docker.

- The command below will create two images - one for the frontend and another for backend. 
- The frontend image is built based on the Dockerfile present in the "/azure-vote/" directory. 
- The backend image is built based on a standard Redis image fetched from the Dockerhub
- If you wish, YOU CAN CHANGE THE IMAGE NAME:TAG  in the docker-compose.yaml file
``` bash
docker-compose up -d
```
- View images locally 
- You will see two new images - "mcr.microsoft.com/azuredocs/azure-vote-front:v1" and "mcr.microsoft.com/oss/bitnami/redis:6.0.8"
``` bash
docker images
```
- You will see two running containers - "azure-vote-front" and "azure-vote-back" 
``` bash
docker ps
```
- Go to http://localhost:8080 see the app running
- Stop the application
``` bash
docker-compose down
```

## Create a Container Registry in Azure
- Create a resource group
  - Cloud Lab users can ignore this command and should use the existing Resource group, such as "cloud-demo-XXXXXX" 

```
az group create --name deletenow --location eastus
```


- ACR name should not have upper case letter

```
az acr create --resource-group deletenow --name myacr202106 --sku Basic
```


- Log in to the ACR

```
az acr login --name myacr202106
```


- Get the ACR login server name
  - To use the azure-vote-front container image with ACR, the image needs to be tagged with the login server address of your registry. 
  - Find the login server address of your registry

```
az acr show --name myacr202106 --query loginServer --output table
```


- Associate a tag to the local image

```
docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 myacr202106.azurecr.io/azure-vote-front:v1
```


- Now you will see myacr202106.azurecr.io/azure-vote-front:v1 if you run docker images
  - Push the local registry to remote

```
docker push myacr202106.azurecr.io/azure-vote-front:v1
```


- Verify if you image is up in the cloud.

```
az acr repository list --name myacr202106.azurecr.io --output table
```



## Create a Kubernetes cluster
### You can create the cluster in one go using the [create-cluster.sh](https://raw.githubusercontent.com/Bayurzx/vmss/master/voting-app/create-cluster.sh) shell script given at the bottom of this page. The content of the script is as follows:
- To ensure your cluster operates reliably, it is preferable to run at least 2 (two) nodes.
- For this exercise, let's start with just one node.
- NOTE: Cloud Lab users cannot use the "--enable-addons monitoring" option in the "az aks create" because they are not allowed to create the Log Analytics workspace.
- Instead, Cloud Lab users should enable the monitoring in a separate command "az aks enable-addons -a monitoring", as shown in the "create-cluster.sh" above, to use the existing Log Analytics workspace's Resource ID.

```
az aks create --name myAKSCluster --resource-group deletenow --node-count 1 --enable-addons monitoring --generate-ssh-keys --attach-acr myacr202106
```


- To connect to the Kubernetes cluster from your local computer, you use kubectl, the Kubernetes command-line client.
- Preferable run as super user sudo

```
az aks install-cli
```


- Configure kubectl to connect to your Kubernetes cluster

```
az aks get-credentials --resource-group deletenow --name myAKSCluster
```


- Verify the connection to your cluster

```
kubectl get nodes
```



## Deploy the images to the AKS cluster
# Get the ACR login server name
```
az acr show --name myacr202106 --query loginServer --output table
```

<strong> <span style="color:red">Some <em>Important Notice: </em> </span> </strong>
- Edit the manifest file, *azure-vote-all-in-one-redis.yaml*, to replace `mcr.microsoft.com/azuredocs/azure-vote-front:v1` with `myacr202106.azurecr.io/azure-vote-front:v1`.  
- If you do not change the manifest file, then it will use Microsoft's image for this application available at Dockerhub - https://hub.docker.com/r/microsoft/azure-vote-front
- The reason for editing is that in order to push an image to ACR, the image needs to be tagged with the login server address of your registry. 


## Continued
- Deploy the application. Run the command below from the parent directory where the *azure-vote-all-in-one-redis.yaml* file is present. 

```
kubectl apply -f azure-vote-all-in-one-redis.yaml
```

- Test the application

```
kubectl get service azure-vote-front --watch
```

- You can also verify that the service is running like this

```
kubectl get service
```

<!-- # Troubleshoot
- Check the status of each node
```
kubectl get pods
```
- It may require you to associate the AKS with the ACR
```
az aks update -n bayurzx-cluster -g deletenow --attach-acr myacr202106
```
- Redeploy
```
kubectl set image deployment azure-vote-front azure-vote-front=myacr202106.azurecr.io/azure-vote-front:v1
``` -->

## Generate synthetic load, and autoscale the Pods
- Let's first autoscale the number of pods in the azure-vote-front deployment. The following command will set the policy such that if average CPU utilization across all pods exceeds 50% of their requested usage, the autoscaler will increase the pods from a minimum of 3 instances up to a maximum of 10 instances
```
kubectl autoscale deployment azure-vote-front --cpu-percent=50 --min=3 --max=10
```
   - Also, note that the deployment file azure-vote-all-in-one-redis.yaml used in the kubectl apply command already has the CPU requests and limits defined for all containers in your pods. The deployment file defines the azure-vote-front container to have 25% CPU requests, with a limit of 50% CPU.

```
     resources:
       requests:
         cpu: 250m
       limits:
         cpu: 500m
```
- Now, to generate the synthetic load on the AKS cluster, you can run:
    - Generate load in the terminal by creating a container with "busybox" image
    - Open the bash into the container
```
kubectl run -it --rm load-generator --image=busybox /bin/sh
# You will see a new command prompt. Enter the following in the new command prompt

while true; do wget -q -O- [Public-IP]; done # add the ip 20.81.11.133

```

- You can check the increase in the number of pods by using the command in a new terminal window
```
kubectl get hpa --watch
```

- Let the loop run for some time and then cancel
    - If you let it run for 5mins you get something like this
``` powershell
PS C:\Users\USER\Desktop\Azure\vmss> kubectl get service azure-vote-front --watch
NAME               TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)        AGE
azure-vote-front   LoadBalancer   10.0.218.16   20.81.11.133   80:31137/TCP   65m
PS C:\Users\USER\Desktop\Azure\vmss> kubectl get hpa
NAME               REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
azure-vote-front   Deployment/azure-vote-front   0%/50%    3         10        3          5m49s
PS C:\Users\USER\Desktop\Azure\vmss> kubectl get hpa --watch
NAME               REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
azure-vote-front   Deployment/azure-vote-front   69%/50%   3         10        3          6m10s
azure-vote-front   Deployment/azure-vote-front   69%/50%   3         10        5          6m23s
azure-vote-front   Deployment/azure-vote-front   26%/50%   3         10        5          6m54s
azure-vote-front   Deployment/azure-vote-front   20%/50%   3         10        5          7m55s
azure-vote-front   Deployment/azure-vote-front   6%/50%    3         10        5          8m55s
azure-vote-front   Deployment/azure-vote-front   0%/50%    3         10        5          9m55s
azure-vote-front   Deployment/azure-vote-front   0%/50%    3         10        5          11m
azure-vote-front   Deployment/azure-vote-front   0%/50%    3         10        3          11m
```

- Delete the horizontalpodautoscaler.autoscaling
```
kubectl delete hpa azure-vote-front
``` 
