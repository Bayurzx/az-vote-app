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

---

# Exercise: Collecting Telemetry Data for an application deployed on AKS cluster

*Prerequisite*
- You should have an Application Insights instance already created in your account
- You should have an application already deployed on the AKS cluster

## Step 1. Copy the Instrumentation Key of your Application Insights instance
- If you aren't already, log in to Azure Portal.
- Navigate to your Application Insights instance.
- Copy the instrumentation key.

## Step 2 - Edit the Dockerfile [Only for deployment to AKS]
- Add the following dependencies to the azure-voting-app-redis/azure-vote/Dockerfile included in the starter code used in the last exercise.
``` dockerfile
# The first line below would already be there
RUN pip install redis
RUN pip install opencensus
RUN pip install opencensus-ext-azure
RUN pip install opencensus-ext-flask
RUN pip install opencensus-ext-logging
RUN pip install flask
```

## Step 3 - Edit the main.py
Note that the working main.py file is attached at the bottom of this page as well. But, we encourage you to go through each code block below and add it to your main.py file.

- Open the azure-voting-app-redis/azure-vote/azure-vote/main.py file, and import the following AppInsight libraries:
``` py
import logging
from datetime import datetime
# App Insights
# TODO: Import required libraries for App Insights
from opencensus.ext.azure.log_exporter import AzureLogHandler
from opencensus.ext.azure.log_exporter import AzureEventHandler
from opencensus.ext.azure import metrics_exporter
from opencensus.stats import aggregation as aggregation_module
from opencensus.stats import measure as measure_module
from opencensus.stats import stats as stats_module
from opencensus.stats import view as view_module
from opencensus.tags import tag_map as tag_map_module
from opencensus.trace import config_integration
from opencensus.ext.azure.trace_exporter import AzureExporter
from opencensus.trace.samplers import ProbabilitySampler
from opencensus.trace.tracer import Tracer
from opencensus.ext.flask.flask_middleware import FlaskMiddleware
# For metrics
stats = stats_module.stats
view_manager = stats.view_manager
```
- Add Logger for custom Events:

``` py
config_integration.trace_integrations(['logging'])
config_integration.trace_integrations(['requests'])
# Standard Logging
logger = logging.getLogger(__name__)
handler = AzureLogHandler(connection_string='InstrumentationKey=[your-guid]')
handler.setFormatter(logging.Formatter('%(traceId)s %(spanId)s %(message)s'))
logger.addHandler(handler)
# Logging custom Events 
logger.addHandler(AzureEventHandler(connection_string='InstrumentationKey=[your-guid]'))
# Set the logging level
logger.setLevel(logging.INFO)
```
- Add Metrics
``` py
# Metrics
exporter = metrics_exporter.new_metrics_exporter(
enable_standard_metrics=True,
connection_string='InstrumentationKey=[your-guid]')
view_manager.register_exporter(exporter)
```

- Add Tracer
``` py
# Tracing
tracer = Tracer(
 exporter=AzureExporter(
     connection_string='InstrumentationKey=[your-guid]'),
 sampler=ProbabilitySampler(1.0),
)
app = Flask(__name__)
```

- Add Requests
``` py
# Requests
middleware = FlaskMiddleware(
 app,
 exporter=AzureExporter(connection_string="InstrumentationKey=[your-guid]"),
 sampler=ProbabilitySampler(rate=1.0)
)
```

*In the code snippet above, replace [your-guid] with the Instrumentation Key copied in the step above. For example, after replacement, the connection string would look like:*

``` bash
connection_string='InstrumentationKey=cc4cd863-cad1-469e-ba7e-87ec2c4ed5d0'),
```

*Also, ensure the Redis connection according to the deployment. For localhost / single-host deployment (VMSS), the configuration will be:*

``` py
# Redis Connection to a local server running on the same machine where the current Flask app is running. 
r = redis.Redis()
```

*Whereas, for multi-container (AKS) deployment, the configuration will be:*

``` py
redis_server = os.environ['REDIS']
# Redis Connection to another container
try:
 if "REDIS_PWD" in os.environ:
     r = redis.StrictRedis(host=redis_server,
                     port=6379,
                     password=os.environ['REDIS_PWD'])
 else:
     r = redis.Redis(redis_server)
 r.ping()
except redis.ConnectionError:
 exit('Failed to connect to Redis, terminating.')
```

- *Use tracer.span() to capture an event. Also, use logger.info() to send custom events to the Application Insights instance. For this purpose, you need to add custom properties to your log messages in the extra keyword argument (see below). For example:*

``` py
@app.route('/', methods=['GET', 'POST'])
def index():

 if request.method == 'GET':

     # Get current values
     vote1 = r.get(button1).decode('utf-8')
     # TODO: use tracer object to trace cat vote
     with tracer.span(name="Cats Vote") as span:
         print("Cats Vote")

     vote2 = r.get(button2).decode('utf-8')
     # TODO: use tracer object to trace dog vote
     with tracer.span(name="Dogs Vote") as span:
         print("Dogs Vote")

     # Return index with values
     return render_template("index.html", value1=int(vote1), value2=int(vote2), button1=button1, button2=button2, title=title)

 elif request.method == 'POST':

     if request.form['vote'] == 'reset':

         # Empty table and return results
         r.set(button1,0)
         r.set(button2,0)
         vote1 = r.get(button1).decode('utf-8')
         properties = {'custom_dimensions': {'Cats Vote': vote1}}
         # TODO: use logger object to log cat vote
         logger.info('Cats Vote', extra=properties)

         vote2 = r.get(button2).decode('utf-8')
         properties = {'custom_dimensions': {'Dogs Vote': vote2}}
         # TODO: use logger object to log dog vote
         logger.info('Dogs Vote', extra=properties)

         return render_template("index.html", value1=int(vote1), value2=int(vote2), button1=button1, button2=button2, title=title)

     else:

         # Insert vote result into DB
         vote = request.form['vote']
         r.incr(vote,1)

         # Get current values
         vote1 = r.get(button1).decode('utf-8')
         properties = {'custom_dimensions': {'Cats Vote': vote1}}
         # TODO: use logger object to log cat vote
         logger.info('Cats Vote', extra=properties)

         vote2 = r.get(button2).decode('utf-8')
         properties = {'custom_dimensions': {'Dogs Vote': vote2}}
         # TODO: use logger object to log dog vote
         logger.info('Dogs Vote', extra=properties)            

         # Return results
         return render_template("index.html", value1=int(vote1), value2=int(vote2), button1=button1, button2=button2, title=title)
```

  Note the following regarding using tracer:

  - The supported versions of Python are v2.7 and v3.4-v3.7.
  - Ensure the sampler=ProbabilitySampler(1.0) so it collects 100% of the data. The range of acceptible values are between (0.0-1.0).
  - With tracer.span(name='myTrace'), a telemetry item will be sent for the span "myTrace"

*These events will be displayed in the Application Insights instance in Azure. Also, you can change the values in the azure-vote/azure-vote/config_file.cfg file to see if the changes persist after re-deployment.*

Reference:

- [Set up Azure Monitor for your Python application](https://docs.microsoft.com/en-us/azure/azure-monitor/app/opencensus-python)
- [Tracking Flask applications](https://docs.microsoft.com/en-us/azure/azure-monitor/app/opencensus-python-request#tracking-flask-applications)
- [Using opencensus-ext-azure package](https://pypi.org/project/opencensus-ext-azure/) - Scroll down to see example snippets. Although, some of them are already the same as the links above.

## Step 4 - Redeploy the app to the VMSS
- If your VMSS is still there, you can try deploying the application manually to at least one of the VMSS instances.
- Remember that the Redis configuration is different in the case of VMSS versus AKS deployment. For VMSS, you will use r = redis.Redis(). Also, note that the driver function will vary slightly:
``` py
if __name__ == "__main__":
  # For running the application locally 
  # app.run() # local
  # For deployment to VMSS
  app.run(host='0.0.0.0', threaded=True, debug=True) # remote
```

## Step 5 - Check the Application Insights instance
Go to the Azure Portal, and open your Application Insights instance. Click 'Events'. You will see your event. Make sure to use the right set of filters.


## Step 6 - Redeploy the app to the AKS cluster
Run the following commands in your terminal:

``` bash
# Stop and remove the local containers
docker-compose down
# Recreate containers because the frontend application has changed in the steps above 
docker-compose up -d --build --force-recreate
# Check the application running at http://localhost:8080/
# Tag the newly generated local image with the new tag, say "v2"
docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 myacr202106.azurecr.io/azure-vote-front:v2
# Login to the the ACR 
az acr login --name myacr202106
# Push the local image to the existing ACR 
docker push myacr202106.azurecr.io/azure-vote-front:v2
# Update the deployment image 
kubectl set image deployment azure-vote-front azure-vote-front=myacr202106.azurecr.io/azure-vote-front:v2
# Test the new deployment - use the external IP in your browser. 
kubectl get service azure-vote-front --watch
```

Run the flask app using the external IP in your browser and make the event trigger.

Reference: [Tutorial: Update an application in Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-app-update?tabs=azure-cli)

<span style="color:red"> Caution:</span> Collecting telemetry data (logs, traces, metrics, and requests) for all events is an expensive task. Azure Log Analytics workspace (created as a part of the Application Insights instance) will be overwhelmed with the telemetry data (in GiB) and thus incur a high cost.
On the other hand, the next lesson, Application Analytics, will need the same application deployed to VMSS and Application Insights enabled as a prerequisite. Therefore, either you can reduce the telemetry data collection or delete your resources accordingly.

