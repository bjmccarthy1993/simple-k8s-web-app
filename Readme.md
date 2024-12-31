# Test project to learn Kubernetes and play around with different configurations

## Tutorial overview
1) Intro to Kubernetes
2) Main components
* Pod
* Service
* ConfigMap
* Secret
* Volume
* Ingress
* Deployment
* StatefulSet
* DaemonSet
3) Local Setup
4) Demo Project


## What is kubernetes?
* Container Orchestration tool
* Helps manage containerized applications in different deployed environments
* Rise of microservice architectures meant increaded usage of containers, meant need to manage many containers

## What does Kubernetes do?
* High availability (no downtime)
* Scalability (high performance)
* Disaster recovery - backup and restore


## Kubernetes Architecture
NOTE: K8s = Kubernetes

* At least one master node (node = virtual or physical machine) - control plane
* A few worker nodes (mostly referred to as nodes)
    * Each node has a kubelet process running on it (kubelet = primary "node agent")
        * Kubelet is a kubernetes process that makes it possible for the cluster to communicate and execute tasks on the nodes
    * Each worker node has collections of containers running on them
    * Worker nodes are generally the biggest part of the cluster since they have the majority of the containers, so have higher workloads and need the most resources.
* Master node runs several kubernetes processes that are necessary to run and manage the cluster properly. 
    * One of these is an API server (that is also a container). This is an entrypoint the the K8s cluster. This is the process that different kubernetes clients will talk to eg Kubernetes dashboard UI, API if you are using some scripts, and a command line tool.
    * Another process running on the master node is the Controller manager - Keeps track of what is happening in the cluster. Eg checks if a container has died and needs to be repaired or restarted.
    * Another process is a Scheduler - Ensures pod placement (Decides on which node a new Pod should be scheduled). Responsible for scheduling containers on different nodes based on the workload and the available server resources on each node.
    * etcd = Kubernetes backing store (key-value storage). Holds the current state of the K8s cluster. Has all the config data and status data of each node and each container within the nodes. Can recover the whole cluster state using the etcd snapshot.
    * NOTE: Master node only has a handful of processes, but is definitely the most important part since without the master node, nothing else will work. Because of this we must always have a backup of the master node, and in production environments you would usually have at least 2 master nodes inside of your K8s cluster.

There is also a virtual network.
* enables nodes to talk to each other
* spans all the nodes that are a part of the cluster
* turns all the nodes within the cluster into one powerful machine.

## Main K8s components

### Pod
Smallest unit in K8s is a 'Pod'. This is an abstraction over a container. Is a layer on top of the container. This is because we only want to be interacting with the k8s layer.
* Pods are meant to encapsulate 1 Application. Eg one main container and maybe a helper service or side service.
* Each pod gets it's own ip address, and each pod can communicate with other pods using the Virtual network.
* Pods are ephemeral which means they can die very easily, and when that happens the pod will die and a new one will be created in its place (with a new ip address). Because of this, another component called 'Service' is used.

### Service
Service is a static ip address (permanent ip address) that can be attached to each pod. That way, if a pod dies, the ip address won't change.
* You will probably want your application to be accessible through a browser. For this you would need to create an 'External Service'. This is a service that opens the application to communication from external sources. But obviously you wouldn't want your Database to be open to request from the public, so for this you would create an 'Internal Service'.
* You specify the type of service on creation. The default is 'Internal'.
* The url for the external service is not very practical. It would be in the form ```http://<node-ip>:<port>```, eg ```http://124.89.101.2:8080```. This is fine for testing and development, but for a production application you would want something more like this: ```https://my-app.com``` which has a secure protocol and a domain name. For that, there is another component of K8s called 'Ingress'.

### Ingress
When a request comes in for a service, instead of going to the service, it first goes to Ingress when then takes care of the forwarding of the request to the Service.

### Configmap and Secret
Pods communicate with each other using a service. Let's say in this example app, the application has a database endpoint called 'mongo-db-service' that it uses to communicate with the database.
* In a normal applicaiton this value would sit in some kind of application config, or in an environment variable. So something in the built image of the application. So if the value of some config changed (eg database endpoint name or value), you would have to adjust this in the application and rebuild, push to an image repository, pull it into the pod, and restart. So a little tedious for small changes. COnfigmap solves this problem.

#### Configmap
Configmap is basically your external configuration for your application eg urls for databases.
* In K8s, you connect the configmap to the pod, so that the pod gets the data that configmap contains
* If you want to change the value for some config, you just adjust the configmap, and thats it.
* Configmap is for non-confidential data only! Not for things like passwords.
* Configmap is stored in plaintext so is not ideal for secrets.

#### Secret
Secret is similar to configmap, but it is used to store secret data (eg passwords).
* Secrets are stored as base64 encoded.
* Storing them as base64 doesn't make it automatically secure.

* From the Docs:
```
Kubernetes secrets are, by default, stored unencrypted in the API server's underlying data store (etcd). Additionally, anyone who is authorized to create a pod in a namespace can use that access to read any Secret in that namespace -> This includes indirect access such as the ability to create a deployment.
In order to safely use secrets, take at least the following steps:

1) Enable Encryption at Rest for Secrets.
2) Enable or configure Role-Based Access Control (RBAC) rules that restrict reading data in Secrets (including via indirect means).
3) Where appropriate, also use mechanisms such as RBAC to limit which principles are allowed to create new Secrets or replace existing ones.
```

* So Secrets are meant to be encrypted using 3rd-party tools in K8s. There are tools you can use for this such as from cloud providers or separate 3rd party tools that you can deploy in kubernetes to encrypt your secrets.
* Secrets are things like credentials (passwords, certificates).
* Just like configmap, you connect Secret to your pod so that the pod can see the data, and read form the secret.
* You can use the data from configmap or secret inside of your application using Environment variables (or as a properties file).


### Volume
Data storage.

In the setup we are looking at in the video, the app and database are 2 separate containers within the pod.
One thing to note is that if the database container (or the entire pod) gets restarted, all of the data within the database would be gone, which is a problem.
We want our database data and log data to be persisted reliably and long term.

What we can do in K8s is use Volumes, which basically attaches some physical storage on a hard-drive to your pod.
* This storage could be on a local machine (on the same server node where the pod is running)
* This storage could also be one remote storage (outside of the K8s cluster) eg cloud storage, on-prem storage. K8s will simply have a reference to the storage.

This means that when the pod gets restarted, all of our data will still be there.
We can think of storage as an External Hard Drive that is plugged into our K8s cluster.
* K8s does NOT manage any data persistence.

### Deployment and Stateful Set

What happens if our application's pod dies or needs to be restarted? This would mean downtime.
The advantage of Distributed systems and containers using K8s is that we aren't relying on 1 application pod, we are replicating everything on multiple servers.
* We replicate everything on multiple servers.
* If a pod is replicated on another node (server), The replica will be connected to the same 'Service'.
    * Remember 'Service' has a static ip with a DNS name, so you don't need to adjust the endpoint when the pod dies.
    * 'Service is ALSO a load balancer, so it will take an incoming request and forward it to whichever pod is least busy.

#### Deployment
In order to create a replica, you wouldn't define a second pod. You would define a blueprint for the pod, and specify how many replicas of the pod you want.
* That blueprint is called 'Deployment'. In practice, you would not be creating pods, you would be creating Deployments, because there you can specify how many replicas.
    * You can also scale up or scale down the number of replicas of pods that you need.
* 'Pods' are a layer of abstraction over 'Containers', and 'Deployments' are a layer of abstraction over 'Pods'. Which makes it more convenient to interact with pods, and handle replication and configuration. So in practice, you would mostly work with deployments and not with pods.

So if one of the replicas of the application (or any) pod dies, the service would forward the request to a replica. But what about if the database pod died?
* We CANNOT replicate databases via deployment. This is because databases have state. Which means if we have replicas of the database, they would need to access the same shared data storage, which would need to manage which pods are reading and writing to the storage in order to manage data inconsistencies. Because of this need, we have another K8s mechanism called 'Stateful Set' (sts).

#### Stateful Set
This is meant specifically for applications like databases eg MySQL, MongoDB, Elastic Search, or any other stateful applications or databases. These should all be created with Stateful Sets, NOT Deployments.
* Just like Deployments, Stateful Sets take care of replicating pods, and scaling them up/down.
* ADDITIONALLY Stateful Set makes sure that database reads and writes are synchronized so that we don't get any database inconsistencies.
* NOTE: Deploying Stateful Set can be somewhat tedious. Much more difficult than working with deployments. Because of this, Databases are often hosted outside of a K8s cluster, while only having the Deployments and stateless applications that replicate and scale with no problems inside of the K8s cluster, and just have these communicate with an external database.



### Summary of components
* Pod (pod) - Abstraction over containers
* Service (svc) - manages communication between pods
* Ingress (ing) - route traffic into cluster
* ConfigMap (cm) - External configuration (non-secret)
* Secret (secret) - External configuration (secret)
* Volume (vol) - Data persistence
* Deployment (deploy) - Pod blueprint and Replication (stateless applications)
* StatefulSet (sts) - Pod blueprint and Replication (stateful applications like databases)


## Kubernetes Configuration

### API Server
All the configuration in a K8s cluster goes through a master node, with a process called API Server.

So Kubernetes clients such as a UI (eg Kubernetes dashboard) or an API (eg a script or a curl command) or a command line tool (such as kubectl, which is k8s CLI)
all talk to the API server, and send their configuration requests to the API server, which is the main (and only) entrypoint into the K8s cluster.

Requests must be in yaml or json format.

In the request, you specify things like the 'kind' of request (eg deployment), metadata (eg name of the app, labels etc), specs such as the number of replicas (pods) etc.
We can also configure the environment variables and port configuration here too.

These requests are declarative. We specify what we want the end state to be, not the steps to take to get there. K8s takes care of this for us by comparing the desired
state to the actual state, and taking the appropriate steps to reach the desired state.

Configuration consists of 3 parts:
(The first two lines specify the apiVersion and the kind of configuration (eg deployment, service))
* metadata
    * name (of the configuration file)
* spec (specification) - This is things like the number of replicas, the template etc. Theses are basically attributes, that will be different depending on the type
  of configuration we are providing (eg different for deployment and service).
* status - this is automatically generated and added by K8s. This is whether the desired state matches the actual state.
    * This is things like the currently available replicas, the last transition time, the last update time, any messages etc.
    * K8s updates status continuously

Where does K8s get the information to generate status from? This comes from etcd, which is basically the cluster brain which contains the current state of any k8s component (all parts of the cluster).

Note that it is a normal practice to store configuration files with our application code, or in it's own repository specifically for configuration files
(eg Infrastructure as Code).


## Setting up a cluster
Here we will set up a cluster using minikube and kubectl

### minikube
Usually in a production setup you would have multiple (at least 2) master nodes (each being a 'Control plane' and containing an API Server, Scheduler, Controller 
manager, and etcd), and multiple worker nodes (with worker nodes each containing at least 1 pod). Nodes are their own separate (physical or virtual) machines.

This is a lot to set up and manage (and may not be possible depending on resources etc) if you are just wanting to set up something for development and testing.

Because of this, we have a tool called minikube, which sets up master and worker node processes on ONE NODE (machine). This node has the docker container runtime
pre-installed, so you are able to run the containers (within pods) on this node.


### kubectl
We need a way to interact with the cluster, to create pods and other k8s components on our node (our minikube node). We can do this with kubectl, which is a command line
tool for k8s.

The API Server enables interaction with the cluster, and is the main entrypoint into the k8s cluster, so to configure anything in the cluster we will need to talk to
the API Server. We can do this with different clients such as a UI like the Kubernetes dashboard, or with the Kubernetes API, or a command line tool which is kubectl.
kubectl is the most powerful of these 3 methods, since we can use it to do basically anything that we want in the k8s cluster.

Once kubectl submits commands to the API Server to create components, delete components etc, the worker processes on the minikube node will execute these commands.
(eg create pods, create services, destroy pods)

It's important to note that kubectl isn't just for minikube. If you have a cloud cluster, or on-prem or hybrid cluster, kubectl is the tool that is used to interact
with any type of k8s cluster setup.

### Installing
I am installing into windows for this project, so I ran the follwing in powershell (taken from the minikube site):
```powershell
New-Item -Path 'c:\' -Name 'minikube' -ItemType Directory -Force
Invoke-WebRequest -OutFile 'c:\minikube\minikube.exe' -Uri 'https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe' -UseBasicParsing

```

I also needed to add the binaries to my PATH environment variables, which I did using the following:
```powershell
$oldPath = [Environment]::GetEnvironmentVariable('Path', [EnvironmentVariableTarget]::Machine)
if ($oldPath.Split(';') -inotcontains 'C:\minikube'){
  [Environment]::SetEnvironmentVariable('Path', $('{0};C:\minikube' -f $oldPath), [EnvironmentVariableTarget]::Machine)
}
```

Note that we need a driver for minikube. There are a list of supported drivers for each Operating System on the minikube website which includes docker, virtualbox,
hyper-v etc. The preferred driver for Windows, Linux, AND MacOS is Docker.

Minikube comes with docker pre-installed to run containers in pods, BUT we also use docker to run minukube on our own machine. These are different concepts and it is
important to know the difference. We are using Docker as a driver to host minikube as a container on our machine, which in turn uses docker to run containers within pods.

So from here, we can start minikube like so (with powershell runnning in admin mode):

```powershell
minikube start --driver=docker
```

We can also set docker to be the default driver like so:
```powershell
minikube config set driver docker
```

This will take a little longer the first time you run it, since minikube will need to download and setup some files.
Once it is up and running, we can check the status of the cluster with the following command:

```powershell
minikube status
```

So now we can interact with our cluster using kubectl. kubectl is installed as a dependency with minikube so no need to install separately.


To display all the nodes in our cluster, we can run:
```powershell
kubectl get node
```

In our case, we only have one node which acts as both our control plane and our worker nodes (since this is minikube).

We can stop our minikube cluster by running the following:
```powershell
minikube stop
```

### Summary of minikube and kubectl
* minikube - for start up/deleting the cluster
* kubectl - for configuring of the cluster



## Example project with mongodb:
This will be a simple project consisting of a MongoDB database, and a web application which will connect to the mongodb database (through Service) using external configuration
data from ConfigMap and Secret.
    * DB URL and credentials

We will make our web application accessible externally using Service (an External Service).

We will create:
* ConfigMap for MongoDB endpoint
* Secret for MongoDB Username and Password
* Deployment and (Internal) Service for MongoDB Application
* Deployment and (External) Service for WebApp (which is provided for us and will be downloaded from Dockerhub)

### ConfigMap Configuration File
```
- kind: "ConfigMap"
- metadata / name: an arbitrary name
- data: the actual contents (key-value pairs)

```

* In our config map for our mongodb database, our 'data' will contain a key 'mongo-url', and the value will be the Service name of MongoDB application which
  we will call 'mongo-service'

### Secret Configuration File
```
- kind: "Secret"
- metadata / name: an arbitrary name
- type: "Opaque" - default for arbitrary key-value pairs
- data: 
```

To encode text as base64, we can run the following:
```bash
echo -n <text-to-encode> | base64
```

-n sets a flag so that the newline character is not outputted. This makes a difference when we encode in base64 so is worth noting.

Note that I did this using WSL since the command line in windows didn't support it.

So for our project, we use:
* username: mongouser
* password: mongopassword

and we encode them as base64.

### Deployment and Service
Now that we have created Configmap and Secret resources, we can reference these from our deployments.


#### MongoDB
We will create a deployment and a service for our MongoDB pod. You can have these as separate yaml files, but these are commonly kept in
the same file because all deployments need services.

Deployment Configuration file:
```
- kind: "Deployment"

Main part - Blueprint for pods
- template: -> configuration for pod -> Why? Because deployment manages pod.
            ->  has its own "metadata" and "spec" section
    - containers -> Container specs for the pod. This could be multiple containers, but usually just one main application per pod.
                 -> We define which image will be used to create the container, and which ports to listen on.
                 -> We'll use the Mongo 5.0 image from dockerhub in our test application, but note that at this point in time (December 2024) this
                    has vulnerabilities and would not be suitable for a production application.
```

* We use port 27017 for our mongo container since that's what is specified for the image on dockerhub.

After setting up the containers section like so:
```yaml
containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
```
We have configured our deployment to create pods using the mongo:5.0 image

Labels:
* You can give any K8s component a label
    * Pod, deployment, configmap etc
* Labels are key/value pairs that are attached to k8s resources
* They are aditional identifiers to the components, which should be meaningful and relevant to users, eg:
    * "release": "stable"
    * "release": "canary"
    * "env": "dev"
    * "env": "production"
    * "tier": "frontend"
    * "tier": "backend"
So you can identify an address different components, using their labels.

* Labels do not provide uniqueness -> Eg all pod replicas will have the same label

Why do we need labels?
* Connecting deployment to all pod replicas:
    * When we have multiple replicas of the same pod, each pod will get a unique name, but will share the same label.
      So we can identify all replicas of the same pod, by using their label.
    * Because of this, labels are a required field for pods

In other components like Deployment and ConfigMap, labels are optional but it is a good practice to set them.

Label Selectors:
When we create pod replicas, How does Deployment know which pods belong to it? (or how does K8s know which pod belongs to which deployment?)

That is defined using this part, the label selector:
```yaml
spec:
  replicas: 3
  selector:       #####
    matchLabels:  #####
      app: mongo  #####
```

This defines that all the pods which match that specific label (in this case 'app: mongo') belong to the deployment that has the same label.
You could use any keyname, but the standard practice in kubernetes is to use 'app' as the key name.

We should also set the number of replicas. In this case we will set this to be 1 since this is a database, and managing consistency between these
is tricky. We have already learned that for databases we should use Stateful Set (or just host the database outside of the cluster which is a common practice)
but for the sake of our test application, we will just use a deployment and limit the replicas to 1.

Note that we get a warning about our container definition not containing resource limits, which could starve other processes. We'll ignore this
in our test setup, but would need to address this  in a production setup by limiting the memory (and maybe cpu, need to look more into this) by
doing something like this:
```yaml
spec:
      containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
      resources:
      limits:
        memory: 512Mi
        cpu: "1"
      requests:
        memory: 256Mi
        cpu: "0.2"
```


Service configuration:
We can use 3 dashes ```---``` to separate sections of a yaml file. Then beneath it we can add our service configuration.

```
- kind: "Service"
- name: an arbitrary name
- selector: this tells the service which pods to forward the requests to. So this needs to match the label of the pods for this service, which in our case is 'app: mongo'
- ports
```

Note that the 'name' that we give can now be referred to to get the endpoint of the service, which we are referring to in our config map.

Lastly for the ports, the Service is accessible within the cluster using its own IP address, which is defined by the port, and we also
need to tell the service the port that the pods are listening on so that requests can be forwarded to that port. So 'targetPort' in the Service should always match the
'containerPort' in the Deployment.
```yaml
  ports:
    - protocol: TCP
      port: 8080 ########## Port of the Service
      targetPort: 27017  ######### Port for the pods that belong to the service
```

Note that it is also a common standard to match the port of the service to the port of the containers within the pods, just for simplicity's sake. So our configuration for the Service
becomes this:
```yaml
  ports:
    - protocol: TCP
      port: 27017 ########## Port of the Service
      targetPort: 27017  ######### Port for the pods that belong to the service
```

#### Web App

We can copy-paste our Deployment and Service for MongoDB into a new file for the web app, and adjust a few things. We change our labels to be webapp,

The image we'll use is ```nanajanashia/k8s-demo-app:v1.0``` which is hosted publically on dockerhub from the creator of the tutorial.
This is a simply nodejs application which starts on port 3000.


This completes the setup for our deployment and  service configuration for the web app and mongodb. All deployment and service configuration would have a basic setup similar to this (for any other app), plus some extra more specific config for things like security and resource limiting.

### Passing Secret and ConfigMap data into the pods

When starting a mongodb application, we need to set the username and password. When mongo db starts, it will generate a new user and set that user's credentials based on the username and password we pass in.
We can then use these credentials to access the db within our cluster.

In the image documentation for our mongodb image in dockerhub, we can see the specifications for the environment variables which we need to pass into the container. Note that these are required fields (and usually are in databases), so if we don't set them then we won't be able to access the database.

We can set environment variables for our container like so:
```yaml
    spec:
      containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
        env:
        - name: <environment_variable_name>
          value: <environment_variable_value> ##### Set the value directly
        - name: <environment_variable_name>
          valueFrom: 
            secretKeyRef:
                name: <secret_file_name> ###### Our secret file
                key: <secret_key_name> ####### The key from our secret
```

We can set the value directly, or we can reference a value from secret and config components.

IMPORTANT: Secret config file should NOT be checked into the Git repo!

After configuring our mongodb container to reference data from the secret, we also need to set up our web app to get data from the configmap.

NOTE: The web app provided has been configured to expect environment variables, and use these when accessing mongodb.

We can pass environment variables from configmap in a similar way:

```yaml
- name: <environment_variable_name>
  valueFrom:
    configMapKeyRef:
      name: <config_file_name>
      key: <config_key_name>
```

### Configuring the external service
The last thing we need to do is make our web application accessible from the browser. We simply need to adjust the service for our web app to be external.

By default, the type of our Service is 'ClusterIP' which is internal:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: ClusterIP ##### This is the default, we can usually just ommit the type
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
```

We can instead set this to 'NodePort' which sets it as an external service. We must also set the 'nodePort', which exposes the Service on each Node's IP address at a static port.

Node Port's range is defined in kubernetes, so we can't just set it as anything we want. It must be within the range 30000-32767.
So the address the request are sent to will end up being ```<NodeIP>:<NodePort>```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort    #### Specifies the service  as external
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30100    ## Sets the port that this service is accessible from
```


### Deploying all resources to minikube
Our minikube cluster is already running, and we can see the node, but if we run:
```kubectl get pod``` we will see that there are no resources found.

We first need to create the external configurations (configMap and secret) because these must exist when we create the mongodb and web app deployments since they reference the configs.

We will need to apply our yaml files using kubectl. 'apply' manages applications through files defining k8s resources.

To apply a configuration, we can run:
```kubectl apply -f <file-name.yaml>```
-f specifies that we are passing in a file

For our project, we open up a terminal in the same directory as our yaml files and run:
```kubectl apply -f mongo-config.yaml```

We should get a message back that the config was created.

We then create our secret:
```kubectl apply -f mongo-secret.yaml```

Next we create our database because our web application depends on it, so it should start first:
```kubectl apply -f mongo.yaml```
This should return a message saying that our deployment and service was created.

Lastly we create our web app:
```kubectl apply -f webapp.yaml```

Now when we run ```kubectl get pod``` we should see our resources. They will probably have a status of 'ContainerCreating' depending on how soon after creating you ran the 'get' command.


### Interacting with our k8s cluster
We can run ```kubectl get all``` to get all the components created in the cluster including: deployments, pods behind the deployment, and all the services.

To get the configmap and secret, we can run:
```kubectl get configmap``` and
```kubectl get secret```

We can use ```kubectl --help``` to view the documentation if we are not sure of the commands we need to run.

To see more detail of a component, we can use ```kubectl describe``` eg:
```kubectl describe service webapp-service```
would give us more detail for the service instance with name 'webapp-service'.

```kubectl describe pod <podname>``` is useful for getting detailed information about the pod, including configuration and event history.

To get the logs for a pod, we can run ```kubectl logs <podname>```
This is useful for troubleshooting, debugging, or just making sure that everything is fine within the pod.

We can also stream the logs using the -f flag like so:
```kubectl logs <podname> -f```

### Validating that we can access the webapp from the browser
We can run ```kubctl get svc``` (svc or service will both work)

We can see that the webapp-service is running, and is of type NodePort.
But which IP address to use?
The NodePort Service is always accessible at the IP address of the cluster node (Worker Node's IP Address).

In our particular case we only have one node (minikube), so we need the IP address of the minikube node. To get that we can run:
```minikube ip```
or we can use kubernetes  to get the nodes ```kubectl get node```.
To get a longer output than the default output we can use:
```kubectl get node -o wide```, and we will be able to see the IP address
of the node (INTERNAL-IP). This should match the address from the minikube command.

Note that the ```-o wide``` command works for other 'get' commands for kubectl too, to get some more detail than the default.

Finally we can use this address, append the port of the webapp service (30100 in our case), and enter it as the browser url.


### Fixes
When I tried to access the webapp from the browser using the ip address for minikube with the port for the webapp, it could not be found.

I came accross this comment on the youtube video for this tutorial:
```
I was following your tutorial on macOS Ventura with Docker Desktop v4.17 and it seems that Docker changed the way it integrates with host's network. What you shown will still work on Linux where Docker adds a new interface to the host OS through which you can communicate with containers. In my case, host OS had no clue how to find the k8s server with the IP returned by `minikube ip`, there was no routing information in the routing table.

What you have to do is to run `minikube service webapp-service` which will create a tunnel to the service via ports published by minikube container and redirect the browser to tunneled port.
```

So I ran the command:
```minikube service webapp-service```

This created a tunnel to the service which I could access from the browser.

