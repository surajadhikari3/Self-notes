https://how.dev/answers/the-kubernetes-architecture-simplified
Study from this site 


The purpose of Kubernetes is to host your application, in the form of containers, in an automated fashion so that you can easily deploy, scale-in and scale-out (as required), and enable communications between different services inside your application.

Master(also called control plane) And Worker node(also refered to slave)

Pod lifecycle --> Pending --> Node is allocated, starting --> Img is pulled from cloud, running, crashLoopBackOff --> Some issue

init --> Loads the default image in the container in pods.

pods -->**Pods** are the smallest unit of deployment in Kubernetes just as a container is the smallest unit of deployment in Docker


![[Pasted image 20250511171919.png]]

![[Pasted image 20250511170644.png]]

![[Pasted image 20250511170654.png]]


Api server is the single entry through which all comunicate..

## Controller Manager:

`Controller Manager` is responsible to make sure the actual state of the system converges towards the desired state, as specified in the resource specification. There are several different controllers available under controller manager. Some of them are `DeploymentControllers`, `StatefulSet` Controllers, `Namespace` Controllers, `PersistentVolume` Controllers etc.

**Etcd** is a data store that stores the cluster configuration.


Kubernetes follows a **master-slave (control plane – node)** architecture.

#### 1. **Control Plane Components** (Master Node)

- **kube-apiserver**: Entry point for all REST commands; validates and processes API requests.
    
- **etcd**: Key-value store used for persistent storage of all cluster data.
    
- **kube-scheduler**: Assigns pods to available nodes based on resource availability.
    
- **kube-controller-manager**: Runs various controllers like replication, node, and endpoints.
    
- **cloud-controller-manager**: Manages cloud-specific controller logic (optional in bare metal).
    

#### 2. **Node Components**

- **kubelet**: Agent that ensures containers are running in a pod.
    
- **kube-proxy**: Maintains network rules and load balances traffic to pods.
    
- **Container Runtime**: Software responsible for running containers (e.g., containerd, Docker).
    

#### 3. **Additional Concepts**

- **Pods**: Smallest deployable unit; wraps containers.
    
- **ReplicaSets**: Ensures a specified number of pod replicas are running.
    
- **Deployments**: Declarative updates for pods and ReplicaSets.
    
- **Services**: Abstraction for a group of pods; ensures connectivity and load balancing.
    
- **Namespaces**: Virtual clusters for multi-tenant environments.
    

---

### 📋 Kubernetes Interview Questions and Answers

#### 1. **Q: What is Kubernetes?**

**A:** Kubernetes is an open-source container orchestration platform for automating deployment, scaling, and management of containerized applications.

#### 2. **Q: What are the main components of the Kubernetes control plane?**

**A:** `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, and optionally `cloud-controller-manager`.

#### 3. **Q: What is the role of `etcd` in Kubernetes?**

**A:** `etcd` is a distributed key-value store that holds the state of the cluster and configuration data.

#### 4. **Q: What is a Pod in Kubernetes?**

**A:** A Pod is the smallest computing unit that  can be deployed into kubernetes which contain one or more containers. 
Note: But in the real world we write the deployment object in the yaml file which handle the replicaset which handles the pod.. Deployment provides the ability for updates, rollback of the application as it manges the version and there is control loop which keeps on looking the changes and act on it..
#### 5. **Q: How does Kubernetes handle networking?**

**A:** Every pod gets its own IP address. `kube-proxy` manages network rules and handles communication between pods and services.

#### 6. **Q: What is a ReplicaSet?**

**A:** A ReplicaSet ensures a specified number of pod replicas are running at any given time. Even if a pod crashes , ReplicaSet replaces it with another pod.

#### 7. **Q: What is the difference between a ReplicaSet and a Deployment?**

**A:** A Deployment manages ReplicaSets and provides declarative updates, rollback, and versioning.

![[Pasted image 20250512103011.png]]

If you want to. manually manage the pods then use the ReplicaSet 
Use deployment if you want upgrades, rollback and easier lifecycle management of pods

Tabular form

|Feature|**ReplicaSet**|**Deployment**|
|---|---|---|
|**Purpose**|Ensures a **fixed number of pods** are running|Manages **ReplicaSets** and handles **rolling updates**|
|**Manages Pods?**|Yes|Yes, **indirectly** via ReplicaSets|
|**Rolling Updates?**|❌ Not supported|✅ Yes, supports zero-downtime updates|
|**Rollback Support?**|❌ No|✅ Yes, can roll back to previous versions|
|**Typical Use**|Rarely used directly by users|Commonly used in real-world deployments|
|**Version History?**|❌ No history maintained|✅ Keeps history of previous ReplicaSets|
|**Scaling**|Manual or via HPA (Horizontal Pod Autoscaler)|Easier, supports declarative scaling with history|

---

#### 8. **Q: How does Kubernetes ensure high availability?**

**A:** It supports multi-master setups,   `etcd` data (for storing the state), auto-scaling via replicaset, Persistent storage , health checks, and restarts failed pods automatically using controllers.

#### 9. **Q: What are DaemonSets?**

**A:** DaemonSets ensure that a pod runs on all (or some specific) nodes in the cluster, commonly used for logging and monitoring agents.

### When to Use DaemonSets:

| Use Case                                              | Why Use DaemonSet?                           |
| ----------------------------------------------------- | -------------------------------------------- |
| **Log collection agents** (e.g. Fluentd)              | To collect logs from each node               |
| **Monitoring agents** (e.g. Prometheus Node Exporter) | For collecting node-level metrics            |
| **Security agents** (e.g. antivirus or audit tools)   | To inspect traffic or processes on each node |
| **Storage plugins**                                   | For mounting volumes on each node            |
| **Custom node-specific services**                     | Like network proxies, service meshes, etc.   |
![[Pasted image 20250512112021.png]]
#### 10. **Q: What are StatefulSets used for?**(Workload controller)

**A:** They manage stateful applications(like database), maintaining sticky(persistent) identities and ordered deployment, scaling, and termination. 

#### 11. **Q: How do Services work in Kubernetes?**

**A:** Service is an abstraction that  provide a stable IP and DNS name to access a group of pods and can load balance across them.

Since the pod is ephemeral(short lived) there ip gets changed too . In the service there is label field by which the pods gots grouped. Lets say all frontend pods can have the label as frontend defined in the yaml file.

![[Pasted image 20250512115729.png]]

Types of services in kubernetes

| Type                    | Description                                                                                                                              |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **ClusterIP** (default) | Exposes the service **internally** within the cluster. Not accessible from outside.                                                      |
| **NodePort**            | Exposes the service on **each Node's IP at a static port** (e.g., 30080). Accessible from outside the cluster via `<NodeIP>:<NodePort>`. |
| **LoadBalancer**        | Provisions an **external load balancer** (if running on cloud providers like AWS, GCP, Azure).                                           |
|                         |                                                                                                                                          |

#### 12. **Q: What is a Namespace in Kubernetes?**

**A:** Namespaces allow logical separation of resources within a K8s cluster for better organization and multi-tenancy.  --> virtual cluster within the k8s cluster.. If the namespace is deleted all the resources that it holds will also be deleted which is managed by the namespace controller.

#### 13. **Q: What is the role of the kubelet?**

**A:** `kubelet` ensures that containers described in the PodSpecs are running and healthy on the node. 
kubelet is the primary node agent in k8s. It run in the every node in the cluster and is responsible for managing the lifecycle of pods on that node 

#### 14. **Q: How does Kubernetes manage secrets and configurations?**

**A:** Using ConfigMaps (for configurations) and Secrets (for sensitive data like passwords).

#### 15. **Q: What is Helm in Kubernetes?**

**A:** Helm is a package manager for Kubernetes that helps in managing Kubernetes applications through Helm Charts.(Like apt for ubuntu)


Persistence volume

![[Pasted image 20250512101437.png]]


Persistent volume (PV)  in kubernetes is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using the storage classes.

It is independent of the pod.

Persistent volume claim(PVC) --> Request for storage by user(Developer/ Devops)


# Why pod is needed for the containers. (Pod abstraction)

For handling the networking as the container does need the port but if we have to map the port and if the containers is high it is harder to mange the port as it is. cumbersome to keep which port is free and which is not..

So pods have there own unique ip address. that means pods is running as a separate space(isolated virtual host with its own network namespace) having its own ip and ports . So we can have the 10 or more postgres app running in the same port 8080 as they are isolated by the pods..

# Controllers in Kubernetes#

controllers are the control loop that continuously watch the state of cluster and make changes to move the current state toward the desired state.

🔁 **Workload Controllers**

Manage **Pods** and ensure the desired number of replicas are running.

| Controller      | Description                                                                                                      |
| --------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Deployment**  | Ensures a specified number of Pods are running, supports rolling updates and rollbacks.                          |
| **ReplicaSet**  | Ensures a specific number of identical Pods are running. Mostly used behind Deployments.                         |
| **StatefulSet** | Manages Pods with **persistent identity** and **stable storage**, used for stateful applications like databases. |
| **DaemonSet**   | Ensures a Pod runs on **every node** (or a subset), useful for logs, monitoring, etc.                            |
| **Job**         | Creates one or more Pods and ensures they **complete successfully**. Used for batch tasks.                       |
| **CronJob**     | Runs Jobs on a **scheduled basis**, like a Linux cron task.                                                      |

---

### 🧠 **Control Plane Controllers** (Run inside the `kube-controller-manager`)


| Controller                             | Description                                                                                                                                                                                                                                                              |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Node Controller**                    | Monitors node health and updates the cluster accordingly (e.g., marking nodes as `NotReady`).                                                                                                                                                                            |
| **Replication Controller**             | Predecessor to ReplicaSet. Maintains specified number of Pod replicas. (Deprecated in favor of ReplicaSet)                                                                                                                                                               |
| **Endpoints Controller**               | Populates endpoint objects (IP/port info) for Services.                                                                                                                                                                                                                  |
| **Namespace Controller**               | Handles lifecycle of namespaces. Deletes all resources in a namespace when it is deleted. <br><br>(Name space is like logical partition or virtual cluster within physical kubernets cluster) Think like it is space where the deployments, pods, services, secrets..... |
| **Service Account & Token Controller** | Creates default service accounts and API tokens for new namespaces.                                                                                                                                                                                                      |
| **ResourceQuota Controller**           | Ensures limits and quotas are enforced on namespaces.                                                                                                                                                                                                                    |
### 📦 **Storage Controllers**

|Controller|Description|
|---|---|
|**PersistentVolume Controller**|Watches and manages lifecycle of PersistentVolumes.|
|**PVC Binder Controller**|Binds PersistentVolumeClaims (PVCs) to available PersistentVolumes (PVs).|
|**Volume Attachment Controller**|Handles volume attach/detach operations for nodes.|


Best practises:

It is recommended to run the one container per pods. If you have dependency or have some other cross cutting concerns like logging then you can run the multiple containers per pods which is the sideCar... 