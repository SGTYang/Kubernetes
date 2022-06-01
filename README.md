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

## Taint - NoExecute
