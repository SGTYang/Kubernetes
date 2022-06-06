# Kubernetes Study

# ETCD

---

ETCD stores data 

- Node
- Pods
- Secrets
- Accounts
- Roles
- Configs
- Bindings
- Others

## Setup - Manual

---

If you setup your cluster from **scratch**, you deploy etcd by downloading etcd binary yourself(Install etcd on your master node)

Make sure etcd listen right address 

```bash
/usr/local/bin/etcd
```

```yaml
--advertise-client-urls https://${INTERNAL_IP}:2379 \\
```

## Setup - kubeadm

---

If you deploy etcd using **kubeadm**, etcd is deployed as a pod in the kube-system namespace

```bash
kubectl get pods -n kube-system
```

To list all keys stored in etcd

```bash
kubectl exec etcd-master -n kube-system etcdctl get / --prefix-keys-only
```

## ETCD in HA Environment

---

In HA environment, you will have multiple master node in your cluster and you will have multiple etcd instances across the master nodes. In that case, make sure etcd instances know each other by setting right parameter.

```bash
/usr/local/bin/etcd
```

```yaml
...
--initial-cluster controller-0=https://${CONTROLLER0_IP}:2380, controller-1=https://${CONTROLLERS1_IP}:2380
...
```

# ETCD - Commands

---

Etcdctl can interact with ETCD Server using 2 API versions - Version 2 and Version 3. By default its set to use Version 2. Each version has different sets of commands.

To set the right version of API set the environment variable ETCDCTL_API command

```jsx
export ETCDCTL_API=3
```

Apart from that, you must also specify path to certificate files so that ETCDCTL can authenticate to the ETCD API Server. The certificate files are available in the etcd-master at the following path. 

```yaml
--cacert /etc/kubernetes/pki/etcd/ca.crt
--cert /etc/kubernetes/pki/etcd/server.crt
--key /etc/kubernetes/pki/etcd/serverl.key
```

You must specify the ETCDCTL API version and path to certificate files.

```bash
	kubectl exec etcd-master -n kube-system -- sh -c "ETCDCTL_API=3 
etcdctl get / --prefix --keys-only --limit=10 --cacert 
/etc/kubernetes/pki/etcd/ca.crt --cert 
/etc/kubernetes/pki/etcd/server.crt  --key 
/etc/kubernetes/pki/etcd/server.key"
```

# kubectl

---

kubectl run nginx —image nginx

kubectl get pods

# YAML in Kubernetes

---

Kubernetes uses YAML as input to create applications

root levels properties(must have in your configuration):

```yaml
**apiVersion**: v1
**kind**: Pod
**metadata**:
	name: myapp-pod
	labels:
		app: myapp
		type: front-end

**spec**:
	containers:
	- name: nginx-container
		image: nginx

```

| Kind | apiVersion |
| --- | --- |
| Pod | v1 |
| Service | v1 |
| ReplicaSet | apps/v1 |
| Deployment | apps/v1 |

Labels is a dictionary. It can have any key and value pairs as you wish.  

Spec is a dictionary. 

```bash
kubectl create -f pod-definition.yml
```

# Replication Controller, Replica Set

---

Replication Controller is the older technology that is being replaced by replicas set up.

We would like to have more than one instance or pod running at the same time. That way, if one fails, we still have our application running on the other one.

Replication controller helps us run multiple instances of a single pod in the K8S cluster.

## Load Balancing & Scaling

When the number of users increase, we deploy additional pod to balance the load across the pods. If the demand further increases and if we were to run out of resources on the first node, we could deploy additional parts across the other nodes in the cluster.

## Labels and Selectors

The role of the replica set is to monitor the pods and if any of them were failed, deploy new ones. The replica set is in fact a process that monitors the pods. How does the replica set know what pods to monitor? There could be hundreds of other parts in the cluster running different applications. This is where labeling our pods during creation comes in handy. We could now provide these labels as a filter for replica set. Under this selector section, we use to match labels filter and provide the same label that we used while creating the pods. This way the replica set knows which parts to monitor.

## Scale

```bash
# edit replicas section
kubectl replace -f replicaset-definition.yml

#replica set section will still remain as before
kubectl scale --replicas=6 -f replicaset-definition.yml

kubectl scale --replicas=6 replicaset myapp-replicaset
```

# Deployment

---

How you might want to deploy your application in a production environment. Let’s say that you have a web server that needs to be deployed in a production environment. You need not one but many such instances of the web server running for obvious reasons. Whenever newer versions of application builds become available on the docker registry you would like to upgrade your docker instances seamlessly. When you upgrade your instances you do not want to upgrade all of them at once may impact users accessing our applications, so you might want to upgrade them one after the other known as rolling updates.

## Create an Pod yaml file

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml

kubectl create deployment --image=nginx nginx --dry-run=client -o yaml
```

# Services

---

Services enable communication between various components within and outside of the application. 

## NodePort

We will use labels and selectors to link together.

```yaml
apiVersion: v1
kind: Service
metadata:
	name: myapp-service

spec:
	type: NodePort
	ports:
	- targetPort: 80
		port: 80
		nodePort: 30008
	selector:
		app: myapp
		type: front-end
```

What do you when you have multiple pods. In a production environment you have multiple instances of your web application running for high availability and load balancing purposes. Let’s say we have multiple similar pods running our web application they all have the same labels with a key. The same label is used as a selector during the creation of the service. So when the service is created it looks for a matching pod with the label and finds them. They service then automatically selects all the pods as endpoints to forward the external requests. coming from the user. The service acts as a built in load balancer to distribute load across the different pods.

Let us look at what happens when the pods are distributed across multiple nodes. When we create a service without us having to do any additional configuration K8S automatically creates a service that spans across all the nodes in the cluster and maps the target port to the same node port on all the nodes in the cluster this way you can access your application using the Ip of any node in the cluster and using the same port number.

## Cluster Ip

Full stack web application typically has different kinds of pods hosting different parts of an application. You may have a number of pods running a front end web server another set of pods running a back end server, a set of pods running a key-value store like Redis, another set of pods running a persistent database like MySQL. The web front end server needs to communicate to the backend servers and the backend servers need to connect to database as well as the redis service … 

So what is the right way to establish connectivity between these services or tiers of my application. 

 The pods all have an Ip address assigned to them. But these Ip as we know are not static. These pods can go down any time and new pods are created all the time and so you can not rely on theses Ip addresses for internal communication between the application.

# Namespace

---

If we want to switch to the default namespace to dev run this command.

```bash
kubectl config set-context $(kubectl config current-context) --namespace=dev
```

## Resource Quota

To limit resources in a namespace. Create a resource quota. To create one 

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
	name: compute-quota
	namespace: dev

spec: 
	hard:
		pods: "10"
		requests.cpu: "4"
		requests.memory: 5Gi
		limits.cpu: "10"
		limits.memory: 10Gi
```

# Imperative vs Declarative

---

There are different approaches in managing the infrastructure, and they are classified into imperative and declarative approaches. 

An example of imperative approach of provisioning infrastructure would be a set of instructions written step by step, such as provisioning a VM, a Web server, installing the engine nginx software editing configuration file to user Port 8080 and setting the path to Web files, downloading source code of repository. So here we’re saying what is required and also how to get things done in the declarative approach.

In declarative approach, we declare out requirements, for instance. All we say is that we need VM by the name, Web server with the nginx software on it with a port 8080 and the path to the Web files defined and where the source code of the application is stored. Everything that’s needed to be done to get this infrastructure in place is done by the system are the software. You don’t have to provide step by step instructions, orchestration, tools.

## ****Imperative Commands with Kubectl****

Before we begin, familiarize with the two options that can come in handy while working with the below commands:

- `-dry-run`: By default as soon as the command is run, the resource will be created. If you simply want to test your command , use the `-dry-run=client` option. This will not create the resource, instead, tell you whether the resource can be created and if your command is right.
- `o yaml`: This will output the resource definition in YAML format on screen.

Use the above two in combination to generate a resource definition file quickly, that you can then modify and create resources as required, instead of creating the files from scratch.

# Kubectl apply command

---

The Apply command takes into consideration the local configuration file, a live object definition on K8S and the last applied configuration before making a decision on what changes are to be made.

So when you run the Apply command if the object does not already exist, the object is created. An object configuration similar to what we created locally is created within K8S but with additional fields to store status of the object.

When you use the kubectl apply command to create an object it does something bit more. The Yaml version of the local object configuration file is converted to a Json format and it is then stored as the last applied configuration. Going forward for any updates to the object. All the three are compared to identify what changes are to be made on the live object.  

# Manual Scheduling

---

To manually schedule pods, you can create a binding object and send a post request to the pod binding API.

```yaml
apiVersion: v1
kind: Binding
metadata:
	name: nginx
target:
	apiVersion: v1
	kind: Node
	name: node02
```

```bash
curl --header "Content-Type:application/json" --request POST --data '{"apiVersion":"v1", "kind": "Binding" ,...}

```

# Label Select

---

To select the pod with the labels use the kubectl get pod command along with the selector option and specify the condition.

```bash
kubectl get pods --selector app=App1
```

e.g.) How many Pods exist in the dev env?

```bash
kubectl get pods --selector env=dev --no-headers | wc -l
```

e.g.) How many objects are in the prod environment?

```bash
kubectl get all --selector env=prod --no-headers | wc -l
```

e.g.) Identify the pod which is part of the prod env, finace bu and frontend tier?

```bash
kubectl get all --selector env=prod,bu=finance,tier=frontend
```

# Taints and Tolerations

---

## Taints

To taint nodes 

```bash
kubectl taint nodes node-name key=value:taint-effect
```

There are three taint-effect 

- NoSchedule: Pod will not be scheduled.
- PreferNoSchedule: They system will try to avoid placing a part on the node.
- NoExecute: New pods will not be scheduled on the node and existing on the node if any will be evicted if they don’t tolerate the taint.

## Tolerations

Tolerations are added to pods. To add toleration add tolerations filed under spec filed

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: myapp-pod
spec:
	containers:
	- name: nginx-container
		image: nginx
	
	tolerations:
	- key:"app"
		operator:"Equal"
		value:"blue"
		effect:"NoSchedule"
```

When the pods are created or updated with the new tolerations, they either not scheduled on nodes or evicted from the existing nodes depending on the effect set.

```bash
kubectl get pods -o wide
```

## To remove taint

```bash
kbuectl taint node nodename key=value:tainteffect-
```

## Node Selectors

---

When we want to set limitation on the pods so that they only run on particular nodes. There are two way to do this. The first is using node selectors.

## Node Selectors

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
	containers:
	- name: data-processor
		image: data-processor
	
	nodeSelector:
		size: Large
```

## Label Nodes

```bash
kubectl label nodes nodename key=value
```

What if our requirement is much more complex? Then you use node affinity and anti affinity features.

# Node Affinity

---

The primary purpose of node affinity feature is to ensure that pods are hosted on particular nodes.

```yaml
apiVersion: v1
kind: Pod

metadata:
	name: myapp-pod
spec:
	containers:
	- name: data-processor
		image: data-processor
	
	affinity:
		nodeAffinity:
			requiredDuringSchedulingIgnoredDuringExecution:
				nodeSelectorTerms:
				- matchExpressions:
					-key: size
					 operator: In
					 values:
					 - Large
```

The operator ensures that the pod will be placed on a node whose label size has any value in the list of values specified.

If you think your pod could be placed on a large or a medium node you could simply add the value to the list of values.

```yaml
apiVersion: v1
kind: Pod

metadata:
	name: myapp-pod
spec:
	containers:
	- name: data-processor
		image: data-processor
	
	affinity:
		nodeAffinity:
			requiredDuringSchedulingIgnoredDuringExecution:
				nodeSelectorTerms:
				- matchExpressions:
					- key: size
					 operator: In
					 values:
					 - Large
					 - Medium
					 - Small
```

```yaml
apiVersion: v1
kind: Pod

metadata:
	name: myapp-pod
spec:
	containers:
	- name: data-processor
		image: data-processor
	
	affinity:
		nodeAffinity:
			requiredDuringSchedulingIgnoredDuringExecution:
				nodeSelectorTerms:
				- matchExpressions:
					- key: size
					 operator: Not In
					 values:
					 - Large
```

If you set operator Not In, node affinity will match the node with values are not matched. 

```yaml
apiVersion: v1
kind: Pod

metadata:
	name: myapp-pod
spec:
	containers:
	- name: data-processor
		image: data-processor
	
	affinity:
		nodeAffinity:
			requiredDuringSchedulingIgnoredDuringExecution:
				nodeSelectorTerms:
				- matchExpressions:
					-key: size
					 operator: Exists 
```

The exists operator will simply check if the label exists on the node and you don’t need values for that as it does not compare the values.

There are number of operators.

## Node Affinity Types

Type of node affinity defines the behaviour of the scheduler with respect to node affinity and the stages in the lifecycle of the pod.

There are currently two types of node affinity available.

```yaml
requiredDuringSchedulingIgnoredDuringExecution

preferredDuringSchedulingIgnoredDuringExecution=
```

## Label nodes

```bash
kubectl label node nodename key=value
```

## Get certain fields items from describe command

```bash
kubectl describe node nodename | grep Taints
```

# Taints/Tolerations And Node Affinity

---

This can be used to completely dedicated nodes for specific pods. We first use taints and tolerations to prevent other pods from being placed on the nodes, and we use node affinity to prevent our pods from being placed on their nodes.

# Resource Requirements and Limits

---

 If the node has no sufficient resources, the scheduler avoids placing the pod on that node, instead places the pod on  one where sufficient resources are available. If there is no sufficient resources available on any of the nodes, K8S hold back scheduling. You will see the pods on the pending states.

By default K8S assumes that a pod or container within a pod requires 0.5CPU and 256 MB memory. This is known as the resource request for a container, the minimum amount of CPU or memory requested by the container. When the scheduler tries to place the pod on a node, it uses these numbers to identify a node which has sufficient amount of resources available. 

If your application will need more than this, you can modify these values by specifying them in your pod or deployment definition files.

Add a section called Resources

```yaml
...
spec:
	containers:
	- name: simple-webapp-color
		image: simple-webapp-color
		ports:
			- containerPort: 8080
		resources:
			requests:
				memory: "1Gi"
				cpu: 1
```

 If you don’t specify explicitly, a container will be limited to consume only one vCPU from the node. The same goes with memory. By default, K8S sets a limit of 512 mb on containers. If you don’t like the default limit, you can change them by adding a limited section under the resources.

```yaml
...
spec:
	containers:
	- name: simple-webapp-color
		image: simple-webapp-color
		ports:
			- containerPort: 8080
		resources:
			requests:
				memory: "1Gi"
				cpu: 1
			limits:
				memory: "2Gi"
				cpu: 2 
```

What happens when a pod tries to exceed resources beyond its specified limit? In case of CPU, K8S throttles the CPU so it does not go beyond the specified limit. A container cannot use more CPU resources than its limit. 

However this is not the case with the memories. A container can use more memory resources than its limit. So if a pod tries to consume more memory than its limit constantly, the pod will be terminated. The status `OOMKilled`indicates that it is failing because the pod ran out of memory.

 "When a pod is created the containers are assigned a default CPU request of .5 and memory of 256Mi". For the POD to pick up those defaults you must have first set those as default values for request and limit by creating a LimitRange in that namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
	name: mem-limit-range
spec:
	limits:
	- default:
	  memory: 512Mi
	defaultRequest:
		memory: 256Mi
	type: Container
```

# Edit a POD

Remember, you CANNOT edit specifications of an existing POD other than the below.

- spec.containers[*].image
- spec.initContainers[*].image
- spec.activeDeadlineSeconds
- spec.tolerations

For example you cannot edit the environment variables, service accounts, resource limits (all of which we will discuss later) of a running pod. But if you really want to, you have 2 options:

1. Run the `kubectl edit pod <pod name>` command.  This will open the pod specification in an editor (vi editor). Then edit the required properties. When you try to save it, you will be denied. This is because you are attempting to edit a field on the pod that is not editable.
    
    A copy of the file with your changes is saved in a temporary location.
    
    You can then delete the existing pod by running the command:
    
    `kubectl delete pod webapp`
    
    Then create a new pod with your changes using the temporary file
    
    `kubectl create -f /tmp/kubectl-edit-ccvrq.yaml`
    

1. The second option is to extract the pod definition in YAML format to a file using the command
    
    `kubectl get pod webapp -o yaml > my-new-pod.yaml`
    
    Then make the changes to the exported file using an editor (vi editor). Save the changes
    
    `vi my-new-pod.yaml`
    
    Then delete the existing pod
    
    `kubectl delete pod webapp`
    
    Then create a new pod with the edited file
    
    `kubectl create -f my-new-pod.yaml`
    

# Edit Deployments

With Deployments you can easily edit any field/property of the POD template. Since the pod template is a child of the deployment specification,  with every change the deployment will automatically delete and create a new pod with the new changes. So if you are asked to edit a property of a POD part of a deployment you may do that simply by running the command

`kubectl edit deployment my-deployment`

# DaemonSets

---

How does daemon sets work? How does it schedule pods on each node and how does it ensure that every node has a pod? We can set node name property on the pod to bypass the scheduler and get the pod placed on a node directly.

Daemon set uses the default scheduler and node affinity rules to schedule pod on nodes. 

# Static Pod

---

kubelet relies on the kube-apiserver for instructions on what pods to load its node which was based on a decision made by the kube-scheduler which was stored in the ETCD datastore.

What if there was no kube-apiserver, and kube-scheduler and no controllers and no ETCD cluster. What if there were no master at all. What if there were no other nodes. Is there anything that the kubelet can do? Can it operates as an independent node? 

The kubelet can manage a node independently. The one things that the kubelet knows to do is create pods. But we don’t have API server to provide pod details. By now we now that to create a pod you need the details of the pod in a pod definition file.

How do you provide a pod definition file to kubelet without a kube-api server? 

You can configure the kubelet to read the pod definition files from a directory on the server designated to store information about pods. Kubelet periodically checks the directory. Not only does it create the pod it can ensure that the pod stays alive. If the application crashes, the kubelet attempts to restart it.

So these pods that are created by the kubelet on its own without the intervention from the API server or rest of the K8S cluster components are known as static pod. You can only create a pod like this way. You cannot create replicasets or deployments or services by placing a definition file in the designated directory.

You can this under /etc/Kubernetes/manifests

```bash
kubelet.service
...
--container-runtime=remote \\
--container-runtime-endpoint=unit:///var/run/containerd/containerd.sock \\
--pod-manifest-path=/etc/Kubernetes/manifests \\
... 
```

You can also provide a path to another config file using the config option, and define the directory path as staticPod path in that file.

```bash
kubelet.service
...
--container-runtime=remote \\
--container-runtime-endpoint=unit:///var/run/containerd/containerd.sock \\
--kubeconfig=/var/lib/kubelet/kubeconfig \\
...
```

```yaml
kubeconfig.yaml
staticPodPath: /etc/kubernetes/manifests
```

Clusters setup by the kubeadmin tool uses this approach. If you are inspecting an existing cluster you should inspect this option of the kubelet to identify the path to the directory. You will then know where to place the definition file for your static pods.

First check the option pod manifest path in the kubelet service file if it’s not there then look for the config option and identify the file used as the config file and then within the config file look for these static pod path option.

### Use case

**Since static pods are not dependent on the k8s control plane, you can use static pods to deploy the control plane components itself as pods on a node.** 

**Start by installing kubelet on all the master node then create pod definition files that uses docker images of the various control plane components such as api server, controller, etcd etc. Place the definition files in the designated manifests folder and kubelet takes care of deploying the control plane components themselves as pods on the cluster.**

**If any of these services were to crash since it’s a static pod it will automatically be restarted by kubelet.**

# Multiple Schedulers

---

In a K8S environment, it has an algorithm that distributes pods across nodes evenly as well as takes into consideration the various conditions we specify through taints & tolerations and node affinity etc. But what if none of these satisfies your needs? Say you have a specific application that requires its components to be placed on nodes after performing some additional checks. 

K8S can have multiple schedulers at the same time. 

This kube-scheduler is the default scheduler. To deploy additional scheduler, you can use the same kube-scheduler binary or use on that you might have built for yoursel, which makes 

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-scheduler
```

kube-scheduler.service

```bash
ExecStart=/usr/local/bin/kube-scheduler \\
	--config=/etc/kubernetes/config/kube-scheduler.yaml \\
	--scheduler-name=my-custom-scheduler
```

my-custom-scheduler.service

```bash
ExecStart=/usr/local/bin/kube-scheduler \\
	--config=/etc/kubernetes/config/kube-scheduler.yaml \\
	--scheduler-name=my-custom-scheduler
```

Let’s take a look at how it works with the kubeadm tool. The kubeadm tool deploys the scheduler as pod. You can find the definition file it uses under the manifests folder. I have removed all the other details from the file so we can focus on the key parts.

/etc/kubernetes/manifests/kube-scheduler.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kube-scheduler
	namespace: kube-system
spec:
	containers:
	- command:
		- kube-scheduler
		- --address=127.0.0.1
		- --kubeconfig=/etc/kubernetes/scheduler.conf
		- --leader-elect=true
		image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
		name: kube-scheduler
```

my-custom-scheduler.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kube-scheduler
	namespace: kube-system
spec:
	containers:
	- command:
		- kube-scheduler
		- --address=127.0.0.1
		- --kubeconfig=/etc/kubernetes/scheduler.conf
		- --leader-elect=true
		- --lock-object-name=my-custom-scheduler

		image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
		name: kube-scheduler
```

The command section has the command and associated options to start the scheduler.

The leader-elect option is used when you have multiple copies of the scheduler running on different master nodes. In HA setup where you have multiple master nodes with the kube-scheduler process running on both of them. If multiple copies of the same scheduler are running on different nodes only one can be active at a time. That’s where the leader-elect option helps in choosing a leader who will lead scheduling activites.

Set leader-elect option to false in case you don’t have multiple masters. In case you do have multiple master you can pass in additional parameter to set a lock object-name. This is to differentiate the new custom scheduler from the default during the leader election process.

Once done, create the pod using the kubectl create command and run the get pods command in kube-system. 

Next step is to configure a new pod or a deployment to use the new scheduler. Add a new filed called schedulerName and specify the name of the new scheduler.

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: nginx
spec:
	containers:
	- image: nginx
	  name: nginx
	
	schedulerName: my-custom-scheduler
```

### To view the events using the kubectl get events command.

```bash
kubectl get events
```

### To view the logs of the pod using the kubectl logs command

```bash
kubectl logs my-custom-scheduler --name-space=kub
```

## Configmap

```bash
kubectl create configmap my-scheduler-config --from-file=/root/my-scheduler-config.yaml -n kube-system
configmap/my-scheduler-config created
```

# Logs

---

```bash
kubectl logs -f pod-name container-name
```

# Rollout and Versioning

---

When you first create a deployment, it triggers a rollout, a new rollout creates a new deployment revision. In the future, when the application is upgraded, meaning when the container version is updated to a new one, a new rollout is triggered and a new deployment revision is created. This helps us keep track of the changes made to our deployment and enables us to roll back to a previous version of deployment.

You can see the status of your rollout status by running the command

```bash
kubectl rollout status deployment-name
```

To see revisions and history of rollout 

```bash
kubectl rollout history deployment-name
```

# Deployment Strategy

---

### Rolling Update

Rolling update is the default deployment strategy

 

### To undo update

```bash
kubectl rollout undo deployment/deployment-name
```

 

# Container

---

Let’s say that you were to run a docker container from an Ubuntu image. When you run the “docker run ubuntu” command, it runs an instance of Ubuntu image and exits immediately. Why is that? Unlike virtual machines, containers are not meant to host an operating system. Containers are meant to run specific task or process such as to host an instance of a web server and etc. Once the task is complete, the container exits container, the container exits. A container only lives as long as the process inside it is alive.

If you look at the docker file like Nginx, you will see an instruction

```bash
# Install Nginx.
RUN \
	add-apt-repository -y ppa:nginx/stable && \
	apt-get update && \
	apt-get install -y nginx && \
	rm -rf /var/lib/apt/lists/* && \
	echo "\ndaemon off;" >> /etc/nginx/nginx.confg && \
	chown -R www-data:www-data /var/lib/nginx

#Define mountable directories.
VOLUME ["/etc/nginx/sites-enabled", "/etc/nginx/certs", "/etc/nginx/config"]

#Define working directory.
WORKDIR /etc/ngixn

#Define default command.
CMD ["nginx"]
```

called CMD which stands for command that defines the program that will be run within the container when it starts. It is the nginx command.

Let us look at the docker file for this image

```bash
#Pull base image.
FROM ubuntu:18.04

#Install
RUN \
	sed -i 's/# \(.*multiverse$\)/\1/g' /etc/apt/sources.list && \
	apt-get update && \
	apt-get -y upgrade && \
	apt-get install -y build-essential && \
	apt-get install -y software-properties-common && \
	...

#Define default command.
CMD ["bash"]
```

You will see that it uses bash as the default command. Bash is not really a process like a web server or database server. It is a shell that listens for inputs from a terminal if it cannot find a terminal it exits.

# Specify command to start the container.

---

1. Append a command to the docker run command and that way it overrides the default command specified within the image.

```bash
#e.g.
docker run ubuntu sleep 5

#Docker 
FROM Ubuntu
CMD sleep 5

#Or
FROM Ubuntu
CMD ["sleep","5"] #parameters should be separate elements in the list
```

1. Entry point instruction is like the command instruction as in you can specify the program that will be run  when the container starts.

```bash
FROM Ubuntu
ENTRYPOINT ["sleep"]
```

<aside>
⚠️ In case of the CMD instruction the command line parameters passed will get replaced entirely, whereas in case of entrypoint the command line parameters will get appended.

</aside>

How do you configure a default value for the command? If one was not specified in the command line. That’s where you would use both entrypoint as well as the command instruction. In this case the command instruction will be appended to the entry point instruction.

```bash
FROM Ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

How do you specify additional argument in the pod definition file.

```yaml
apiVersion: v1
kind: Pod
...
spec:
	containers:
		- name: ubuntu-sleeper
			image: ubuntu-sleeper
			command: ["sleep"]
			args: ["10"]
```

Anything that is appended to the docker run command will go into the args property of the pod definition file in the form of the array.

**The command field overrides the enrtypoint instruction and the args field overrides the command instruction**

# Env Variables in K8S

---

To set an environment variable, use the ENV property. Env is an array. Every item under the Env property starts with a dash indicating an item in the array. Each item has a name and a value property.

There are other ways of setting the environment variables such as using config maps and secrets

# ConfigMaps

---

When you have a lot of pod definition files it will become difficult to manage the environment data stored within the query files. We can take this information out of the pod definition file and manage it centrally using Configuration Mpas. 

ConfigMaps are used to pass configuration data in the form of key value pairs in K8S. When pod is created inject the config map into the pod.

```yaml
ConfigMap

APP_COLOR: blue
APP_MODE: prod
```

So the key value pairs that are available as environment variables for the application hosted inside the container in the pod

```yaml
pod-definition.yaml

apiVersion: v1
...
spec:
	...
	envFrom:
	- configMapRef:
		name: app-configmap
```

There are two phases involved in configuring ConfigMaps. First create the ConfigMaps and second inject them into the pod.

## Create ConfigMaps

### Imperative

```bash
kubectl create configmap \
	app-config --from-literal=APP_COLOR=blue
```

**—from-literal**

The option is used to specify the key value pairs in the command itself. If you wish to add additional key value pairs simply specify the from literal options multiple times. 

```bash
kubectl create configmap \
	app-config --from-file=app_config.properties
```

**—from-file**

The option is used to specify a path to the file that contains the required data. The data from this file is read and stored under the name of the file.

### Declarative

```yaml
config-map.yaml

apiVersion: v1
kind: ConfigMap
metadata:
	name: app-config
data:
	APP_COLOR: blue
	APP_MODE: prod
```

e.g.)

```yaml
mysql-config

port: 3306
max_allowed_packet: 128M
```

```yaml
redis-config

port: 6379
rdb-compression: yes
```

# ConfigMap in Pods

---

pod-definition.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: simple-webapp-color
	labels:
		name: simple-webapp-color
spec:
	containers:
	- name: simple-webapp-color
		image: simple-webapp-color
		ports:
			- containerPort:8080
		envFrom:
			- configMapRef:
					name: app-config
```

config-map.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: app-config
data:
	APP_COLOR: blue
	APP_MODE: prod 
```

You can inject it as a single environment or the whole data as files in a volume.

```yaml
env:
	- name: APP_COLOR
		valueFrom:
			configMapKeyRef:
				name: app-config
				key: APP_COLOR
```

```yaml
volumes:
- name: app-config-volume
	configMap:
		name: app-config
```

# Secrets

---

app.py

```python
import os
from flask import Flask

app = Flask(__name__)

@app.route("/")
def main():
	mysql.connector.connect(host='mysql', database='mysql', user='root', password='paswrd')

	return render_template('hello.html', color=fetchcolor())

if __name__ == "__main__":
	app.run(host="0.0.0.0", port="8080")
```

If you look closely into the code you will see the hostname, username and password hardcoded. **This is of course not a good idea.** As we learned in the previous lecture, one option would be to move these values into a configMap.

ConfigMap stores configuration data in plain text format. So while it would be okay to move the hostname and username into a configMap **it is definitely not the right place to store a password.**

This is where secrets coming. Secrets are used to store sensitive information like passwords or keys. They’re similar to configMap except that they’re stored in an encoded or hashed format. 

As with ConfigMaps, there are two steps involved in working with secrets. First create the secret and second injected into pod. 

### Imperative

```bash
kubectl create secret generic \
	app-sercret **--from-literal**=DB_Host=mysql \
							--from-literal=DB_User=root
```

```bash
kubectl create secret generic \
		app-secret **--from-file**=app_secret.properties
```

### Declarative

```yaml
apiVerion: v1
kind: Secret
metadata:
	name: app-secret
data:
	DB_Host: mysql
	DB_User: root
	DB_Password: paswrd
```

Here we have specified the data in plain text which is not very safe. So while creating a secret with declarative approach you must specify the secret values in a hashed format. So you must specify the data in an encoded form like below.

```yaml
apiVerion: v1
kind: Secret
metadata:
	name: app-secret
data:
	DB_Host: bXlzcWw=
	DB_User: cm9vdA==
	DB_Password: cGFzd3Jk
```

How do you convert the data from plain text to an encoded format? 

```bash
echo -n 'mysql' | base64
echo -n 'root' | base64
echo -n 'paswrd' | base64
```

How do you decode these hashed values? Add a decode option to it

```bash
echo -n 'bXlzcWw=' | base64 --decode
echo -n 'cm9vdA==' | base64 --decode
echo -n 'cGFzd3Jk' | base64 --decode
```

### Inject an environment variable

To inject an environment variable, add a new property to the container called envFrom. The envFrom property is a list so we can pass as many environment variables as required each item in the list corresponds to a secret item.

pod-definition.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
	...
spec:
	...
	envFrom:
		- secretRef:
				name: **app-secret**
```

secret-data.yaml

```yaml
apiVerion: v1
kind: Secret
metadata:
	name: **app-secret**
data:
	DB_Host: bXlzcWw=
	DB_User: cm9vdA==
	DB_Password: cGFzd3Jk
```

### Inject as single environment variables

```yaml
env:
	- name: DB_Password
		valueFrom:
			secretKeyRef:
				name: app-secret
				key: DB_Password
```

### Inject the whole secret as files in a volume

```yaml
volumes:
- name: app-secret-volume
	secret:
		secretName: app-secret
```

If you were to mount the secret as a volume in the pod. Each attribute in the secret is created as a file with the value of the secret as its content.

<aside>
⚠️ Remember that secrets encode data in base64 format. Anyone with the **base64 encoded secret can easily decode it**. As such the secrets can be considered as not very safe. **It's not the secret itself that is safe**

</aside>

[https://kubernetes.io/docs/concepts/configuration/secret/#risks](https://kubernetes.io/docs/concepts/configuration/secret/#risks)
