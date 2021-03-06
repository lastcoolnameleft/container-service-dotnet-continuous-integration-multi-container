***Sample app for demonstrating continuous integration and deployment of a multi-container Docker app to Azure Container Service***
---
This repository contains a sample Azure multi-container Docker application.

* service-a: Angular.js sample application with Node.js backend 
* service-b: ASP .NET Core sample service

## **Run application locally**
First, compile the ASP .NET Core application code. This uses a container to isolate build dependencies that is also used by VSTS for continuous integration:

```
docker-compose -f docker-compose.ci.build.yml run ci-build
```

(On Windows, you currently need to pass the -d flag to docker-compose run and poll the container to determine when it has completed).

```
docker-compose -f docker-compose.ci.build.yml run -d ci-build
```

Now build Docker images and run the services:

```
docker-compose up --build
```

The frontend service (service-a) will be available at http://localhost:8080.

## **Kubernetes (k8) deployment**

- [Create k8 cluster](#create-k8-cluster)
- [Install the kubectl command line](#install-the-kubectl-command-line)
- [Get the k8 cluster configuration](#get-the-k8-cluster-configuration)
- [Verify cluster operation and connectivity](#deploying-a-pod-and-service-from-a-public-repository)
- [Create Azure Container Registry (ACR)](#create-azure-container-registry)
- [Push demo app images to ACR](#push-demo-app-images-to-acr)
- [Enable access to ACR from k8](#create-a-k8-docker-registry-secret-to-enable-read-only-access-to-acr)
- [Deploy the sample app to k8](#deploy-the-application-to-the-k8-cluster)
- [Enable OMS monitoring of containers](#enable-oms-monitoring-of-containers)
- [Create and deploy into namspaces](#create-and-deploy-into-namspaces)

---
**Important Notes:** 
 - This lab assumes use of the [Azure CLI v2](https://github.com/Azure/azure-cli)
 - A number of resources created in this demo have names that must be globally unique (e.g. ACR endpoints).  In those cases the commands will include a placeholder value noted within angle brackets `<>` to signal that a value specific to your environment needs to be provided.

---

### **Create k8 cluster**
You can create a cluster through CLI v2 or the Azure Portal.  If you use the 
portal you will need to have created and provide a SSH key and Service Principal. 
If you use the command line you can have the SSH key and Service Principal 
created for you.  
For production deployments you would want to create and managed keys and Service Principals 
for specific purposes.  To quickly create a cluster for demo/development purposes
you can use the command line which will auto create:
- SSH keys - in your home/.ssh directory
- Service Principal - in your home/.azure directory

_Note: If you already have ssh keys in your home directory, then you should use those keys on the command line rather then allowing the CLI to create
new keys which will overrite any existing keys in your home/.ssh directory._

```
az login
az group create --name my-k8-clusters --location westus
az acs create --name my-k8-cluster --resource-group my-k8-clusters --orchestrator-type Kubernetes --dns-prefix <my-k8> --generate-ssh-keys 
```
### **Install the kubectl command line**
If not already installed, you can use the cli to install the k8 command line utility (kubectl).
Note:
- On Windows you need to have opened the command windows with Administrator rights as the installation tries write the program to C:\Program Files\kubectl.exe
- You may also have to add C:\Program Files to your PATH    
```
az acs kubernetes install-cli
```
_If you have difficulty using the CLI to install the application, you can download it from the [Kubernetes web site](https://kubernetes.io/docs/user-guide/prereqs/)_


### **Get the k8 cluster configuration**
The kubectl application requires configuration data which includes the cluster endpoint and credentails. The default location/name for the file is:`~/.kube/config`

The credentails and configuration data are created on the cluster admin server during installation and can be downloaded to
your machine using the `get-credential` subcommand.  

```
az acs kubernetes get-credentials --resource-group=my-k8-clusters --name=my-k8-cluster
```
_If you have difficulty using the CLI to get the configuration,  you can use ssh/scp to download the file from the `~/.kube/config` on the cluster master._


After downloading the cluster configuration you should be able to connect to the cluster using kubectl.  For example the cluster-info command will show the master and cluster services for your cluster.
```
kubectl cluster-info
```
### **Deploying a Pod and Service from a public repository**
The following steps can be used to quickly deploy an image from DockerHub (a public repository) that is made available via Azure external load balancer.  
The `kubectl get services` command will show the EXTERNAL-IP as 'Pending' until a public IP is provisioned for the service on the load balancer.  Once the EXTERNAL-IP
is assigned you can use that IP to render the nginx landing page.

```
kubectl run nginx --image nginx
kubectl get pods
kubectl expose deployments nginx --port=80 --type=LoadBalancer
kubectl get services
```
The default ACS deployment enables management access only through ssh.  You can enable admin access to the k8 web UI console by creating
a ssh tunnel or by using the `kubectl proxy` command to create the tunnel for you.

```
kubectl proxy
```
The output will note the port that the proxy binds to.  The console will then be available at that port on localhost.  e.g. <http://localhost:8001/ui>

### **Create Azure Container Registry**
In the previous step the image for ngnix was pulled from a public repository.  For  many customers they want to only deploy images from internal (controlled) private
registries. 

Note: ACR names are globally scoped so you can check the name of a regsitry before trying to create it
```
az acr check-name --name <myk8acr>
```
The minimal parameters to create a ACR are a name, resource group and location.  With these 
paramters a storage account will be created and administrator access will not be created.

Note: the command will return the resource id for the registry.  That id will need to be used in subsequent steps if you want to create service principals that are scoped 
to this registry instance.

```
az acr create --name <myk8acr> --resource-group my-k8-clusters --location westus
```

Create a two service principals, one with read only and one with read/write access.

Note: 
- Ensure that the password length is 8 or more characters
- The command will return an application id for each service principal.  You'll need that id in subsequent steps.
- You should consider using the --scope property to qualify the use of the service principal a resource group or registry

```  
az ad sp create-for-rbac --name my-acr-reader --role Reader --password my-acr-password
az ad sp create-for-rbac --name my-acr-contributor  --role Contributor --password my-acr-password
```

### **Push demo app images to ACR**
List the local docker images.  You should see the images built in the initial steps when deploying the application locally.
```
docker images
```

Tag the images for service-a and service-b to associate them with you private ACR instance.  

Note that you must provide your ACR registry endpoint

```
docker tag service-a:latest <myk8acr-microsoft.azurecr.io>/service-a:latest
docker tag service-b:latest <myk8acr-microsoft.azurecr.io>/service-b:latest
```

Using the *Contributor* Service Principal, log into the ACR.  The login command for a remote registry has the form:

docker login -u user -p password server

```
docker login -u <ContributorAppId>  -p <my-acr-password> <myk8acr-microsoft.azurecr.io> 
```

Push the images
```
docker push <myk8acr-microsoft.azurecr.io>/service-a
docker push <myk8acr-microsoft.azurecr.io>/service-b
```
You can list the repositories in a registry and the tags in a repository with the following command.

_Note that credentials are not needed if you enabled the Administrator login._

```
az acr repository list -n <myRegistry> [-u myUsername -p myPassword]
az acr repository show-tags -n <myRegistry> --repository <myRepository> [-u myUsername -p myPassword]
```

At this point the images are in ACR, but the k8 cluster will need credentails to be able to pull and deploy the images

### **Create a k8 docker-registry secret to enable read-only access to ACR**

```
kubectl create secret docker-registry acr-reader --docker-server=<myk8acr-microsoft.azurecr.io> --docker-username=<ContributorAppId> --docker-password=my-acr-password --docker-email=a@b.com
```

Note: If you use a different name for the secret you will have to update the imagePullSecrets name property in the k8-demo-app.yml file 

```
    imagePullSecrets:
        - name: acr-reader
```

### **Deploy the application to the k8 cluster**
Review the contents of the k8-demo-app.yml file.  

- It contains the objects to be created on the k8 cluster.  
  - Note that multple objects can be included within the same file
- Note the environment variables that are used to configure endpoints.  Since these containers are being deployed to a Pod:
  - The containers will communicate via localhost
  - Containers cannot listen on the same port
- Update the `image` references in the k8-demo-app.yml file to reference your ACR endpoint

```
    spec:
      containers:
        - name: web
          image: <myk8acr-microsoft.azurecr.io>/service-a
          ...
        - name: api
          image: <myk8acr-microsoft.azurecr.io>/service-b
          ...
        - name: mycache  
```

Deploy the application using the kubectl create command

```
kubectl create -f k8-demo-app.yml
```
 
### **Enable OMS monitoring of containers**
This expands on the steps described here <https://docs.microsoft.com/en-us/azure/container-service/container-service-kubernetes-oms>
by using a k8 secret to store the workspace id and key.

- Go to the OMS portal and get the workspace id and a key (see the page referenced in the prior link).
- Create a k8 secret to hold the workspace id and key.  The following command creates a generic secret named `oms-agent-secret` that holds the properties
`WSID` and `KEY`.  Substitute your WorkspaceID and WorkspaceKey for the placholder values.

```
kubectl create secret generic oms-agent-secret --from-literal=WSID=<WorkspaceID> --from-literal=KEY=<WorkspaceKey>
```

Note two points in the file k8-demo-enable-oms.yml
- The deployment type is `DaemonSet` which means one instance will be deployed to each agent Node
- The reference to the secret created in the prior step to hold the OMS WorkspaceID and WorkspaceKey 

Deploy the agent with the following command
```
kubectl create -f k8-demo-enable-oms.yml
```

Within a few minutes you should see metrics and logs for containers deployed in the k8 cluster.

### **Create and deploy into namspaces**
Kubernetes provides namespaces as a way to create isolated environments within a cluster (e.g. dev,test,prod)

Create a `test` and `prod` namespace using the kubectl create command
```
kubectl create -f k8-create-namespaces.yml
```

Use the UI or command line to verify the available namespaces now include `test` and `prod`
```
kubectl get namespaces
```
Deploy the demo application into the `test` namespace
- Get a list of the current contexts
- Create a context named `test` and bind it to the test namespace
- Set the current context to `test`
- Show the current context again and note that `test` is tagged as CURRENT
- Create a registry secret in the `test` namespace
- Deploy the application into the `test` namespace

```
kubectl config get-contexts
kubectl config set-context test --cluster=<my-k8-cluster> --user=<my-k8-cluster-admin> --namespace=test 
kubectl config use-context test
kubectl config get-contexts
kubectl create secret docker-registry acr-reader --docker-server=<myk8acr-microsoft.azurecr.io> --docker-username=<ContributorAppId> --docker-password=my-acr-password --docker-email=a@b.com
kubectl create -f k8-demo-app.yml
```


