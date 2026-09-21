# Kubernetes Comprehensive Guide: From Clusters to Storage Classes

## 1. What is a Cluster in Kubernetes?

A **Kubernetes cluster** is a set of connected computing machines (physical or virtual) that work together as a single unit to run, manage, and scale containerized applications. Instead of deploying your software onto one specific computer, you deploy it to the cluster, and Kubernetes automatically decides which machine has the space and resources to run it.

Every Kubernetes cluster is split into two major parts:

### The Control Plane (The "Brain")
The **Control Plane** manages the overall state of the cluster, makes global decisions (like scheduling applications), and detects and responds to cluster events. It includes components like:
* **API Server (`kube-apiserver`):** The front door to the cluster that accepts commands.
* **etcd:** A secure, key-value database that stores all cluster configuration and state data.
* **Scheduler (`kube-scheduler`):** Looks at new application requests and chooses the best machine to run them on.
* **Controller Manager (`kube-controller-manager`):** Watches the cluster to make sure the actual state matches your requested state (e.g., restarting a crashed application).

### Worker Nodes (The "Muscle")
**Worker Nodes** are the actual machines (servers or virtual machines) where your applications run. Each worker node contains:
* **Pods:** The smallest deployable units in Kubernetes, which wrap around your application containers.
* **Kubelet:** A tiny agent that communicates with the control plane and ensures the containers are running properly inside their pods.
* **Kube-proxy:** A network agent that handles communication between your applications, inside and outside the cluster.
* **Container Runtime:** The software engine (like Docker or containerd) that actually runs the containers.

### Why use a cluster?
* **Self-healing:** If a container or a physical server crashes, the cluster automatically spins up a replacement on a healthy machine to prevent downtime.
* **Scalability:** You can easily scale your applications up or down by telling the cluster to run more copies of your application across the available machines.
* **Infrastructure Abstraction:** Developers don't need to care which specific server their application is sitting on; they just hand it to the cluster.

---

## 2. API Server

The **API Server (`kube-apiserver`)** is the central management hub and the single entry point for everything that happens inside a Kubernetes cluster. 

Think of it as the **cluster's primary gateway** or receptionist. Whether an action is taken by a human administrator using a tool like `kubectl`, an automated script, or an internal Kubernetes component, it *must* go through the API Server.

### Core Responsibilities
The API Server handles four critical functions for every single request:
* **Authentication:** It verifies **who** is making the request (e.g., checking if the user token or SSL certificate is valid).
* **Authorization:** It checks **what** the authenticated user is allowed to do based on defined permissions (e.g., Role-Based Access Control, or RBAC).
* **Validation & Mutation:** It inspects the request to ensure the formatting is correct (validation) and modifies it if necessary (mutation) before accepting it.
* **Database Synchronization:** It is the **only** component in the entire cluster allowed to talk directly to `etcd` (the cluster's database). It writes the new state or configuration to `etcd` once a request passes all checks.

### How it Communicates
The API Server is **stateless**, meaning it doesn't store data itself—it scales horizontally by simply spinning up more instances. It communicates using a **RESTful API** over HTTP(S), exposing JSON or Protocol Buffers endpoints. 

Whenever you run a command like `kubectl get pods`, `kubectl` sends an HTTP request to the API Server, which reads the current pod data from `etcd` and passes it back to your terminal.

---

## 3. Scheduler

The **Kubernetes Scheduler (`kube-scheduler`)** is the control plane's ultimate matchmaker. Its sole responsibility is to watch for newly created Pods that don't have a node assigned to them yet and **decide which physical or virtual worker node is the best fit for that Pod**.

Crucially, **the scheduler only makes the decision; it does not actually run the Pod**. Once the scheduler chooses a node, it writes that decision back to the API Server, and the `Kubelet` on the destination worker node takes over to spin up the container.

### The Filter, Score, Bind Pipeline
To find the perfect node for a "Pending" Pod, the scheduler executes a rapid, precise three-phase workflow:

#### Filtering (Predicates)
The scheduler evaluates all available worker nodes and eliminates the ones that **cannot physically or logically host the Pod**. It checks criteria like:
* **Resource Sufficiency:** Does the node have enough unallocated CPU and RAM to satisfy what the Pod requested?
* **Node Selectors & Affinity:** Does the Pod request a specific type of machine (e.g., a server with an attached GPU)?
* **Taints and Tolerations:** Is the node configured to repel certain types of workloads?

The surviving nodes that pass this phase are called **feasible nodes**.

#### Scoring (Priorities)
Once it has a list of feasible nodes, the scheduler ranks them to find the absolute best option. It applies a series of priority functions to score each node on a scale from 0 to 10. Factors include:
* **Bin Packing / Resource Balancing:** Spreading workloads out evenly or packing them tightly to save costs.
* **Image Locality:** Scoring a node higher if it has already downloaded the required container image, which helps the Pod start faster.

The node with the highest cumulative score wins. If there is a tie, it chooses one at random.

#### Binding
This is the final hand-off. The scheduler contacts the API Server and updates the Pod’s specifications, assigning the winning node's name to the `nodeName` field. The scheduler's job for that Pod is now complete.

### Key Architectural Behaviors
* **One-Shot Contract:** The scheduler is **intentionally narrow and one-shot**. It makes a decision once when the Pod is created. If the cluster's actual resource usage changes drastically later, the default scheduler *will not* move or re-balance the running Pods.
* **Highly Customizable:** Kubernetes allows you to write custom scheduling plugins or even run **multiple different schedulers** simultaneously inside the same cluster.

---

## 4. Controller Manager

The **Kubernetes Controller Manager (`kube-controller-manager`)** is the cluster's continuous automation engine and "autopilot". Its primary responsibility is to maintain the **desired state** of your cluster. 

In Kubernetes, you define how you want things to look (e.g., *"I want exactly 3 replicas of my web app running at all times"*), and the Controller Manager works 24/7 to make sure reality matches that definition.

### The Control Loop (How It Works)
The Controller Manager runs a continuous, non-terminating **control loop**. The logic follows a simple, infinite three-step cycle:
1. **Watch:** Check the current actual state of the cluster by querying the API Server.
2. **Compare:** Evaluate the actual state against the desired state stored in configuration.
3. **Act:** If there is a difference, execute commands to fix it and drive the cluster back to the desired state.

### One Process, Many Controllers
Architecturally, the `kube-controller-manager` is a single binary process, but inside it, it runs **dozens of independent, specialized sub-controllers** via lightweight Go routines. Each sub-controller looks after one specific resource type:
* **Deployment / ReplicaSet Controller:** Ensures the exact number of Pod replicas you requested are actively running. If a Pod crashes, this controller notices the count dropped and asks the API Server to create a new one.
* **Node Controller:** Monitors the health of worker nodes. If a machine stops sending "heartbeats" (e.g., due to hardware failure), this controller marks it as unreachable and handles moving your workloads to healthy nodes.
* **Namespace Controller:** Manages the lifecycle of namespaces, cleanly deleting all objects inside a namespace when that namespace is deleted.
* **EndpointSlice / Endpoints Controller:** Links your backend Pods to your Services so network traffic safely reaches active containers.

### Cloud Environments
If you run Kubernetes on a public cloud (like AWS, Azure, or Google Cloud), a companion component called the **`cloud-controller-manager`** separates cloud-specific automation—like provisioning cloud load balancers or managing cloud storage attachments—from standard, on-premise cluster logic.

---

## 5. etcd

**`etcd`** is the cluster’s ultimate source of truth. It is a highly available, distributed, **key-value store** that serves as Kubernetes' database. 

Every single piece of data about your cluster—including the state of every node, pod, configuration, secrets, and deployment—is saved inside `etcd`. If the API Server is the brain, `etcd` is the **long-term memory**.

### Core Characteristics & Architecture
* **Strong Consistency (Raft Protocol):** `etcd` prioritizes data accuracy over speed. It uses the **Raft consensus algorithm** to ensure that all copies of the database across multiple control plane nodes agree on the exact same data at the exact same millisecond. 
* **Key-Value Format:** Unlike a traditional relational database (like MySQL) with tables and rows, `etcd` stores data hierarchically in directories and keys, much like a file system (e.g., `/registry/pods/default/my-pod`).
* **Watch Mechanism:** It features a highly efficient "Watch" API. Instead of other Kubernetes components constantly polling the database for updates, they can "subscribe" to certain keys. `etcd` will instantly stream an event notification to them the exact moment a value changes.
* **Strict Access Security:** No component in the cluster can talk to `etcd` directly except for the **API Server**. If a worker node or a developer wants to update something, they must ask the API Server, which validates the request and writes it to `etcd`.

### Why odd-numbered clusters matter
Because `etcd` is a distributed system, it requires a **quorum** (a strict majority) to accept any database writes. If a network split occurs, the database needs to know which side has the true majority to prevent data corruption ("split-brain" scenario). 

For this reason, production control planes deploy `etcd` in **odd numbers**—typically 3, 5, or 7 nodes:
* A **3-node** cluster can survive **1** node failing (Majority = 2).
* A **5-node** cluster can survive **2** nodes failing (Majority = 3).

### What happens if etcd goes down?
If `etcd` crashes completely, the cluster effectively becomes **read-only and frozen**. Your existing applications running on worker nodes will continue to run and handle user traffic, but you cannot create new pods, scale deployments, delete resources, or recover from node crashes because Kubernetes cannot save its new state anywhere.

---

## 6. Kubelet

The **`kubelet`** is the primary **"node agent"** that runs on every single worker node in a Kubernetes cluster. 

If the Control Plane is the brain of the cluster, the `kubelet` is the **on-site construction foreman** on the ground. It is responsible for making sure that the containers assigned to its specific machine are actually running and healthy.

### Core Responsibilities
The `kubelet` does not manage the whole cluster; it only cares about the specific machine it is installed on. It performs four essential duties:
* **Pod Lifecycle Management:** It constantly watches the API Server for new **PodSpecs** (YAML or JSON files that describe a Pod) assigned to its node. Once assigned, it tells the container runtime to pull the images and spin up the containers.
* **Health Monitoring (Probes):** It continuously executes the Liveness, Readiness, and Startup probes defined in your configurations. If a container fails its health check, the `kubelet` will kill and restart it.
* **Status Reporting:** It reports back to the Control Plane API Server at regular intervals, providing updates on the node's resource usage (CPU/RAM) and the current status of all its pods.
* **Volume Mounting:** It handles configuring and mounting the local or network storage directories (volumes) that your containers need to access.

### How the Kubelet Works: The Declared State
The `kubelet` works entirely on a **declarative model**. It operates in a loop, ensuring that the actual state of the containers on the machine matches the desired state described in the PodSpec.
1. **Receive:** It receives a PodSpec from the API Server (or a local file).
2. **Execute:** It translates that spec into commands for the **Container Runtime** (like `containerd` or Docker) via the **Container Runtime Interface (CRI)**.
3. **Monitor:** It doesn't run the containers itself; it watches the runtime to ensure they stay up. If a container dies, the `kubelet` immediately kicks off a replacement loop.

### Key Architectural Traits
* **Runs as a Native OS Service:** Unlike most Kubernetes components which run inside containers, the `kubelet` runs directly on the node's host operating system as a native daemon/system service (e.g., via `systemd`). This ensures it stays running even if the container runtime crashes.
* **The `cAdvisor` Integration:** The `kubelet` includes an embedded software agent called `cAdvisor` (Container Advisor). This agent collects, aggregates, and analyzes resource usage data from all running containers on the node, which is then used by the metrics server and the horizontal pod autoscaler.

---

## 7. What is a Deployment in Kubernetes?

A **Kubernetes Deployment** is a high-level resource object that automates the lifecycle of your application containers. It acts as a **manager for your Pods**, allowing you to describe your *desired state* in a configuration file (YAML), while Kubernetes works in the background to make reality match that definition.

Instead of creating individual Pods manually—which won't restart if they fail—you use a Deployment to ensure your application is scalable, self-healing, and easy to update.

### The 3-Tier Hierarchy
A Deployment does not actually manage Pods directly. It orchestrates them using a layer of abstractions:
1. **Deployment:** The top-level manager where you define your application version, update strategies, and total replica counts.
2. **ReplicaSet:** Created automatically by the Deployment. Its sole job is to maintain the exact number of stable Pod clones you requested. 
3. **Pods:** The actual running containers housing your application code.

```
  [ Deployment ]
        │
        ▼
  [ ReplicaSet ]
   /    │      ▼     ▼     ▼
[Pod] [Pod] [Pod]
```

### Key Capabilities of a Deployment
* **Zero-Downtime Rolling Updates:** When you update your application's code (e.g., changing from version `v1` to `v2`), the Deployment performs a **Rolling Update**. It gracefully spins up a new `v2` Pod, waits for it to be healthy, and then terminates a `v1` Pod—repeating this cycle until the entire fleet is updated without taking your app offline.
* **Automated Self-Healing:** If a worker node crashes or a Pod unexpectedly fails, the Deployment Controller immediately notices the actual count dropped below your desired count and schedules replacement Pods on healthy infrastructure.
* **Seamless Scaling:** Need to handle a massive spike in traffic? You can scale your app out instantly by telling the Deployment to change its `replicas` from 3 to 30. 
* **Instant Rollbacks:** If you push a bad software update that starts crashing or throwing errors, you can run a single command (`kubectl rollout undo`) to instantly revert the cluster back to the previous stable revision.

### What it looks like (Minimal YAML Example)
This blueprint tells Kubernetes to keep exactly 3 copies of an Nginx web server running:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-deployment
spec:
  replicas: 3  # Desired state: always keep 3 copies running
  selector:
    matchLabels:
      app: web-server
  template:    # The blueprint for the Pods themselves
    metadata:
      labels:
        app: web-server
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.25.0
        ports:
        - containerPort: 80
```

---

## 8. Deployment Performs a Rolling Update

When a Deployment triggers a **Rolling Update** (typically because you changed the container image version in the YAML file), it shifts traffic from your old version (`v1`) to your new version (`v2`) gradually. 

To achieve this without downtime, the Deployment creates a **brand-new ReplicaSet** for `v2` and systematically scales it up while scaling the old `v1` ReplicaSet down.

### The Rolling Update Algorithm
The entire process is governed by a precise, safe dance controlled by two key settings in your Deployment configuration: **`maxSurge`** and **`maxUnavailable`**. By default, both are set to **25%**, which dictates the exact steps Kubernetes takes:

```
[Old ReplicaSet: 4 Pods]                [New ReplicaSet: 0 Pods]
    ↓↓↓↓ (Start Update)
[Old: 4 Pods]                           [New: +1 Pod creating] (MaxSurge allows 5 total pods)
[Old: 4 Pods]                           [New: 1 Pod Healthy]   
[Old: -1 Pod terminated]                [New: 1 Pod Healthy]   (Scale down old)
[Old: 3 Pods]                           [New: +1 Pod creating] ...
    └─► This cycle repeats until all 4 Pods are running on the New ReplicaSet.
```

1. **Create the New Fleet:** The Deployment asks the new `v2` ReplicaSet to spin up a small batch of new Pods.
2. **Wait for Health Checks:** The new Pods are *not* given traffic immediately. The `kubelet` evaluates their **Readiness Probes**. Only when a new Pod is confirmed healthy does it get added to the network load balancer.
3. **Kill the Old Fleet:** Once a new `v2` Pod is up and running, the Deployment instructs the old `v1` ReplicaSet to terminate one of the old Pods.
4. **Repeat:** This loop continues until 100% of the active Pods belong to the new `v2` ReplicaSet. The old ReplicaSet is not deleted; it is scaled down to `0` replicas so it can be used for an instant rollback if needed.

### Controlling the Flow: Surge & Unavailability

| Setting | What it means | Default Value |
| :--- | :--- | :--- |
| **`maxSurge`** | How many extra Pods can be created *above* your desired replica count during the update. | **25%** (rounded up) |
| **`maxUnavailable`** | How many Pods can be taken offline *below* your desired replica count during the update. | **25%** (rounded down) |

#### Scenario: Tuning for Cost vs. Speed
* **Maximum Safety (Zero impact on capacity):** Set `maxUnavailable: 0` and `maxSurge: 1`. Kubernetes will never kill an old Pod until a new one is 100% ready. 
* **Fastest / Cloud Native:** Set `maxSurge: 100%` and `maxUnavailable: 0`. Kubernetes will instantly double the infrastructure size, spin up all new Pods at once, wait for them to pass health checks, and then tear down the old environment completely.

### Crucial `kubectl` Commands for Rolling Updates
* **Trigger an update (via command line):**
  ```bash
  kubectl set image deployment/web-app-deployment nginx-container=nginx:1.26.0
  ```
* **Watch the update happen in real-time:**
  ```bash
  kubectl rollout status deployment/web-app-deployment
  ```
* **Pause an update midway (if you suspect an issue):**
  ```bash
  kubectl rollout pause deployment/web-app-deployment
  ```
* **Abort and instantly revert back to the previous version:**
  ```bash
  kubectl rollout undo deployment/web-app-deployment
  ```

---

## 9. Rollback

A **Deployment Rollback** is your safety net in Kubernetes. If you roll out an update (like changing from application version `v1` to `v2`) and the new code starts crashing, throwing errors, or running out of memory, a rollback allows you to instantly revert your cluster back to a previous, known stable state.

Kubernetes makes this incredibly fast and safe because it **never deletes your old ReplicaSets**—it merely scales them down to `0` replicas during an update. Rolling back simply reverses the process, scaling the old ReplicaSet back up and the bad one down.

### Essential Rollback Commands

#### Check the Deployment History
Before rolling back, you want to see your available revisions:
```bash
kubectl rollout history deployment/web-app-deployment
```
*Output looks like this:*
```text
REVISION  CHANGE-CAUSE
1         <none>
2         Updated image to v2.0.0
3         Updated image to broken-v2.1.0  <-- Current active bad version
```

#### Undo the Last Update (Instant Fix)
If you just want to go back exactly **one step** to the previous working version, run:
```bash
kubectl rollout undo deployment/web-app-deployment
```
This takes the immediately preceding revision (Revision 2) and brings it back to life.

#### Roll Back to a Specific Historical Revision
If you want to jump back to a much older, specific revision (e.g., Revision 1), target it directly:
```bash
kubectl rollout undo deployment/web-app-deployment --to-revision=1
```

### Best Practice: Tracking the `CHANGE-CAUSE`
By default, the `CHANGE-CAUSE` column in your history will often show `<none>`. To document your history automatically, you can add an annotation to your configuration or append parameters when updating images:
```bash
kubectl annotate deployment/web-app-deployment kubernetes.io/change-cause="Rolled back to stable production release v1"
```

### How it Handles Traffic Mid-Rollback
Just like a rolling update, a rollback is graceful. It doesn't instantly kill all your bad `v2` pods and leave your users with a blank screen. It uses the exact same `maxSurge` and `maxUnavailable` pacing rules to safely swap the pods out while keeping the application fully available.

---

## 10. Service

In Kubernetes, a **Service** is an abstract way to expose an application running on a set of Pods as a network service. 

Because Kubernetes Pods are mortal—they are constantly created, destroyed, and given **dynamic, unpredictable IP addresses** during deployments and scaling—you cannot rely on their direct IPs. A Service provides a **single, permanent IP address and DNS name** that acts as a stable load balancer in front of your shifting Pods.

### How a Service Finds Your Pods (Selectors)
Services do not track Pods by hardcoded lists. Instead, they use **Labels and Selectors**. If a Service’s selector is set to `app: web-server`, it will instantly and automatically route traffic to any Pod in the cluster that has the label `app: web-server`, no matter what its IP address is or what node it is sitting on.

### The 4 Core Service Types

#### ClusterIP (Default)
Exposes the Service on an **internal cluster-only IP**. 
* **Who can access it:** Only other Pods inside the exact same Kubernetes cluster.
* **Best use case:** Backend databases, caching layers, or internal microservices that should never be exposed to the public internet.

#### NodePort
Exposes the Service on a **specific, static port on every Worker Node’s IP address**.
* **Who can access it:** Anyone who can access the IP address of any of your physical or virtual worker machines. It opens a port (typically in the `30000-32767` range) across all nodes.
* **Best use case:** Quick testing or on-premise development where you don't have an automated cloud load balancer.

#### LoadBalancer
Exposes the Service externally using a **cloud provider's native load balancer**.
* **Who can access it:** The public internet. When you deploy this on AWS, Azure, or Google Cloud, Kubernetes automatically provisions an external cloud load balancer (like an AWS ALB) and maps a public IP to your Service.
* **Best use case:** Public-facing websites, APIs, and user entry points.

#### ExternalName
Maps a Kubernetes Service to a **traditional, external DNS name** (like `database.company.com`) by returning a `CNAME` record.
* **Who can access it:** Pods looking to communicate with legacy systems outside the cluster.
* **Best use case:** Pointing your internal pods to an external database without hardcoding endpoints into your application code.

### What a Service Looks Like (YAML Blueprint)
This Service creates a stable internal entry point (`ClusterIP`) that routes incoming traffic on port `80` to any backend Pods labeled `app: web-server` on their container port `8080`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # Accessible only inside the cluster
  selector:
    app: web-server  # Finds pods with this label
  ports:
  - protocol: TCP
    port: 80         # The port the Service listens on
    targetPort: 8080 # The port your actual container is running on
```

### The Under-the-Hood Network Engine
While a Service feels like a physical router, it is actually a logical abstraction. The **`kube-proxy`** agent running on every worker node modifies the server's network routing tables (using `iptables` or `IPVS`) so that network traffic destined for the Service IP is transparently intercepted and forwarded directly to a healthy Pod.

---

## 11. Do Services Also Track Health of Pods?

**Yes, a Kubernetes Service tracks the health of its Pods, but it does not evaluate their health directly.** Instead, the Service relies on an assistant called the **Endpoints Controller** (or EndpointSlice Controller) and the **`kubelet`** to keep the target list up to date.

If a Pod fails a health check, it is temporarily disconnected from the Service so your users never hit a broken container.

### How the Health-Tracking Engine Works
1. **The Kubelet Checks:** The `kubelet` agent running on the worker node continuously runs **Readiness Probes** against your containers (e.g., hitting a `/healthz` HTTP endpoint or checking if a TCP port is open).
2. **The Status Update:** If the container fails this check, the `kubelet` reports it to the API Server, changing the Pod's status from `Ready: True` to `Ready: False`.
3. **The Endpoints Controller Cleans Up:** The Endpoints Controller continuously watches the API Server. The moment it sees a Pod's status switch to *Not Ready*, it immediately **removes that Pod's IP address** from the Service's active routing list (called the Endpoints list).
4. **Traffic stops:** Because the IP is gone from the list, the Service instantly stops sending network traffic to that specific Pod.

### What it looks like in practice
You can actually see this tracking list live using `kubectl`. If you have a deployment with 3 replicas, running `kubectl get endpoints` will show the 3 healthy Pod IPs:
```text
NAME               ENDPOINTS
backend-service    192.168.1.5:8080,192.168.2.10:8080,192.168.2.11:8080
```

If the Pod on `192.168.1.5` fails its readiness probe, the list immediately updates to hide the broken Pod:
```text
NAME               ENDPOINTS
backend-service    192.168.2.10:8080,192.168.2.11:8080
```

---

## 12. ConfigMaps

A **Kubernetes ConfigMap** is an API object used to store non-confidential configuration data in key-value pairs. 

ConfigMaps allow you to cleanly **decouple your application code from your configuration blueprints**. Instead of hardcoding environment variables, database URLs, or feature flags inside your Docker images, you store them in a ConfigMap. This means you can deploy the exact same container image across Development, Staging, and Production environments, simply by swapping out the ConfigMap attached to it.

### 3 Ways to Inject ConfigMaps into a Pod
1. **Individual Environment Variables:** You can pull specific keys out of a ConfigMap and map them to standard environment variables inside the container.
2. **Mass Environment Variables (`envFrom`):** You can dump the entire contents of a ConfigMap into a container as environment variables all at once.
3. **Config Files Mounted as Volumes:** Kubernetes can take the key-value pairs in a ConfigMap, turn them into physical files, and **mount them inside a directory** in your container.

### What it looks like (YAML Example)

#### Step 1: Create the ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  DB_URL: "jdbc:mysql://db.example.com:3306/mydb"
  ENABLE_FEATURES: "true"
```

#### Step 2: Inject it into a Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: main-container
        image: my-app:v1.0
        env:
        - name: DATABASE_ENDPOINT
          valueFrom:
            configMapKeyRef:
              name: app-settings
              key: DB_URL
```

### Crucial Caveats & Best Practices
* **No Sensitive Data:** ConfigMaps do **not** provide encryption. The values are stored as plain text. For passwords or keys, use **Kubernetes Secrets** instead.
* **Hot-Reloading Behavior:** 
  * If injected via **Environment Variables**, the running containers *will not* see the changes until the Pods are restarted.
  * If mounted as a **Volume**, Kubernetes will automatically sync and update the files inside the running container after a brief delay.

---

## 13. Secrets

A **Kubernetes Secret** is an object designed specifically to store and manage sensitive, confidential information. By using Secrets, you avoid accidentally checking raw credentials into git repositories or embedding them directly into your container code.

### Essential Difference: Encoding vs. Encryption
* **Base64 Encoding (By Default):** When you write a Secret manifest, Kubernetes requires you to encode the data strings into **Base64** format (e.g., `password123` becomes `cGFzc3dvcmQxMjM=`). **Base64 is not encryption.** It is a simple text obfuscation formatting method. Anyone with access to the YAML can decode it instantly.
* **Encryption at Rest:** To truly secure secrets, you must enable **Encryption at Rest** inside your cluster's `etcd` control plane configuration.

### What it looks like (YAML Blueprint)

#### Step 1: Create the Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
data:
  db-username: ZGVidXBnZXI=      # "debugger" in base64
  db-password: c3VwZXItc2VjcmV0LXBhc3M= # "super-secret-pass" in base64
```

#### Step 2: Injecting it Safely into a Pod
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: app-container
        image: secure-image:v1
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: db-password
```

---

## 14. Horizontal and Vertical Scaling

In Kubernetes, scaling refers to adjusting your cluster's capacity to handle changes in application traffic and workload demands. You can scale your infrastructure in two entirely different directions: **Horizontal Scaling (Scaling Out/In)** and **Vertical Scaling (Scaling Up/Down)**.

### Direct Comparison

| Feature | Horizontal Scaling (HPA) | Vertical Scaling (VPA) |
| :--- | :--- | :--- |
| **Core Action** | **Adds or removes Pod copies** (replicas). | **Increases or decreases CPU/RAM size** of running Pods. |
| **Kubernetes Component** | Horizontal Pod Autoscaler (**HPA**) | Vertical Pod Autoscaler (**VPA**) |
| **Cost Efficiency** | High (Highly dynamic). | Moderate (Limited by underlying machine). |
| **Complexity** | High (Requires stateless apps). | Low (App stays exactly the same). |
| **Downtime Impact** | **Zero downtime**. Pods are cleanly added/removed. | **Requires a restart** by default to apply new specs. |

### Horizontal Pod Autoscaler (HPA)
Horizontal scaling is the textbook cloud-native approach. If the average CPU utilization across all pods exceeds your target threshold (e.g., 70%), it instructs the deployment to spin up new pod clones.
* **Prerequisite:** Your application **must be stateless**.

### Vertical Pod Autoscaler (VPA)
Vertical scaling is ideal for legacy workloads, stateful databases, or monolithic applications that cannot be easily broken down into cloned replicas.
* **The Catch:** Because Linux containers cannot change their core cgroup memory limits dynamically on the fly without a process interrupt, the VPA must **evict and restart the Pod** to apply the upgraded hardware sizes.

### Critical Warning: Never Mix HPA and VPA on CPU/Memory
You should **never configure both the HPA and VPA to track CPU or Memory usage on the same deployment**. Because they act independently, they will trigger a destructive loop where they continuously scale in opposing directions, resulting in complete cluster instability.

---

## 15. HPA and VPA Manifests

### Horizontal Pod Autoscaler (HPA) Manifest
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Vertical Pod Autoscaler (VPA) Manifest
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: database-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-database
  updatePolicy:
    updateMode: "Auto"     # Options: Auto, Recreate, Initial, Off
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "2"
        memory: "4Gi"
```

#### Understanding `updateMode` options:
* **`Auto` / `Recreate`:** The VPA will actively evict and restart the Pod mid-flight to apply larger resource limits.
* **`Off` (Recommendation Mode):** The VPA only calculates and suggests what the ideal resources should be without restarting your pod.

---

## 16. Metrics Server

The **Kubernetes Metrics Server** is a highly efficient, lightweight, cluster-wide aggregator of resource usage data. It serves as the core database engine that enables autoscaling. Without the Metrics Server running in your cluster, **neither HPA nor VPA can function**.

### How the Metrics Pipeline Works
1. **Local Collection:** The `kubelet` tracks container resource usage via `cAdvisor`.
2. **The Aggregation Scrape:** The Metrics Server pulls these raw statistics directly from every node’s `kubelet` API every **15 seconds**.
3. **API Presentation:** It formats this data and securely exposes it via the `metrics.k8s.io` stable API.

### Practical Diagnostic Commands
* **See the actual CPU and Memory usage of all Worker Nodes:**
  ```bash
  kubectl top node
  ```
* **See the resource consumption of all Pods:**
  ```bash
  kubectl top pod
  ```

### What the Metrics Server is NOT
It is **strictly a short-term buffer** with **no historical storage** and **no non-autoscaling metrics**. For custom graphing dashboards, system alerts, or historic data analysis, you must supplement your cluster with an observability stack like **Prometheus and Grafana**.

---

## 17. Resource Requests and Limits

In Kubernetes, **Resource Requests and Limits** are the mechanism you use to control how much **CPU** and **Memory (RAM)** your application containers are allowed to consume on worker nodes. 

### The Critical Distinction
* **Requests** are what you are **guaranteed** to get (your minimum reservation). Used by the **Scheduler** to place the pod.
* **Limits** are the absolute **maximum** you are allowed to consume (your ceiling). Enforced by the **Container Runtime**.

```yaml
resources:
  requests:
    memory: "256Mi"  # Reserving 256 Mebibytes of RAM
    cpu: "500m"      # Reserving 0.5 (half) of a CPU core
  limits:
    memory: "512Mi"  # Hard ceiling of 512 Mebibytes of RAM
    cpu: "1"         # Hard ceiling of 1 full CPU core
```

### What happens when a container hits its limits?
* **Overheating CPU (Throttling):** Kubernetes **will NOT kill your container**. Instead, it transparently **throttles (slow down)** the container using the Linux kernel's CFS engine.
* **Running out of Memory (OOMKilled):** Memory cannot be compressed. The Linux kernel's **OOM Killer** will step in and **instantly terminate your container process**, showing an error: **`OOMKilled`**.

### The "Quality of Service" (QoS) Classes
1. **Guaranteed:** You set both requests and limits to the *exact same values*. These are the most stable pods.
2. **Burstable:** Your requests are lower than your limits. Allows your application to idle cheaply but temporarily scale up for spikes.
3. **BestEffort:** You don't configure *any* requests or limits at all. The second a worker node experiences a resource crunch, **BestEffort pods are the very first to be terminated**.

---

## 18. Volume

In Kubernetes, a **Volume** is a directory containing data that can be accessed by the containers inside a Pod. By default, data stored inside a running container is **ephemeral**. A Volume solves this problem by decoupling storage from the individual container's lifecycle.

### Common Kubernetes Volume Types

#### Ephemeral Local Storage
* **`emptyDir`:** A clean directory created inside the worker node's drive space when a Pod is assigned to it. It starts out completely empty and is deleted when the Pod is deleted.
* **`hostPath`:** Mounts a physical directory from the underlying **worker node's host file system** directly into the container.

#### Configuration Injection
* **`configMap` / `secret`:** Allows you to inject configuration settings or secure API credentials as physical files.

#### Persistent Volumes (For Databases)
* **PersistentVolume (PV):** A piece of actual storage in the cluster that has been provisioned by an administrator or dynamically created via a cloud provider.
* **PersistentVolumeClaim (PVC):** A ticket or request written by a developer asking for a specific size and access mode of storage.

### What a Volume Looks Like (YAML Blueprint)
```yaml
apiVersion: apps/v1
kind: Pod
metadata:
  name: cache-app-pod
spec:
  containers:
  - name: web-container
    image: nginx
    volumeMounts:
    - name: shared-cache-space
      mountPath: /var/cache/data
  volumes:
  - name: shared-cache-space
    emptyDir: {}
```

---

## 19. Access Modes

In Kubernetes, **Access Modes** define how a network persistent plugin or storage volume can be mounted onto your physical or virtual worker nodes.

| Access Mode | CLI Shorthand | What it means | Common Use Case |
| :--- | :--- | :--- | :--- |
| **`ReadWriteOnce`** | `RWO` | Mounted as read-write by a **single node** at a time. | Standard SQL databases (e.g., PostgreSQL). |
| **`ReadOnlyMany`** | `ROX` | Mounted as read-only by **many nodes** simultaneously. | Static content distribution (e.g., assets shared across Nginx pods). |
| **`ReadWriteMany`** | `RWX` | Mounted as read-write by **many nodes** simultaneously. | Shared network filesystems (e.g., AWS EFS, NFS). |
| **`ReadWriteOncePod`** | `RWOP` | Mounted as read-write by a **single Pod** in the entire cluster. | Restricting microservice storage strictly to one active Pod. |

### What it looks like (YAML Blueprint)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

---

## 20. Static and Dynamic Provisioning

### Static Provisioning (The Manual Way)
In a static configuration, a cluster administrator must act as a middleman. Before any developer can request storage, the admin must physically provision a storage disk and write a specific `PersistentVolume` manifest to represent it.
* **The Problem:** If a developer requests a 10Gi disk, but the admin only pre-created 100Gi disks, Kubernetes will bind the 10Gi request to a 100Gi disk, wasting 90 Gigabytes of storage.

### Dynamic Provisioning (The Cloud-Native Way)
Dynamic provisioning eliminates manual intervention entirely. Instead of pre-allocating physical disks, the cluster administrator defines a blueprint called a **StorageClass**. When a developer deploys a `PersistentVolumeClaim` (PVC), the `StorageClass` automatically calls the cloud provider's API to purchase, format, and mount the exact disk space required on the fly.

---

## 21. Reclaim Policies

A **Reclaim Policy** determines what happens to a physical storage disk (the **PersistentVolume / PV**) after its user is done with it—specifically, when a developer deletes the **PersistentVolumeClaim (PVC)** that was bound to it.

### The 3 Reclaim Policies
1. **`Delete` (Automatic Cleanup):** When the PVC is deleted, Kubernetes automatically contacts the cloud infrastructure and **permanently destroys the physical disk** to stop billing.
2. **`Retain` (Manual Safeguard):** When the PVC is deleted, the physical disk and the `PersistentVolume` object continue to exist. However, the volume is marked as **`Released`**. No other Pod or PVC can reuse this volume until a cluster administrator manually cleans or recovers the data.
3. **`Recycle` (Deprecated):** Performs a basic data wipe (`rm -rf /volume/*`) on the disk and makes it instantly available for a new PVC claim. Deprecated in modern Kubernetes versions.

### Setting the Reclaim Policy on a StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: secure-database-storage
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain
```

---

## 22. StorageClass

A **StorageClass** is an administrative blueprint that acts as an **automated storage provisioner**. It allows cluster administrators to define different tiers of storage (e.g., "fast-ssd", "cheap-hdd") that developers can dynamically provision on-demand.

### Anatomy of a StorageClass (YAML Example)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd-storage
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
parameters:
  type: gp3
  iops: "3000"
```

### Core Configuration Fields Explained
* **Provisioner (`provisioner`):** The driver/plugin that talks to the cloud or infrastructure infrastructure API (e.g., `ebs.csi.aws.com`).
* **Volume Binding Mode (`volumeBindingMode`):** 
  * `Immediate`: The disk is created immediately when the PVC is submitted.
  * `WaitForFirstConsumer`: Kubernetes waits until a Pod is scheduled to a specific node before creating the disk, ensuring the disk is created in the **exact same availability zone** as the Pod.
* **Allow Volume Expansion (`allowVolumeExpansion`):** When set to `true`, it allows developers to upscale their disk size later by simply updating the PVC manifest, without losing data or recreating the resource.
