# The Kubernetes Book — Chapter Notes

## About the book

*The Kubernetes Book* (2025 edition, Kubernetes 1.32) by Nigel Poulton covers Kubernetes from first principles to production security in 17 chapters. These notes follow the book chapter by chapter: concepts first, then the commands and YAML you need, a **Deeper dive** with worked examples and comparisons, then the takeaways. Anything marked **Going further** is extra context not in the book (including updates since the 2025 edition), so you can tell the two apart.

- **Sample code:** `git clone https://github.com/nigelpoulton/TKB.git`, then `cd TKB && git fetch origin && git checkout -b 2025 origin/2025`.
- **Windows users:** where the book splits a command with backslashes, run it on one line or swap backslashes for backticks.
- **Convention:** Kubernetes objects are capitalised (Pod, Service, LoadBalancer) to separate them from generic terms like a cloud load balancer.
- **Book arc:** Ch 1–3 background and setup · Ch 4–8 core workload and networking objects · Ch 9 Wasm · Ch 10–13 service discovery, storage, config and stateful apps · Ch 14–15 API and RBAC · Ch 16–17 security.

## Chapter 1 — Kubernetes primer

Kubernetes deploys, scales, self-heals and updates applications whose features are packaged and run as containers. It is often called the operating system of the cloud.

### Decoding the jargon

"Kubernetes is an orchestrator of containerized cloud-native microservices applications."

- **Orchestrator:** a system that deploys apps and responds to change on its own: scaling with demand, self-healing, rollouts and rollbacks. You configure it once, then it runs without you.
- **Containerization:** packaging apps and dependencies as images and running them as containers. Think of containers as the next generation of VMs: smaller, faster, more portable. They run side by side with VMs, but are the first choice for new apps.
- **Cloud native:** apps with cloud-like behaviour (auto-scaling, self-healing, automated updates, rollbacks). Running an ordinary app in a public cloud does not make it cloud native.
- **Microservices:** an app built from many small, independent, specialised parts. Example: an e-commerce app split into web front-end, catalog, cart, auth, logging and store = six microservices, each with its own team, release cycle and scaling. Usual pattern: one microservice per container; scaling = adding or removing containers.

### Where Kubernetes came from

- Built by Google engineers, partly in response to **AWS** (Google needed a way to abstract cloud infrastructure so customers could move off AWS) and **Docker** (explosive container growth needed management).
- Draws on Google's internal orchestrators **Borg** and **Omega**. Kubernetes is not open-source Borg; it shares their DNA.
- Open-sourced in **2014**, donated to the **CNCF**, Apache 2.0 licence. v1.0 shipped July 2015; v1.32 at the time of writing, about three releases a year.
- Its two lasting jobs: **abstract infrastructure** and **simplify application portability**.

### Kubernetes and Docker

- Early Kubernetes used Docker as its runtime. Docker grew bloated and alternatives appeared, so Kubernetes created the **Container Runtime Interface (CRI)** to make runtimes pluggable.
- **Kubernetes 1.24 removed Docker as a runtime.** Most clusters now use **containerd**, a stripped-down Docker optimised for Kubernetes.
- Docker images still run fine: Docker, containerd and Kubernetes all follow **OCI** standards.
- A cluster can run several runtimes, even on the same node (used for Wasm in Chapter 9).
- Docker Swarm and Mesosphere DCOS competed in 2016–17; Kubernetes won. Swarm survives as a simple option for small companies.

### The name

- From the Greek for **helmsman** (the person who steers a ship), hence the ship's-wheel logo.
- Engineers wanted to call it *Seven of Nine* (a Borg drone in Star Trek); the logo's **seven spokes** nod to that.
- **K8s** ("kates"): the 8 stands for the eight letters between K and s.

### The operating system of the cloud

- Linux and Windows abstract server resources and schedule processes; **Kubernetes abstracts cloud resources and schedules microservices.**
- You don't care which CPU core your code runs on; with Kubernetes you rarely care which node, failure zone or volume your app uses.
- Same app runs on AWS, Azure, GCP, Civo or on-prem, which enables **hybrid cloud, multi-cloud and cloud migration**.

### Deeper dive

**What "orchestration" means day to day.** You configure a few things once (which images, how many replicas, how to update), and from then on Kubernetes acts without you: it places workloads, replaces failures, scales with load, and rolls updates forward or back. Your job shifts from running commands to describing outcomes.

**Containers vs VMs, as the book frames them across chapters:**

|  | Virtual machines | Containers |
| --- | --- | --- |
| What's packaged | A full OS plus the app | The app and its dependencies, sharing the host kernel |
| Size and speed | Large; start in minutes | Small; start in seconds |
| Portability | Heavy to move | Easy to copy between hosts and registries (but tied to one CPU architecture and OS, see Ch 9) |
| Isolation | Strong: own kernel, hardware-enforced (Ch 17) | Weaker: kernel namespaces and cgroups |
| Where they fit today | Still common, side by side with containers | First choice for new apps |

**Why microservices suit Kubernetes.** Splitting the e-commerce app into six services means six small teams, six release cycles and six independent scaling decisions. Kubernetes makes that manageable: each service is its own set of Pods, so scaling the cart during a sale means adding cart Pods only, and a bug fix to the catalog ships without touching the web front end.

**The "OS of the cloud" analogy, side by side:**

|  | A server OS (Linux, Windows) | Kubernetes |
| --- | --- | --- |
| Abstracts | CPU cores, memory DIMMs, flash chips, NICs | Nodes, failure zones, storage volumes, cloud load balancers |
| Schedules | Processes | Microservices (Pods) |
| You rarely care about | Which core or DIMM your code uses | Which node, zone or volume your app uses |
| Enables | Running apps on any hardware | Hybrid cloud, multi-cloud, migrating between clouds |

**Kubernetes milestones mentioned in the book** (newest first):

| When | Milestone | Chapter |
| --- | --- | --- |
| v1.32 | Version current at the time of writing; \~3 releases a year | 1 |
| v1.29 | Native sidecars reach beta; KMS v2 goes GA | 4, 16 |
| v1.28 | Native sidecar containers introduced (alpha) | 4 |
| v1.27 | Official CVE feed; default seccomp profile available to all containers | 16 |
| v1.26 | All release binaries and images cryptographically signed | 16 |
| v1.24 | Docker removed as a container runtime | 1 |
| v1.19 | Ingress and seccomp go GA | 8, 16 |
| v1.8 | RBAC becomes stable | 17 |
| July 2015 | Kubernetes v1.0 ships | 1 |
| 2014 | Google open-sources Kubernetes and donates it to the CNCF | 1 |

**Going further (not in the book):** the CNCF also hosts many projects you'll meet around Kubernetes, such as containerd, CoreDNS, etcd, Helm and Prometheus. Knowing they share the same foundation helps explain why they integrate so cleanly.

### Takeaways

- Kubernetes = cluster + orchestrator, born from Google's experience running billions of containers a week.
- It is the industry standard platform for cloud-native apps, owned by the CNCF.
- Docker images still work; Docker the runtime is gone (containerd is the default).

## Chapter 2 — Kubernetes principles of operation

A Kubernetes cluster is a set of control plane nodes (the brains) and worker nodes (where apps run). You describe what you want in YAML, and controllers keep the cluster matching it.

&#91;embedded content: cluster architecture · control plane and worker node components\]

Controllers, the scheduler and the kubelet never talk to each other directly; they all watch and update objects through the API server.

### Cluster and orchestrator

- **Cluster:** one or more nodes providing CPU, memory and other resources. Nodes can be physical servers, VMs or cloud instances, on ARM or x86-64.
- **Control plane nodes must be Linux;** worker nodes can be Linux or Windows. One cluster can mix both, and Kubernetes schedules apps to the right OS.
- Every cluster needs at least one control plane node; production should run **three or five**, spread across availability zones. Every control plane node runs every control plane service.
- Dev/test clusters often run apps on control plane nodes; production usually keeps apps on workers only.
- **Orchestrator:** deploys apps across nodes and failure zones, fixes failures, scales with demand, manages rollouts and rollbacks.

### Control plane services

**API server** — the front end of Kubernetes. All commands and requests go through it, including traffic between control plane services. RESTful API over HTTPS; every request is authenticated and authorized. Deploying an app: describe it in YAML → post to the API server → authN/authZ → persisted in the cluster store → scheduled to nodes.

**Cluster store** — holds the desired state of every app and component; the **only stateful part** of the control plane. Built on **etcd**, usually one replica per control plane node; busy, large clusters may run a separate etcd cluster.

- HA is not backup: you still need a way to recover the store.
- etcd wants an **odd number of replicas** to avoid **split brain**. With 4 nodes split 2/2, neither side has a quorum; with 3 nodes split 1/2, the pair knows it has the majority.
- In a split brain etcd goes **read-only**: running apps keep working, but Kubernetes cannot scale or update them.
- etcd uses the **RAFT** consensus algorithm to keep writes consistent.

**Controllers and the controller manager** — controllers (Deployment, StatefulSet, ReplicaSet and many more) are background **watch loops that reconcile observed state with desired state**. Ask for three replicas and a controller makes sure three healthy replicas exist. The controller manager spawns and manages them.

**Scheduler** — watches the API server for new tasks, identifies capable nodes, assigns tasks.

1. **Filter:** predicate checks for taints, affinity and anti-affinity rules, port availability, free CPU and memory.
2. **Rank:** points for already having the image, free CPU/memory and current task count; the highest score wins.
3. If no node fits, the task stays **Pending**. With node autoscaling, a Pending task triggers a new node.

**Cloud controller manager** — on public clouds (AWS, Azure, GCP, Civo), integrates the cluster with cloud instances, load balancers and storage. Example: an app asks for a load balancer and it provisions one of the cloud's.

### Worker node components

- **kubelet:** the main agent. Watches the API server for tasks, tells the runtime to run them, reports status back. If a task won't run, it reports the problem and lets the control plane decide.
- **Runtime:** pulls images and starts/stops containers. Usually **containerd**; OpenShift uses **CRI-O**; Docker is no longer supported. A node can have several runtimes.
- **kube-proxy:** implements cluster networking and load-balances traffic to tasks on the node.

### Packaging apps: Pods inside controllers

Kubernetes runs containers, VMs, Wasm apps and more, but **everything must be wrapped in a Pod**. The book's courier analogy: use the approved packaging and labels, and the courier handles the logistics; controllers are the extras like insurance or express delivery.

- The **container** wraps the app and its dependencies.
- The **Pod** wraps the container so it can run on Kubernetes.
- The **Deployment** wraps the Pod and adds self-healing, scaling and rollouts.

### The declarative model and desired state

- **Desired state** = what you want; **observed state** (also called actual or current state) = what you have; **reconciliation** = keeping them in sync.

1. Describe desired state in a YAML manifest (images, replicas, ports...).
2. Post it to the API server, usually with `kubectl`.
3. After authN/authZ it is stored in the cluster store as a **record of intent**.
4. A controller notices observed state no longer matches.
5. It reconciles: schedules Pods, pulls images, starts containers, attaches networks.
6. It keeps watching, forever.

- **Imperative:** platform-specific scripts of commands to reach an end state. **Declarative:** a platform-agnostic description of the end state. Kubernetes supports both but prefers declarative: it fits version control and enables self-healing, autoscaling and rolling updates.
- Example: you ask for 10 replicas and a node with 2 of them dies → observed 8 ≠ desired 10 → controller schedules 2 more. Updating an image is a one-line YAML change; Kubernetes replaces the old replicas.

### Pods

- The **atomic unit of scheduling** (the VM is VMware's equivalent).
- Usually one container per Pod. Multi-container Pods suit service meshes, helpers that initialise environments, and tightly coupled helpers such as log scrapers. A **sidecar** is a helper container next to the main app (e.g. a mesh sidecar that encrypts traffic and adds telemetry), which keeps each container to a **single responsibility**.
- A Pod is a **shared execution environment**: network stack, volumes, shared memory. Containers in one Pod share its IP (e.g. app on `10.0.10.15:8080`, sidecar on `10.0.10.15:5005`) and talk to each other over `localhost`.
- Use multi-container Pods only for tightly coupled parts that share memory or storage; otherwise loosely couple single-container Pods over the network.
- All containers of a Pod are scheduled to **one node**. Startup is **atomic**: a Pod is ready only when all its containers run.
- **Unit of scaling:** scale by adding or removing Pods, never by adding containers to a Pod.
- **Mortal:** a replacement Pod has a new ID and a new IP, so design apps to tolerate Pod failures.
- **Immutable:** never change a running Pod; "updating" a Pod always means replacing it. Fits GitOps workflows.

### Deployments

You almost always deploy Pods through higher-level controllers: **Deployments, StatefulSets, DaemonSets**. Deployments add self-healing, scaling, rolling updates and versioned rollbacks to **stateless** apps.

### Services

- Failures, rollouts and scaling all create new Pods with new IPs (**IP churn**), so clients can't rely on individual Pods.
- A **Service** gives a group of Pods a reliable name and IP and load-balances across them.
- **Front end:** stable DNS name, IP and port that never change. **Back end:** uses **labels** to find a dynamic set of healthy Pods.

### Deeper dive

**One request, end to end.** Putting the chapter's components together, here is what happens when you deploy a 3-replica Deployment:

1. `kubectl apply -f deploy.yml` turns the YAML into an HTTPS REST request using your kubeconfig credentials.
2. The **API server** authenticates you, authorizes the action, and validates the object.
3. The Deployment is written to the **cluster store (etcd)** as a record of intent.
4. The **Deployment controller** (via the controller manager) sees a Deployment with no ReplicaSet and creates one; the **ReplicaSet controller** sees 0 of 3 Pods and creates 3 Pod objects.
5. The **scheduler** sees 3 unassigned Pods, filters and ranks nodes, and writes a node name into each Pod.
6. Each chosen node's **kubelet** sees a Pod assigned to it and asks the **runtime** (containerd) to pull the image and start the containers.
7. The kubelet reports status back; once all containers run, each Pod is Ready.
8. If a **Service** selects these Pods, its EndpointSlice is updated and **kube-proxy** on every node programs rules so Service traffic reaches them.
9. The controllers keep watching. If a Pod dies, observed state (2) ≠ desired state (3), and a replacement is created.

No component calls another directly; **each one watches the API server** and reacts to changes there.

&#91;embedded content: one request, end to end · the 9 steps above\]

The numbers match the steps above: kubectl talks only to the API server, and every other component watches it and writes back through it.

**Why odd etcd numbers (quorum maths).** etcd needs a majority, `floor(n/2) + 1`, to accept writes. (Going further: the formula isn't in the book, but it explains the book's split-brain example.)

| etcd replicas | Majority needed | Failures tolerated |
| --- | --- | --- |
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |

Four replicas tolerate no more failures than three, and a 2/2 network split leaves neither side with a majority. That's why the book recommends 3 or 5.

**How a controller thinks.** Every controller runs the same loop: observe current state through the API server → compare with desired state → act to close the gap → repeat. The book's two examples:

- **Failure:** desired 10 replicas, a node dies taking 2 → observed 8 → controller schedules 2 more.
- **Update:** you change the image tag in YAML → desired state now says "10 on the new version" → controller replaces old replicas with new ones.

**Scheduler detail.** Filtering removes nodes that can't run the Pod (taints, affinity rules, port conflicts, not enough CPU/memory). Ranking awards points, for example for already having the image cached, having more free resources, or running fewer tasks. Highest score wins; no candidates means Pending (and, with node autoscaling, a new node).

**When to use a multi-container Pod.** Only when containers are tightly coupled and must share memory, volumes or localhost networking (e.g. app + service mesh sidecar, app + log scraper). Otherwise use separate single-container Pods and connect them over the network, so each can scale and be updated independently.

**Common misconceptions the chapter corrects:**

- *"Kubernetes runs containers."* It runs **Pods**; containers (and VMs, Wasm apps) run inside them.
- *"You scale by adding containers to a Pod."* You scale by adding **Pods**.
- *"You can update or restart a Pod."* Pods are immutable and mortal; you **replace** them, and the replacement has a new ID and IP.
- *"Clients should connect to Pods."* Pod IPs churn; clients connect to a **Service**.
- *"An HA etcd is a backup."* It isn't; you still need backup and recovery for the cluster store.

### Takeaways

- Control plane = API server + cluster store (etcd) + scheduler + controllers (+ cloud controller manager). Workers = kubelet + runtime + kube-proxy.
- Everything flows through the API server; desired state lives in etcd; controllers reconcile.
- Pods run workloads, Deployments add self-healing, scaling and rollouts, Services add stable networking.

## Chapter 3 — Getting Kubernetes

You need three tools: Docker, a Kubernetes cluster and `kubectl`. Docker Desktop gives you all three; a cloud cluster (Linode LKE in the book) is needed only for Chapter 8 (Ingress) and Chapter 11 (Storage), which use cloud load balancers and cloud storage.

### Option 1: Docker Desktop (local)

- Free for personal and education use; paid licence for companies with more than 250 staff or over $10M revenue.
- Needs **v4.38 or newer**. Windows: install **WSL 2** when prompted.
- Verify: `docker --version` and `kubectl version --client=true -o yaml`.
- Settings → General: enable **Use containerd for pulling and storing images**.
- Settings → Kubernetes: **Enable Kubernetes**, choose **kind (sign-in required)** (the kubeadm option gives only a single node), set **Nodes = 3**, tick **Show system containers**. Result: 1 control plane + 2 workers.
- Missing the kind option? Enter the Konami code (up up down down left right left right b a) on the Kubernetes settings page and enable **MultiNodeKubernetes** under experimental features.
- `kubectl get nodes` shows `desktop-control-plane`, `desktop-worker`, `desktop-worker2`. The nodes are actually containers (visible in the Containers tab), but the experience is identical.

### Option 2: Linode Kubernetes Engine (cloud)

- A **hosted Kubernetes** service: the provider builds the cluster and manages availability, performance and updates. The closest thing to a zero-effort production cluster.
- The book's link gives new users $100 credit for 60 days. Chapter 11 storage examples are written for Linode; other clouds need edited storage files.
- Cluster settings: label `tkb`, a nearby region, a recent version, HA control plane **No**, control plane ACL **Disabled**, node pool **3 × Linode 2GB** (shared CPU).
- `kubectl get nodes` shows three `lke...` nodes with ROLES `<none>`: all workers, because Linode hides the managed control plane.
- **Delete the cluster when done** to avoid charges.

### Configuring kubectl

- Keep kubectl within **one minor version** of the cluster (cluster 1.32 → kubectl 1.31–1.33).
- kubectl reads the **kubeconfig** file `~/.kube/config` (no file extension). macOS: `/Users/<user>/.kube/`; Windows: `C:\Users\<user>\.kube\`.
- **No existing kubeconfig:** create `~/.kube`, download the LKE kubeconfig into it, rename it `config`.
- **Existing kubeconfig:** rename it `config-bkp`, then merge:

```bash
export KUBECONFIG=~/.kube/config-bkp:~/.kube/tkb-kubeconfig.yaml
kubectl config view                      # check both clusters appear
kubectl config view --flatten > ~/.kube/config
export KUBECONFIG=~/.kube/config
kubectl config get-contexts
kubectl config use-context lke349416-ctx   # your context name will differ
```

Docker Desktop users can also switch context from the whale menu → Kubernetes Context.

### How kubectl and kubeconfig work

Every kubectl command: (1) is converted into an **HTTP REST request**, (2) is sent to the cluster in the **current context**, (3) authenticates with that context's credentials.

A kubeconfig defines four things:

| Section | Holds |
| --- | --- |
| `clusters` | Known clusters: friendly name, API server endpoint, CA certificate |
| `users` | Credentials: client certificate and key (the name is only a label) |
| `contexts` | Pairs of cluster + user, e.g. `ops-prod` = ops credentials on the prod cluster |
| `current-context` | The context kubectl uses by default |

```yaml
apiVersion: v1
kind: Config
clusters:
- name: shield
  cluster:
    server: https://192.168.1.77:8443     # API server endpoint
    certificate-authority-data: LS0tLS...
users:
- name: coulson
  user:
    client-certificate-data: LS0tLS1CRU...
    client-key-data: LS0tLS1CRUdJTiBFQyB
contexts:
- name: director        # context = cluster + user
  context:
    cluster: shield
    user: coulson
current-context: director
```

Useful: `kubectl config view` (redacts secrets), `kubectl config current-context`.

### Deeper dive

**Which cluster for which chapter:**

| Chapters | Docker Desktop (kind, 3 nodes) | Cloud cluster (LKE) |
| --- | --- | --- |
| 4–7, 10, 12–16 | Works | Works |
| 8 Ingress | Doesn't work (≤ v4.38) | Needed: uses a cloud load balancer |
| 9 Wasm | Not used: the chapter builds its own k3d cluster | Not used |
| 11 Storage | Doesn't work | Needed: uses Linode block storage CSI |
| 13 StatefulSets | Works with `dd-kind-sc.yml` | Works with `lke-sc.yml` |

**kubectl version skew, worked through.** The rule is ± one minor version:

| Cluster version | Supported kubectl |
| --- | --- |
| 1.31.x | 1.30 – 1.32 |
| 1.32.x | 1.31 – 1.33 |
| 1.33.x | 1.32 – 1.34 |

Check both sides with `kubectl version` (client and server) before chasing strange errors.

**How the KUBECONFIG merge works.** `KUBECONFIG` can hold a **colon-separated list** of files (semicolon on Windows). kubectl reads them all and merges their `clusters`, `users` and `contexts`. `kubectl config view --flatten` writes that merged view out, with certificate data inlined, as one self-contained file. That's why the book merges first and then points `KUBECONFIG` back at the single new `~/.kube/config`.

**Reading `kubectl get nodes` output:**

- **ROLES `control-plane`** on Docker Desktop: you can see and use the control plane node.
- **ROLES `<none>` on every LKE node:** all workers; the hosted provider runs and hides the control plane.
- **STATUS `Ready`:** the node's kubelet is healthy and reporting to the API server.
- **VERSION:** the kubelet's Kubernetes version (e.g. v1.32.2).

**Troubleshooting setup:**

- **Nodes don't start with `lke`, or you get auth errors:** you're on the wrong context or the kubeconfig file is wrong. Check `kubectl config current-context` and re-do the merge steps exactly.
- **The file must be named `config`** with no `.yaml` extension; turn on "show file extensions" to be sure.
- **Only one node on Docker Desktop:** you picked the kubeadm option; choose **kind (sign-in required)** instead (or enable MultiNodeKubernetes via the Konami code).
- **Unexpected cloud bills:** LKE nodes cost money after the free credit; delete the cluster when you're done.

**Going further (not in the book):** a few handy kubeconfig commands — `kubectl config rename-context <old> <new>` (give LKE's long context name something friendly), `kubectl config delete-context <name>`, and `kubectl cluster-info` (shows the API server URL for the current context, used in Ch 15).

### Takeaways

- Docker Desktop's multi-node kind cluster covers most of the book; other local options are k3d, KinD and minikube.
- A kubeconfig context = which cluster + which credentials. Switching context switches where commands go.

## Chapter 4 — Working with Pods

Every app on Kubernetes runs in a Pod: deploying, scaling, updating and terminating an app all mean creating or deleting Pods. Pods are mortal and immutable, and Kubernetes never restarts a Pod, it replaces it.

### Why Pods exist

- **Abstraction layer:** Kubernetes manages Pods without caring what's inside. Containers and Wasm apps use standard Pods, controllers and runtimes. Serverless functions need **Knative**; VMs need **KubeVirt**, which runs them in a **VirtualMachineInstance (VMI)**, a modified Pod.
- **They augment workloads with:** resource sharing, advanced scheduling, health probes, restart policies, security policies, termination control, volumes. Pods stay lightweight despite all this.
- Explore every field: `kubectl explain pods --recursive | more` (1,000+ lines) or drill in, e.g. `kubectl explain pod.spec.restartPolicy`.

### Shared execution environment

Containers in one Pod share these Linux namespaces:

| Shared | Namespace |
| --- | --- |
| Filesystem and volumes | mnt |
| Network stack (one IP, ports, localhost) | net |
| Memory | IPC |
| Process tree | pid |
| Hostname | uts |

Example: a sidecar syncs static content from Git into a shared volume; the main container serves it. Outside clients reach `10.0.10.15:8080` (app) and `:5005` (sidecar).

### Scheduling Pods

All containers in a Pod land on the same node. Put containers in one Pod only if they must **share resources**; if you just want two workloads on the same node, use separate Pods plus:

- **nodeSelectors** — simplest: a list of labels; the Pod only goes to nodes with all of them.
- **Affinity / anti-affinity** — a more powerful nodeSelector. Affinity attracts, anti-affinity repels; **hard** rules must be obeyed, **soft** rules are best effort; can select on nodes or on other Pods. E.g. a hard node-affinity rule on `project=tkb` means no matching node = Pod not scheduled; soft = scheduled anyway.
- **Topology spread constraints** — spread Pods across zones for HA, or across custom domains (near data, near clients).
- **Resource requests and limits** — every Pod should set them, or the scheduler may put it on a node without enough resources.

### Deploying a Pod, step by step

1. Define the Pod in a YAML manifest.
2. Post it to the API server.
3. Request is authenticated and authorized.
4. Pod spec is validated.
5. Scheduler filters nodes (selectors, affinity, spread constraints, resources...).
6. Pod assigned to a healthy node that meets all requirements (none → **Pending**).
7. The node's kubelet sees the assignment.
8. Kubelet downloads the spec and asks the runtime to start it.
9. Kubelet monitors and reports status to the API server.

Deployment is **atomic**: a Pod serves requests only when all its containers run.

### Lifecycle

- **Pending** (finding a node, pulling images) → **Running** (all containers up) → stays Running if long-lived (web server) or goes to **Succeeded** when a short-lived task (batch job) completes; failure ends in **Failed**.
- **Mortal:** a failed or deleted Pod cannot be restarted; a controller replaces it with a new Pod (new UID, new IP, no state). Same for node failure, eviction, scaling, updates and rollbacks.
- **Immutable:** never modify a deployed Pod; create a new one. Store data that must outlive the Pod in a **volume**.
- VMs are the opposite (mutable, immortal: restartable, reconfigurable, migratable), which is why KubeVirt uses VMIs and custom controllers.

### Restart policies (container-level)

Kubernetes can't restart Pods but the local kubelet can restart **containers**, governed by Pod-wide `spec.restartPolicy` (applies to all containers except init containers):

| Policy | Behaviour | Typical use |
| --- | --- | --- |
| `Always` (default) | Always restart | Long-lived: web servers, data stores, queues |
| `OnFailure` | Restart only on non-zero exit | Batch jobs that should finish |
| `Never` | Never restart | Jobs where failure doesn't matter |

### Static Pods vs controllers

- **Static Pod** (deployed directly from a Pod manifest, rare): managed only by the node's kubelet, so no self-healing, scaling or rolling updates; if the node dies, so does the Pod.
- **Via a workload controller** (normal): an HA controller can recreate Pods on other nodes, scale them, roll updates and roll back.

### The pod network

- Every cluster runs a **pod network**, usually a flat Layer-2 overlay across all nodes: every Pod can reach every other Pod directly, on any node. Nodes connect to external networks, not to the pod network itself.
- Provided by a third-party plugin via the **Container Network Interface (CNI)**, chosen at build time. **Cilium** is the most popular (security, observability).
- New clusters are usually wide open; secure them with **Network Policies**.

### Multi-container Pods

Follow the **single responsibility principle** (separation of concerns): one job per container. Usually separate Pods talk over the network; share a Pod when containers must share a volume or memory.

**Init containers** (`spec.initContainers`)

- Start and **complete before** the app containers start; run **once**.
- Prepare the environment: wait for a remote API to be ready, clone a repo, etc., keeping that logic out of the app.
- Several run in listed order; all must complete first. A failing init container is retried, unless `restartPolicy: Never`, which fails the Pod.
- Limitation: they only run before the app, not alongside it.

**Sidecar containers**

- Add functionality without changing the app: log scraping, syncing content, brokering connections, munging data, encrypting traffic (service mesh).
- **Native sidecars**: alpha in v1.28, beta in v1.29, still beta but **enabled by default** in v1.32; used by Argo CD and Istio.
- Defined as an **init container with `restartPolicy: Always`**. Guaranteed to start before the app, run alongside it, and stop after it. They keep init-container rules like startup order and can have probes.

### Hands-on: manifests and commands

Repo folder: `pods`. The basic Pod manifest (`pod.yml`):

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: hello-pod
  labels:
    zone: prod
    version: v1
spec:
  containers:
  - name: hello-ctr
    image: nigelpoulton/k8sbook:1.0
    ports:
    - containerPort: 8080
    resources:
      limits:
        memory: 128Mi
        cpu: 0.5
```

- Four top-level fields: **kind** (object type), **apiVersion** (schema version), **metadata** (name, labels — used later to connect Services), **spec** (containers, image, ports, resources).
- **Empathy as code** (Nirmal Mehta, DockerCon 2017): manifests document the app for new team members and ops, and live in version control.
- Images come from Docker Hub by default; prefix the registry URL to use another.

| Command | What it does |
| --- | --- |
| `kubectl apply -f pod.yml` | Post the manifest to the API server |
| `kubectl get pods [-o wide \| -o yaml]` | Status; `-o yaml` shows everything: `spec` = desired state, `status` = observed state (plus defaults you didn't set) |
| `kubectl get pods --watch` | Stream status changes |
| `kubectl describe pod <pod>` | Formatted overview incl. conditions and **Events** |
| `kubectl logs <pod> [--container <name>]` | Logs; defaults to the first container |
| `kubectl exec <pod> -- <cmd>` | Run a command in the container (remote execution) |
| `kubectl exec -it <pod> -- sh` | Interactive exec session (like SSH) |
| `kubectl edit pod <pod>` | Edit a live object (most Pod fields are rejected as immutable) |
| `kubectl delete pod <a> <b>` / `kubectl delete -f <file>` | Delete by name or by manifest |

- **Hostnames:** every container's hostname = the Pod's `metadata.name` (`HOSTNAME=hello-pod`), so Pod names must be valid DNS names (a–z, 0–9, `-`, `.`).
- **Immutability at two levels:** Kubernetes blocks changes to the Pod object (container name, port, resources); you are responsible for keeping the app and filesystem inside containers immutable. Installing `curl` via `apk add` in an exec session works but is an anti-pattern.

### Resource requests and limits

```yaml
resources:
  requests:      # minimums, used by the scheduler
    cpu: 0.5
    memory: 256Mi
  limits:        # maximums, enforced via the runtime
    cpu: 1.0
    memory: 512Mi
```

- Scheduler places the Pod on a node with enough for the **requests**; none → Pending, and the Cluster Autoscaler may add a node.
- Kubelet **reserves** the requests and has the runtime cap usage at the **limits**. A container may use spare node capacity above its request, never above its limit.
- Multi-container Pods: requests of all containers are summed.
- If you set only limits, Kubernetes sets **requests = limits**.

### Init container example (`initpod.yml`)

```yaml
spec:
  initContainers:
  - name: init-ctr
    image: busybox:1.28.4
    command: ['sh', '-c', 'until nslookup k8sbook; do echo waiting for k8sbook service; sleep 1; done; echo Service found!']
  containers:
  - name: web-ctr
    image: nigelpoulton/web-app:1.0
    ports:
    - containerPort: 8080
```

Status shows `Init:0/1` (Pod phase Pending) until you create the `k8sbook` Service (`kubectl apply -f initsvc.yml`); then `PodInitializing` → `Running`. The init container ends Terminated with exit code 0.

### Sidecar example (`initsidecar.yml`)

```yaml
spec:
  initContainers:
  - name: ctr-sync
    restartPolicy: Always          # this makes it a sidecar
    image: k8s.gcr.io/git-sync:v3.1.6
    volumeMounts:
    - name: html
      mountPath: /tmp/git
    env:
    - name: GIT_SYNC_REPO
      value: https://github.com/<you>/ps-sidecar...
    - name: GIT_SYNC_BRANCH
      value: master
    - name: GIT_SYNC_DEPTH
      value: "1"
    - name: GIT_SYNC_DEST
      value: "html"
  containers:
  - name: ctr-web
    image: nginx
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/
  volumes:
  - name: html
    emptyDir: {}
```

- Needs v1.29+. Fork `github.com/nigelpoulton/ps-sidecar`, put your fork URL in `GIT_SYNC_REPO`, apply, browse to the Service's EXTERNAL-IP (or `localhost` on Docker Desktop): "This is version 1.0". Edit `index.html` in your fork and refresh: the sidecar syncs the change.
- Pod shows `2/2 Running`; `describe` events show `ctr-sync` started before `ctr-web`.
- Show init vs app containers: `kubectl get pod -o "custom-columns=NAME:.metadata.name,INIT:.spec.initContainers[*].name,CONTAINERS:.spec.containers[*].name"`.

### Deeper dive

**Pod phases and the statuses you'll see:**

| What you see | Where | Meaning |
| --- | --- | --- |
| `Pending` (phase) | `describe` Status | Accepted, but not all containers running yet: waiting for a node or pulling images |
| `ContainerCreating` | `get pods` STATUS | Scheduled; image pulling / container starting |
| `Init:0/1` | `get pods` STATUS | 0 of 1 init containers finished; app containers wait |
| `PodInitializing` | `get pods` STATUS | Init containers done, app containers starting |
| `Running` (phase) | both | All containers started; READY shows how many are ready (e.g. `1/1`, `2/2`) |
| `Succeeded` / `Completed` | both | Every container exited 0 (batch jobs, the `startup-pod` in Ch 12) |
| `Failed` | phase | At least one container exited with an error and won't be restarted |
| `Terminating` | `get pods` STATUS | Being deleted (graceful shutdown in progress) |

**Reading the Events section.** A healthy start in `kubectl describe pod` reads top to bottom as: `Scheduled` (assigned to a node) → `Pulling` → `Pulled` (image download, the slowest step) → `Created` → `Started`. When a Pod is stuck, the last event usually tells you why: no `Scheduled` means a scheduling problem (resources, selectors, taints); a `Pulling` that never becomes `Pulled` means an image or registry problem.

**Why `-o yaml` shows so much more than your 17-line file.** Two sources: (1) **defaults** for every field you didn't set (e.g. `imagePullPolicy: IfNotPresent`, `namespace: default`, `restartPolicy: Always`, and requests copied from limits), and (2) the **`status`** block, which is observed state and never part of your YAML. `spec` = what you asked for; `status` = what Kubernetes sees.

**Scheduling controls compared:**

| Mechanism | Rule type | Selects on | Example |
| --- | --- | --- | --- |
| nodeSelector | Hard only | Node labels (all must match) | Only nodes labelled `disk=ssd` |
| Node affinity | Hard or soft | Node labels | Hard: must have `project=tkb`, else not scheduled; soft: prefer it, else schedule anywhere |
| Node anti-affinity | Hard or soft | Node labels | Avoid nodes labelled `project=tkb` |
| Pod affinity / anti-affinity | Hard or soft | Labels of other Pods | Run next to cache Pods; keep replicas apart |
| Topology spread constraints | Configurable | Domains (zones, racks, custom) | Spread replicas evenly across 3 zones |
| Requests and limits | Always applied | Free node CPU/memory | Needs 0.5 CPU and 256Mi |

**Requests vs limits, worked example.** A container requests 0.5 CPU / 256Mi and limits 1 CPU / 512Mi on a node with spare capacity. It is **guaranteed** 0.5 CPU and 256Mi (reserved by the kubelet), may **burst** above that if the node has idle resources, and is **capped** at 1 CPU / 512Mi by the runtime. A two-container Pod requesting 0.5 + 0.25 CPU needs a node with 0.75 CPU free.

**Init vs sidecar vs regular containers:**

|  | Init container | Native sidecar | App container |
| --- | --- | --- | --- |
| Defined in | `spec.initContainers` | `spec.initContainers` + `restartPolicy: Always` | `spec.containers` |
| Starts | Before app containers, in order | Before app containers, in order | After all init containers |
| Runs | To completion, once | For the Pod's whole life | For the Pod's whole life |
| Stops | When its task is done | **After** app containers stop | When the Pod ends |
| Typical job | Wait for a dependency, clone a repo | Service mesh proxy, log shipper, git-sync | The application |

**Static Pods in practice.** A Pod posted directly (not via a controller) has no controller watching it: if its container crashes the kubelet restarts it on the same node, but if the node dies the Pod is simply gone. That's why the book only uses bare Pods for demos.

**Going further (not in the book): health probes.** The book lists "application health probes" as a Pod feature without detail. Kubernetes has three, set per container:

- **startupProbe** — gives slow-starting apps time; other probes wait until it succeeds.
- **readinessProbe** — failing removes the Pod from Service EndpointSlices (no traffic) without restarting it.
- **livenessProbe** — failing makes the kubelet **restart the container**.

```yaml
containers:
- name: hello-ctr
  image: nigelpoulton/k8sbook:1.0
  readinessProbe:
    httpGet:
      path: /
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 10
  livenessProbe:
    httpGet:
      path: /
      port: 8080
    periodSeconds: 20
```

Readiness probes also matter for rollouts (Ch 6): a new replica only counts as ready, and the rollout only moves on, once its readiness probe passes.

### Takeaways

- Pods are the atomic unit of scheduling and abstract any workload type (containers, Wasm, serverless, VMs).
- Multi-container Pods share network, volumes and memory; use init containers to prepare and sidecars to augment.
- Deploy Pods through controllers (Deployments, Jobs, DaemonSets), always set requests and limits, and treat Pods as disposable.

## Chapter 5 — Virtual clusters with Namespaces

Namespaces divide one cluster into virtual clusters, each with its own users, RBAC, quotas and policies. They are **soft isolation** only: a compromised workload in one Namespace can affect the others.

### Key ideas

- **Not Linux kernel namespaces.** Kernel namespaces split an OS into containers; Kubernetes Namespaces split a cluster into virtual clusters.
- They enable **soft multi-tenancy**, e.g. `dev`, `test`, `qa` with different quotas and policies.
- Most objects are **namespaced** (Pods, Services, Secrets, PVCs, ServiceAccounts, ResourceQuotas). Some are **cluster-scoped** (Nodes, PersistentVolumes). Check with `kubectl api-resources` (NAMESPACED column).
- Objects go to the `default` Namespace unless you say otherwise.

### Use cases

- **Tenants** can be apps, teams, departments or customers. Most common: internal tenants, e.g. a prod cluster split into `finance`, `hr`, `corporate-ops`, each with its own users, permissions, quotas and policies.
- **External tenants** (customers) are rarely separated by Namespaces because isolation is soft. Strong isolation = **separate clusters on separate hardware**: more cost and management, but hard boundaries.

### Built-in Namespaces

| Namespace | Purpose |
| --- | --- |
| `default` | Where objects go if no Namespace is given |
| `kube-system` | Control plane components: internal DNS, metrics server |
| `kube-public` | Objects readable by anyone |
| `kube-node-lease` | Node heartbeats and node leases |

### Commands

```bash
kubectl get namespaces                    # or: kubectl get ns
kubectl describe ns default               # quotas, LimitRanges
kubectl get svc --namespace kube-system   # -n for short
kubectl get pods --all-namespaces
kubectl create ns hydra                   # imperative
kubectl apply -f shield-ns.yml            # declarative
kubectl delete ns hydra                   # deletes everything inside it
kubectl config set-context --current --namespace shield   # make shield the default for kubectl
kubectl config set-context --current --namespace default  # reset
```

Namespaces are first-class resources in the core `v1` API group, so both imperative and declarative styles work:

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: shield
  labels:
    env: marvel
```

### Deploying into a Namespace

- **Imperatively:** add `-n <ns>` / `--namespace <ns>` to commands.
- **Declaratively:** set `metadata.namespace` in every object's YAML. The `app.yml` example puts a ServiceAccount, a LoadBalancer Service (`the-bus`, port 8080, selector `env: marvel`) and a Pod (`triskelion`) all in `shield`.
- Connecting works exactly as in `default`: `curl localhost:8080` on Docker Desktop (substitute `localhost` if a 172.x EXTERNAL-IP appears).
- Deleting the Namespace deletes all objects in it.

### Deeper dive

**Namespaced vs cluster-scoped resources** (from `kubectl api-resources`, NAMESPACED column):

| Namespaced (live inside a Namespace) | Cluster-scoped (one per cluster) |
| --- | --- |
| Pods, Deployments, ReplicaSets, StatefulSets | Nodes |
| Services, EndpointSlices, Ingresses | PersistentVolumes |
| ConfigMaps, Secrets, ServiceAccounts | StorageClasses |
| PersistentVolumeClaims | Namespaces themselves |
| ResourceQuotas, Roles, RoleBindings | ClusterRoles, ClusterRoleBindings, CRDs (`scope: Cluster`) |

Rule of thumb: things an app owns are namespaced; things the whole cluster shares are cluster-scoped. Note the PVC/PV split: a claim belongs to a Namespace, the volume it binds to doesn't.

**What a Namespace can carry.** The book's `kubectl describe ns default` shows "No resource quota" and "No LimitRange resource" — the two per-Namespace controls you'd typically add, along with RBAC (Ch 14) and Pod Security labels (Ch 16):

| Control | What it does | Covered in |
| --- | --- | --- |
| ResourceQuota | Caps totals per Namespace (Pods, CPU, memory, objects) | Ch 16 example: max 100 Pods in `skippy` |
| LimitRange | Default and min/max requests/limits per container | Going further, below |
| Role / RoleBinding | Who can do what inside the Namespace | Ch 14 |
| Pod Security labels | Enforce baseline/restricted Pod policies | Ch 16 |
| DNS partition | `<svc>.<namespace>.svc.cluster.local` | Ch 10 |

**Soft vs hard isolation in one picture.** A cluster split into `dev` and `prod` Namespaces still shares the same nodes, kernel, control plane and pod network. A compromised `dev` workload can reach `prod`. Two clusters on separate hardware share nothing, at the cost of running and paying for two control planes.

**Hands-on gotchas:**

- After `kubectl config set-context --current --namespace shield`, every command targets `shield` — including after you delete it. Reset to `default` or commands return nothing.
- Applying `app.yml` warns about a missing annotation on the `default` ServiceAccount: every Namespace gets a ServiceAccount called `default` automatically, and `apply` is updating that existing one. The warning is harmless.
- Some Docker Desktop clusters show a 172.x EXTERNAL-IP; use `localhost:8080` instead.
- `kubectl delete ns` can take a while: it deletes every object inside first.

**Going further (not in the book): a LimitRange** sets per-container defaults so Pods that forget requests/limits still get sensible ones:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: shield
spec:
  limits:
  - type: Container
    defaultRequest:     # applied when a container sets no request
      cpu: 250m
      memory: 128Mi
    default:            # applied when a container sets no limit
      cpu: 500m
      memory: 256Mi
```

### Takeaways

- Namespaces are for organising, quotas, RBAC and policy, not for security boundaries; use separate clusters for hard multi-tenancy.
- Set a default Namespace in your kubeconfig context instead of typing `-n` everywhere, and remember to reset it.

## Chapter 6 — Kubernetes Deployments

Deployments are the standard way to run **stateless** apps: they add self-healing, scaling, zero-downtime rolling updates and versioned rollbacks. Under the hood a Deployment manages **ReplicaSets**, which manage the Pods.

### Architecture: resource + controller

- Every Kubernetes feature = a **resource** (defines the object; Deployment lives in `apps/v1`) + a **controller** (control plane watch loop that reconciles observed with desired state).
- **One Deployment per microservice**, each managing one or more **identical** Pods (e.g. `web` Deployment with 4 Pods, `cart` Deployment with 2).
- The Pod is defined inside the Deployment as the **Pod template** (`spec.template`).
- Nesting: **Deployment → ReplicaSet → Pods → containers.** The ReplicaSet provides self-healing and scaling; the Deployment adds rollouts and rollbacks. You only ever manage the Deployment, never the ReplicaSet or Pods directly.

### Autoscalers

| Autoscaler | What it scales | Notes |
| --- | --- | --- |
| Horizontal Pod Autoscaler (HPA) | Number of Pods | Installed on most clusters, widely used |
| Cluster Autoscaler (CA) | Number of nodes | Installed by default, widely used |
| Vertical Pod Autoscaler (VPA) | CPU/memory of Pods | Not default; replaces the Pod to resize it (disruptive, may move nodes); in-place resize is early alpha |

- HPA + CA example: HPA range 2–10 Pods; demand rises → HPA asks for 4, then 6 → no room → 2 Pods Pending → CA adds a node → scheduler places them. Scale-down works in reverse; removing a node evicts and replaces its Pods.
- **Multi-dimensional autoscaling** = combining methods (Pods + nodes, horizontal + vertical). **karmada** scales apps across multiple clusters.

### Desired state, declarative vs imperative

- Desired state = what you want; observed state = what you have; controllers reconcile.
- The book's analogy: declarative = "give me a chocolate cake for ten people"; imperative = every step from driving to the store to adding frosting.
- Imperative commands vary by platform (containerd vs CRI-O), are error-prone, and give no self-healing because no desired state is recorded.
- The ReplicaSet controller's loop: desired 10, observed 8 (failure or autoscaler, doesn't matter) → create 2. The same reconciliation powers self-healing, scaling, rollouts and rollbacks.

### Rolling updates: how they work

Zero-downtime rollouts work best when apps are:

1. **Loosely coupled via APIs** — update any microservice without affecting others.
2. **Backward and forward compatible** — clients of any version keep working (like a car: keep the steering wheel and pedals the same, change the engine freely).

Mechanics:

- You change the Pod template in the same Deployment YAML (e.g. a new image with a security fix) and re-post it.
- The Deployment controller creates a **new ReplicaSet** for the new template and scales it up while scaling the **old ReplicaSet** down, a few Pods at a time.
- Old ReplicaSets stay around with **0 Pods but their config intact** — that's what makes rollbacks easy.
- **Rollback** = the reverse: wind an old ReplicaSet back up and the current one down. It follows the same rules and pace as a rollout (a rollback is an update to an older image).
- Keep the Deployment manifest in version control and always update the same file.

### The Deployment manifest (`deploy.yml`)

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: hello-deploy             # must be a valid DNS name
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world           # must match the Pod template labels
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:                      # the Pod template
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:1.0
        ports:
        - containerPort: 8080
```

| Field | Meaning |
| --- | --- |
| `replicas` | Number of identical Pods to run |
| `selector.matchLabels` | Labels the Deployment/ReplicaSet use to find their Pods; must match `template.metadata.labels`; immutable after creation |
| `revisionHistoryLimit: 5` | Keep the last 5 ReplicaSets for rollback (too many bloats the object) |
| `progressDeadlineSeconds: 300` | Each new replica gets 5 minutes to start before it's reported as stalled |
| `minReadySeconds: 10` | Wait 10 s after each replica before replacing the next: throttles the rollout so you catch bad releases before they replace everything |
| `strategy.type: RollingUpdate` | Replace replicas incrementally |
| `maxUnavailable: 1` | May drop 1 below desired (10 → 9) |
| `maxSurge: 1` | May go 1 above desired (10 → 11) |

With desired = 10, the range 9–11 means **two Pods are replaced at a time**.

### Inspecting

- `kubectl get deploy hello-deploy` → READY, UP-TO-DATE, AVAILABLE.
- `kubectl describe deploy hello-deploy` → revision annotation, replicas (desired/updated/total/available), strategy, Old/New ReplicaSets, conditions.
- `kubectl get rs` → the ReplicaSet is named `<deployment>-<hash>`; the hash is a **crypto-hash of the Pod template**, so any template change creates a new ReplicaSet.
- `kubectl describe rs <name>` → "Controlled By: Deployment/hello-deploy"; the ReplicaSet's status rolls up into the Deployment's.

### Accessing the app

A LoadBalancer Service (`lb.yml`) with `selector: app: hello-world` on port 8080. `kubectl get svc lb-svc` → EXTERNAL-IP (`localhost:8080` on Docker Desktop, public IP/DNS in the cloud).

### Scaling manually

- Imperative: `kubectl scale deploy hello-deploy --replicas 5`.
- Danger: your YAML still says 10, so the next `kubectl apply` (e.g. for an image update) silently scales back to 10. **Keep manifests in sync by making changes declaratively.**
- Scaling is near-instant; rollouts are not.

### Rollout and rollback commands

```bash
# edit deploy.yml: image -> nigelpoulton/k8sbook:2.0
kubectl apply -f deploy.yml
kubectl rollout status deployment hello-deploy
kubectl rollout pause deploy hello-deploy
kubectl rollout resume deploy hello-deploy
kubectl rollout history deployment hello-deploy
kubectl rollout undo deployment hello-deploy --to-revision=1   # imperative; update your YAML afterwards
```

- Mid-rollout `get deploy` shows e.g. `9/10` ready, 6 up-to-date: `maxUnavailable` at work.
- A paused rollout shows condition `Progressing: Unknown (DeploymentPaused)`, old RS 3 replicas, new RS 6, revision 2.
- Scaling during a rollout spreads new replicas proportionally across old and new ReplicaSets (\~3 old, \~6 new for +10).
- Terms used interchangeably: rollout, release, rolling update, zero-downtime update. Every update is really a **replacement** (Pods are immutable).

### Labels and pod-template-hash

- Older Kubernetes let a Deployment **seize existing static Pods** whose labels matched its selector (5 static `app=front-end` Pods + a 10-replica Deployment = only 5 new Pods created).
- Now controllers add a system label **`pod-template-hash`** to Pods they create. ReplicaSet selectors include it (Deployment selectors don't), so controllers only manage their own Pods. Don't modify this label.
- See it with `kubectl get pods --show-labels`.

### Deeper dive

**The rollout from the hands-on, step by step** (simplified). Desired = 10, `maxSurge: 1` (up to 11 Pods), `maxUnavailable: 1` (no fewer than 9 available), `minReadySeconds: 10`:

| Round | Old ReplicaSet (1.0) | New ReplicaSet (2.0) | What's happening |
| --- | --- | --- | --- |
| Start | 10 | 0 | Revision 1 serving all traffic |
| 1 | 9 | 2 | One old Pod removed, two new created: the 9–11 window lets **two change at once** |
| 2 | 7 | 4 | Each new Pod must be ready, then wait 10 s, before the next round |
| 3 | 5 | 6 | The book's paused snapshot sits around here: 6 updated, 9 available |
| 4 | 3 | 8 |  |
| 5 | 1 | 10 |  |
| Done | 0 | 10 | Old ReplicaSet kept at 0 replicas, its config intact for rollback |

Clients never see downtime because at least 9 Pods are always serving, and both versions answer the same API (backward and forward compatible).

**How the three timing settings interact:**

- **`minReadySeconds`** sets the pace: larger values slow the rollout but give you time to spot a bad release before it replaces everything.
- **`progressDeadlineSeconds`** is the safety net: a new replica that can't become ready within 300 s marks the rollout as stalled (Progressing = False) instead of hanging forever.
- **`revisionHistoryLimit`** decides how far back you can roll: 5 means the last five ReplicaSets are kept.

**Scaling vs rolling, side by side:**

|  | Scaling | Rolling update |
| --- | --- | --- |
| What changes | `replicas` | The Pod template (image, env, ports...) |
| New ReplicaSet? | No, same one resized | Yes, new hash = new ReplicaSet |
| Speed | Near-instant | Paced by maxSurge, maxUnavailable, minReadySeconds |
| New revision? | No | Yes (`deployment.kubernetes.io/revision` increments) |

**The HPA + Cluster Autoscaler story in detail.** HPA range 2–10 → load rises, HPA asks for 4 Pods (fits) → load keeps rising, HPA asks for 6 → scheduler can't place 2 → they sit **Pending** → CA sees Pending Pods and adds a node → scheduler places them. On the way down, HPA removes Pods, CA may remove a now-empty node, and any Pods still on it are evicted and recreated elsewhere.

**Why the old ReplicaSet appears in Deployment output.** The Deployment owns its ReplicaSets (`Controlled By: Deployment/hello-deploy`), and ReplicaSet status rolls up into Deployment status. That's why `describe deploy` lists OldReplicaSets and NewReplicaSet, and why you should never edit ReplicaSets directly.

**Going further (not in the book):**

- **Record why each revision exists.** CHANGE-CAUSE shows `<none>` in the book. Annotate after each rollout and it appears in `kubectl rollout history`:

```bash
kubectl annotate deployment hello-deploy kubernetes.io/change-cause="Upgrade to 2.0"
kubectl rollout history deployment hello-deploy
kubectl rollout history deployment hello-deploy --revision=2   # full template of one revision
```

- **Imperative image change** (handy for tests, but update your YAML afterwards): `kubectl set image deployment/hello-deploy hello-pod=nigelpoulton/k8sbook:2.0`.
- **The other strategy:** `strategy.type: Recreate` deletes all old Pods before creating new ones, causing downtime but guaranteeing two versions never run together.
- **An HPA manifest** (needs the metrics server and CPU requests on the Pods):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hello-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hello-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # add Pods when average CPU passes 70% of requests
```

Once an HPA manages a Deployment, remove `replicas` from your YAML (or keep it in sync), or each `kubectl apply` will fight the autoscaler — the same drift problem the book warns about with `kubectl scale`.

### Takeaways

- Deployments (apps/v1) wrap ReplicaSets, which wrap Pods; manage everything through the Deployment, declaratively.
- Rollouts create a new ReplicaSet and shift Pods across under `maxSurge`/`maxUnavailable`/`minReadySeconds` control; old ReplicaSets enable instant-config rollbacks.
- Use autoscalers (HPA, CA) rather than manual scaling in production.

## Chapter 7 — Kubernetes Services

Pods are unreliable, so never connect to them directly: connect through a Service, which gives a set of identical Pods a DNS name, IP and port that never change.

### Why Services

Pods get deleted on scale-down, rolling updates, rollbacks, node maintenance/evictions and failures, and replacements get new IPs. A Service sits in front of one or more identical Pods and hides that churn.

- **Front end:** DNS name, IP address and port, guaranteed never to change.
- **Back end:** a **label selector**; traffic goes to healthy Pods with matching labels. E.g. clients hit `app1:8080` or `10.99.11.23:8080` and reach a Pod labelled `project=tkb`.
- The Service keeps an up-to-date list of healthy Pods through scaling, rollouts, rollbacks and failures.

### Labels and loose coupling

A Pod must carry **all** the selector's labels; extra labels don't matter. Selector `project=tkb, zone=prod` matches Pods A, B and D (D has extra labels) but not C (missing one).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tkb-2024
spec:
  replicas: 10
  template:
    metadata:
      labels:
        project: tkb      # create Pods with these labels
        zone: prod
---
apiVersion: v1
kind: Service
metadata:
  name: tkb
spec:
  ports:
  - port: 8080
  selector:
    project: tkb          # send to Pods with these labels
    zone: prod
```

### EndpointSlices

- Creating a Service makes the **EndpointSlice controller** create an EndpointSlice that tracks healthy Pods matching the selector; matching Pods are added, deleted ones removed.
- Flow: the app sends to the Service **name** → cluster DNS resolves it to the Service IP → the Service forwards to a Pod listed in the EndpointSlice.
- They replace the older **Endpoints** object (same function, better on big busy clusters). More than 100 Pods → multiple slices; dual-stack clusters get one per IP family.

### Service types

&#91;embedded content: Service types · how a request travels through LoadBalancer, NodePort and ClusterIP\]

A LoadBalancer Service creates the NodePort and ClusterIP plumbing for you; in-cluster clients use the ClusterIP name directly, and only internet clients go through the cloud load balancer.

| Type | Reachable from | How |
| --- | --- | --- |
| **ClusterIP** (default) | Inside the cluster only | Internal IP routable only on the pod network; name auto-registered in cluster DNS; all containers use cluster DNS. E.g. Service `skippy` lets every app connect to `skippy`. |
| **NodePort** | Outside, via any node | Adds a port in **30000–32767** on **every node**, mapped to the ClusterIP. Limits: high ports, and clients must know node IPs and health. |
| **LoadBalancer** | Internet | NodePort fronted by an HA **cloud load balancer** with a friendly DNS name on a low port. The easiest way to expose apps externally. |

```yaml
# NodePort
spec:
  type: NodePort
  ports:
  - port: 8080         # internal ClusterIP port
    targetPort: 9000   # app port in the container
    nodePort: 30050    # external port on every node
  selector:
    app: hello-world
---
# LoadBalancer
spec:
  type: LoadBalancer
  ports:
  - port: 8080         # load balancer port
    targetPort: 9000   # app port in the container
  selector:
    project: tkb
```

### Hands-on: imperative

- `kubectl apply -f deploy.yml` (Deployment `svc-test`, 10 Pods on 8080, label `chapter=services`).
- `kubectl expose deployment svc-test --type=LoadBalancer` inspects the Deployment and builds the right selector, ports and DNS records; the Service takes the Deployment's name.
- `kubectl get svc -o wide` columns: TYPE, CLUSTER-IP (internal only), EXTERNAL-IP (cloud IP/DNS, `localhost` on Docker Desktop; `<pending>` while provisioning), PORT(S) `8080:31755` = LB port : random NodePort, SELECTOR.
- The system Service **`kubernetes`** (ClusterIP, 443) exposes the API server to every Pod.

`kubectl describe svc svc-test` extra fields:

- **Endpoints:** healthy Pod IPs from the EndpointSlice.
- **Session Affinity:** `None` (default; a client's connections can hit different Pods) or `ClientIP` (sticky). Stickiness is an anti-pattern: design for **process disposability**.
- **External Traffic Policy:** `Cluster` (default; balance across Pods on all nodes, hides client source IP) or `Local` (only Pods on the receiving node, preserves source IP).

### Hands-on: declarative (the Kubernetes way)

```yaml
kind: Service
apiVersion: v1
metadata:
  name: svc-lb
spec:
  type: LoadBalancer
  ports:
  - port: 9000        # load balancer port
    targetPort: 8080  # app port in the container
  selector:
    chapter: services
```

- `kubectl apply -f lb.yml`, then browse to EXTERNAL-IP:9000.
- `kubectl get endpointslices` and `kubectl describe endpointslice <name>`: per-Pod address, Ready condition, TargetRef (Pod), NodeName.
- Deleting a Service deletes its Endpoints/EndpointSlices: `kubectl delete -f deploy.yml -f lb.yml`.

### Deeper dive

**The three port fields, which trip everyone up:**

| Field | Where it lives | Who uses it | Book example |
| --- | --- | --- | --- |
| `port` | The Service (ClusterIP, and the load balancer for LoadBalancer type) | Clients | `8080` (or `9000` in `lb.yml`) |
| `targetPort` | The container | The Service, when forwarding to Pods | `9000` (or `8080` in `lb.yml`) |
| `nodePort` | Every node's IP | External clients (NodePort type) or the cloud LB | `30050`; random in 30000–32767 if omitted |

If you omit `targetPort` it defaults to the same value as `port` — which is why `kubectl expose` produced `8080:31755` (LB port 8080 → NodePort 31755 → container 8080).

**Following one request through a LoadBalancer Service** (`svc-lb`, port 9000 → targetPort 8080):

1. Browser connects to the cloud load balancer at `212.2.247.202:9000`.
2. The load balancer forwards to a **healthy node** on the Service's NodePort (e.g. 30202).
3. The node maps the NodePort to the Service's **ClusterIP**.
4. The Service picks a Pod from its **EndpointSlice** (simple round-robin).
5. Traffic lands on that Pod's IP on **port 8080**, where the app listens.

**Selector matching rules.** A Pod is selected only if it has **every** label in the selector; extra labels are ignored. So `project=tkb, zone=prod` matches a Pod labelled `project=tkb, zone=prod, ver=4.1` but not one with only `project=tkb`. This is the same mechanism Deployments use, which is why a Deployment's Pod-template labels and its Service's selector must line up.

**The two traffic settings from `describe svc`:**

| Setting | Options | Effect | Use when |
| --- | --- | --- | --- |
| Session Affinity | `None` (default) / `ClientIP` | `ClientIP` sends a client's connections to the same Pod | Legacy apps that keep session state in the Pod (an anti-pattern) |
| External Traffic Policy | `Cluster` (default) / `Local` | `Cluster` balances across all nodes but hides the client IP; `Local` only uses Pods on the receiving node but keeps the client IP | You need real client IPs (logging, allow-lists) |

**EndpointSlice anatomy.** Each entry records the Pod's address, a `Ready` condition, the `TargetRef` (which Pod), the `NodeName` and zone. Labels tie the slice to its Service (`kubernetes.io/service-name=svc-lb`) and to its manager (`endpointslice-controller.k8s.io`). Slices cap at 100 endpoints each, and dual-stack clusters keep separate IPv4 and IPv6 slices.

**The `kubernetes` Service.** Every cluster has a ClusterIP Service called `kubernetes` in `default` (port 443) that fronts the API server. It's how Pods and Kubernetes-native apps find the API (Ch 10 uses it to test DNS).

**Going further (not in the book):**

- **Headless Services** (`clusterIP: None`) return Pod IPs directly from DNS instead of a virtual IP; Ch 13 uses them for StatefulSets.
- **ExternalName** Services map a cluster name to an external DNS name (a CNAME), e.g. pointing `db` at a managed database.
- **Quick local testing without a LoadBalancer:** `kubectl port-forward svc/svc-lb 9000:9000` tunnels from your laptop through the API server to the Service.

### Takeaways

- Services = stable front end + label-selector back end, kept current by EndpointSlices.
- ClusterIP for internal, NodePort for node-level external access, LoadBalancer for internet-facing apps; each builds on the previous.
- Manage Services declaratively in version-controlled YAML.

## Chapter 8 — Ingress

Ingress exposes many web apps through **one** cloud load balancer, routing by hostname or path at layer 7. Without it, every internet-facing app needs its own LoadBalancer Service (25 apps = 25 paid cloud load balancers, possibly over your cloud's limit).

### Terminology and status

- **LoadBalancer** = a Kubernetes Service of `type: LoadBalancer`; **load balancer** = your cloud's actual load balancer.
- Ingress went **GA in Kubernetes 1.19** after 15+ releases in beta. Service meshes overlap heavily, so with a mesh you may not need Ingress.
- **HTTP and HTTPS only** (ports 80/443).

### Architecture

- Lives in the **`networking.k8s.io/v1`** API. As usual: a **resource** (routing rules) + a **controller** (implements them).
- **No built-in Ingress controller** — unlike Deployments or Services, you must install one (some clouds offer one at cluster build). The book uses **NGINX**.
- Operates at **OSI layer 7** (application layer): it reads HTTP headers and routes on hostnames and paths. DNS points all hostnames at the same load balancer IP.

| Host-based | Path-based | Backend ClusterIP Service |
| --- | --- | --- |
| `shield.mcu.com` | `mcu.com/shield` | `shield` |
| `hydra.mcu.com` | `mcu.com/hydra` | `hydra` |

Traffic flow: client → DNS resolves to the load balancer → Ingress controller reads host/path from HTTP headers → matching rule sends it to the backend **ClusterIP** Service → Service delivers to a Pod.

### Ingress classes

Run several Ingress controllers on one cluster by mapping each to an **IngressClass** and assigning each Ingress object to a class (`spec.ingressClassName`). Installing NGINX creates class `nginx` (controller `k8s.io/ingress-nginx`). `kubectl get ingressclass`, `kubectl describe ingressclass nginx` (no short name).

### Hands-on (needs a cloud cluster; not Docker Desktop ≤ 4.38)

1. **Install NGINX controller:** `kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml` (creates a Namespace, ServiceAccounts, ConfigMaps, Roles, RoleBindings...). Check: `kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx` (the `admission-create/patch` Pods show Completed; that's normal).
2. **Deploy apps:** `kubectl apply -f app.yml` → Pods `shield` and `hydra` on 8080, behind ClusterIP Services `svc-shield` and `svc-hydra`.
3. **Create the Ingress** (`ig-all.yml`):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mcu-all
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /   # NGINX-specific
spec:
  ingressClassName: nginx
  rules:
  - host: shield.mcu.com          # host-based rule
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc-shield
            port:
              number: 8080
  - host: hydra.mcu.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc-hydra
            port:
              number: 8080
  - host: mcu.com                 # path-based rules
    http:
      paths:
      - path: /shield
        pathType: Prefix
        backend:
          service:
            name: svc-shield
            port:
              number: 8080
      - path: /hydra
        pathType: Prefix
        backend:
          service:
            name: svc-hydra
            port:
              number: 8080
```

- The **rewrite-target** annotation makes a best-effort rewrite of `mcu.com/shield` to `/`; it's controller-specific and doesn't work for every app (in the lab the image fails to load on path routes).
- Change or comment out `ingressClassName` and the annotation if you use a different controller.

4. **Inspect:** `kubectl get ing` → CLASS, HOSTS, ADDRESS (the load balancer IP/DNS; can take a minute), PORTS 80/443. `kubectl describe ing mcu-all` → rules table (host, path, backend with Pod IPs), annotations, and the **default backend** (where unmatched hosts/paths go; not every controller implements one).
5. **DNS for the lab:** add lines to `/etc/hosts` (Mac/Linux, root) or `C:\Windows\System32\drivers\etc\hosts` (Windows, as admin) mapping `shield.mcu.com`, `hydra.mcu.com` and `mcu.com` to the Ingress address (`127.0.0.1` if it shows localhost). In production use real DNS.
6. **Test:** `shield.mcu.com` and `hydra.mcu.com` reach their apps; bare `mcu.com` hits the default backend (on GKE: "response 404 (backend NotFound)"); `mcu.com/shield` and `/hydra` use path rewriting.
7. **Clean up:** delete `ig-all.yml`, `app.yml`, the controller URL, and your hosts-file entries.

### Deeper dive

**Important update since the book (2025 edition):** the **ingress-nginx controller** used in the hands-on was **retired in March 2026**. It gets no further releases, bug fixes or security patches, and the Kubernetes Steering and Security Response Committees urge users to migrate to the **Gateway API** or another Ingress controller ([Kubernetes statement, Jan 2026](https://www.kubernetes.io/blog/2026/01/29/ingress-nginx-statement/); [retirement announcement, Nov 2025](https://www.kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)). The Ingress **concepts** in this chapter still apply; pick a maintained controller for real clusters.

**The cost problem, made concrete:**

| Approach | 25 internet-facing apps need | Routing |
| --- | --- | --- |
| One LoadBalancer Service per app | 25 cloud load balancers (each billed; clouds may cap the count) | Layer 4: one IP/port per app |
| NodePort per app | 0 load balancers, but high ports and clients must track node IPs | Layer 4 |
| **Ingress** | **1** cloud load balancer on 80/443 | Layer 7: by hostname and path |

**Anatomy of a rule:**

| Field | Role | Example |
| --- | --- | --- |
| `host` | Match the HTTP Host header (omit = any host) | `shield.mcu.com` |
| `path` | Match the URL path | `/` or `/shield` |
| `pathType` | How to compare paths | `Prefix` |
| `backend.service.name` / `.port.number` | Where to send matches: a **ClusterIP** Service | `svc-shield` : `8080` |

`pathType` values (going further): **`Prefix`** matches by path segments (`/shield` matches `/shield` and `/shield/img`, not `/shieldx`); **`Exact`** matches only that exact path; **`ImplementationSpecific`** leaves it to the controller.

**Why the image broke on path routes.** With `rewrite-target: /`, a request for `mcu.com/shield` reaches the app as `/`. The HTML loads, but it references its image at an absolute path the app expects (e.g. `/images/...`). The browser asks for `mcu.com/images/...`, which matches no rule and falls to the default backend. Host-based routing has no such problem because paths aren't rewritten.

**Troubleshooting checklist:**

1. Controller running? `kubectl get pods -n ingress-nginx` (the `admission-create/patch` Pods are supposed to show Completed).
2. Class matches? `kubectl get ingressclass`, and the Ingress's CLASS column isn't empty when you rely on classes.
3. ADDRESS populated? An empty ADDRESS means the cloud load balancer isn't ready yet (or the cluster can't create one, e.g. Docker Desktop ≤ 4.38).
4. Backends healthy? `kubectl describe ing` shows each rule's backend with Pod IPs; `<error: endpoints not found>` means the Service selector matches no Pods.
5. DNS right? `/etc/hosts` (or real DNS) must point every hostname at the Ingress ADDRESS.
6. Unexpected 404 → the request matched no rule and hit the default backend.

**Going further (not in the book):**

- **HTTPS:** add a `tls` block listing hosts and a Secret of type `kubernetes.io/tls` holding the certificate and key; the controller terminates TLS on 443.

```yaml
spec:
  tls:
  - hosts:
    - shield.mcu.com
    secretName: shield-tls     # kubectl create secret tls shield-tls --cert=... --key=...
```

- **Gateway API** (`gateway.networking.k8s.io`) is the successor to Ingress. It splits the job into **GatewayClass** (which implementation), **Gateway** (listeners: ports, TLS) and **HTTPRoute** (host/path rules), so platform teams own Gateways and app teams own Routes. The Ingress API is frozen: new routing features go into Gateway API.

### Takeaways

- Ingress = many ClusterIP Services behind one load balancer, with host- and path-based HTTP routing.
- You usually install the controller yourself; IngressClasses let several coexist.
- Controller-specific behaviour (rewrites, default backend) lives in annotations and varies by controller.

## Chapter 9 — Wasm on Kubernetes

WebAssembly (Wasm) is the third wave of cloud computing after VMs and containers: smaller, faster, more portable and more secure. On Kubernetes, Wasm apps run through **containerd Wasm shims**, selected with node labels and a **RuntimeClass**.

### Wasm primer

- Appeared in **2017** to speed up web apps; now a **W3C standard** in every major browser.
- Write **Wasm**, not WASM: it's short for WebAssembly, not an acronym.
- Solomon Hykes (Docker founder): "If Wasm+WASI existed in 2008, we wouldn't have needed to create Docker." He also predicted Linux and Wasm containers running side by side, which is now true (even in the same Pod).
- **Technically:** a binary **instruction set architecture** (like ARM, x86, RISC-V). Languages compile to Wasm, which runs anywhere with a Wasm runtime.
- **WASI** (WebAssembly System Interface) lets sandboxed apps safely reach key-value stores, networks and the host. **WASI Preview 2** (a.k.a. WASI 0.2, wasip2) is a big step forward.

| Property | Wasm | Containers |
| --- | --- | --- |
| Security | **Deny-by-default** sandbox, battle-hardened on the web; access must be explicitly granted | Allow-by-default with broad access to a shared kernel |
| Portability | **Build once, run anywhere** with a runtime (even IoT/edge architectures) | Architecture-specific (no ARM image on AMD, no Windows image on Linux) → image sprawl |
| Startup | \~10 ms or less, sometimes microseconds; enables true **scale-to-zero** | Seconds (VMs: minutes) |
| Size | Tiny (the lab image is **104 kB**) | Several MB for hello world |

**Best fits today:** event handlers, serverless functions, anything needing very fast startup, IoT/edge, plugins and extensions. **Containers still better** for traditional cloud apps with heavy networking, I/O and service connections.

### How Wasm runs on Kubernetes (containerd)

- Normal path: Kubernetes → kubelet → **containerd** → **runc** builds the container and exits, leaving a **shim** process connecting the container to containerd.
- Kubernetes sees nothing below containerd, so you can swap runc + standard shim for a **Wasm shim**. One containerd instance can run Linux and Wasm workloads side by side.
- A Wasm shim is **one binary** = **runwasi** (the Rust library that talks to containerd) + an embedded Wasm runtime (Spin, Slight, WasmEdge, Lunatic, wws...). The runtime creates the Wasm host and executes the app.
- **Shim naming rule:** `containerd-shim-<runtime>-<version>`, e.g. `containerd-shim-spin-v2` (runc's is `containerd-shim-runc-v2`).
- When nodes have different shims, use **node labels + a RuntimeClass**: the RuntimeClass's `scheduling.nodeSelector` picks labelled nodes, and its `handler` names the shim.

**Deployment workflow**

1. Write the app and compile it to a Wasm binary.
2. Package it as an **OCI image** and push to an OCI registry.
3. Install the Wasm shim on at least one node and label the node.
4. Create a RuntimeClass with the node selector and handler.
5. Create a Pod/Deployment using the Wasm image and referencing the RuntimeClass (`runtimeClassName`).
6. Deploy: the scheduler picks a matching node → kubelet passes the work and shim info to containerd → containerd starts the app with that shim.

In practice, **SpinKube** (Fermyon) and cloud platforms automate most of this; the chapter does it by hand for understanding.

### Hands-on

**Prerequisites:** Docker Desktop with Wasm support, Rust ≥ 1.82 with the `wasm32-wasip1` target, Spin ≥ 3.1.2, k3d ≥ 5.8.1.

```bash
rustup target add wasm32-wasip1
spin --version
spin new tkb-wasm -t http-rust      # description "My first Wasm app", HTTP path /tkb
cd tkb-wasm                          # Cargo.toml, spin.toml, src/lib.rs
# edit src/lib.rs: .body("The Kubernetes Book loves Wasm!")
spin build                           # runs cargo build --target wasm32-wasip1 --release
# output: target/wasm32-wasip1/release/tkb_wasm.wasm
```

- `spin.toml` tells Spin how to build and run the app; `src/lib.rs` is the source.

**Package as an OCI image:**

```dockerfile
FROM scratch
COPY /target/wasm32-wasip1/release/tkb_wasm.wasm .
COPY spin.toml .
```

- `FROM scratch` = empty image: Wasm needs no Linux filesystem. At runtime the container is little more than namespaces and cgroups.
- Edit `spin.toml` so `[component.tkb-wasm] source = "tkb_wasm.wasm"` (file now sits at the image root).
- `docker build --platform wasi/wasm --provenance=false -t <you>/k8sbook:wasm-0.2 .` — `--platform wasi/wasm` sets OS/arch so containerd knows how to run it. Then `docker push`.

**Build a Wasm-ready k3d cluster:**

```bash
k3d cluster create wasm \
  --image ghcr.io/deislabs/containerd-wasm-shims/examples/k3d:v0.11.1 \
  -p "5005:80@loadbalancer" --agents 2
```

- Custom image with Wasm shims pre-installed; `-p` maps host port 5005 to the in-cluster Ingress on 80; 1 control plane + 2 agents.
- A node needs (1) containerd running and (2) a Wasm shim **installed and registered**. Check on a node: `docker exec -it k3d-wasm-agent-1 ash`, `ps | grep containerd`, `ls /bin | grep shim` (lunatic, runc, slight, spin, wws).
- Registration lives in containerd's `config.toml` (k3d: `/var/lib/rancher/k3s/agent/etc/containerd/config.toml`), e.g. `[plugins.cri.containerd.runtimes.spin] runtime_type = "io.containerd.spin.v2"`. Verify with `containerd --config <file> config dump | grep spin`.

**Label and RuntimeClass:**

```bash
kubectl label nodes k3d-wasm-agent-1 wasm=yes
kubectl get nodes --show-labels | grep wasm=yes
kubectl apply -f rc-spin.yml
kubectl get runtimeclass        # NAME rc-spin, HANDLER spin
```

`rc-spin.yml`, reconstructed from the fields the book describes:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: rc-spin
handler: spin              # containerd uses the spin shim
scheduling:
  nodeSelector:
    wasm: "yes"            # only nodes with this label
```

**Deploy the app** (`app.yml` = Deployment + ClusterIP Service + Ingress on `/`):

```yaml
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: wasm
    spec:
      runtimeClassName: rc-spin       # the key line
      containers:
      - name: testwasm
        image: nigelpoulton/k8sbook:wasm-0.1
        command: ["/"]
```

- `kubectl get pods -o wide` shows all 3 replicas on `k3d-wasm-agent-1`.
- `curl http://localhost:5005/tkb` → "The Kubernetes Book loves Wasm!"
- Clean up: `k3d cluster delete wasm` (or delete `app.yml` and the RuntimeClass), `docker rmi`, delete the `tkb-wasm` folder, switch kubectl context back.

### Deeper dive

**The three waves of cloud computing** (the book's opening figure):

| Wave | Unit | Typical start time | Portability | Default security |
| --- | --- | --- | --- | --- |
| 1. Virtual machines | Whole OS + app | Minutes | Hypervisor-specific images | Strong (own kernel) |
| 2. Containers | App + dependencies on a shared kernel | Seconds | One OS/architecture per image | Allow-by-default |
| 3. Wasm | Compiled Wasm binary | \~10 ms or less (sometimes µs) | Any host with a Wasm runtime | Deny-by-default sandbox |

**Why "containers aren't portable".** An image is built for one OS and CPU architecture. The author keeps two images for most book apps (linux/arm64 and linux/amd64) and has broken examples by forgetting one. A Wasm binary built on an ARM Mac runs unchanged on any architecture with a runtime, including exotic IoT and edge chips.

**Inside a node running Wasm:**

1. The kubelet passes the Pod to **containerd**, including the runtime handler from the RuntimeClass.
2. For Linux containers containerd uses **`containerd-shim-runc-v2`**: runc builds the container and exits; the shim keeps the link to containerd.
3. For Wasm it uses e.g. **`containerd-shim-spin-v2`**: a single binary where **runwasi** handles the containerd side and the embedded **Spin runtime** creates the Wasm host and runs the app.
4. The handler name in the RuntimeClass (`spin`) must match a runtime registered in containerd's `config.toml` (`[plugins.cri.containerd.runtimes.spin]` → `runtime_type = "io.containerd.spin.v2"`).

Three names have to line up: **RuntimeClass `handler`** → **containerd runtime name** → **shim binary** on disk.

**The `k3d cluster create` flags:**

| Flag | Does |
| --- | --- |
| `wasm` | Cluster name (kubectl context becomes `k3d-wasm`) |
| `--image ghcr.io/deislabs/containerd-wasm-shims/examples/k3d:v0.11.1` | Node image with lunatic, slight, spin and wws shims pre-installed |
| `-p "5005:80@loadbalancer"` | Host port 5005 → the cluster's Ingress on port 80 |
| `--agents 2` | Two worker nodes (plus one server/control plane) |

**The image build flags:**

- `--platform wasi/wasm` stamps the OS/arch as `wasi/wasm` so runtimes know it's a Wasm image.
- `--provenance=false` skips the build attestation manifest, keeping a plain single-platform image.
- `FROM scratch` + two `COPY` lines → a 104 kB image containing only `tkb_wasm.wasm` and `spin.toml`.

**Troubleshooting:**

- **Pods stuck Pending:** no node matches the RuntimeClass `nodeSelector` — check the node label (`kubectl get nodes --show-labels | grep wasm=yes`).
- **Pods fail to start / handler errors:** the shim isn't installed or registered on that node, or the handler name is wrong.
- **App returns 404 or fails:** `spin.toml`'s `source` still points at `target/wasm32-wasip1/release/...` instead of the image root.
- **curl to localhost:5005 fails:** the Ingress or Service in `app.yml` isn't up, or the cluster was created without the `-p` port mapping.

**Going further:** SpinKube (started by Fermyon) packages all of this — shim installation on nodes, RuntimeClass and a `SpinApp` custom resource — so you deploy Wasm apps without hand-labelling nodes.

### Takeaways

- Wasm compiles once and runs anywhere a runtime exists, in a deny-by-default sandbox with millisecond cold starts, but can't yet do everything containers can.
- Build Wasm apps into OCI images with normal Docker tooling, then run them on Kubernetes like containers.
- On containerd clusters: install and register a Wasm shim, label nodes, create a RuntimeClass, set `runtimeClassName`. SpinKube automates this in real life.

## Chapter 10 — Service discovery deep dive

Kubernetes makes service registration and discovery automatic: the cluster DNS registers every Service's name and IP, every container is configured to query it, and kube-proxy rules on each node turn ClusterIP traffic into Pod traffic.

### The problem

Apps talk to other apps through their **Services**. To send a request an app needs (1) the other Service's **name** — the developer's job — and (2) a way to turn the name into an **IP** — Kubernetes' job. Only step 1 is manual.

### The service registry: cluster DNS

- Every cluster has a built-in **cluster DNS** as its service registry: a Kubernetes-native app on the control plane.
- Two or more **Pods** (image usually `registry.k8s.io/coredns/coredns`) managed by a **Deployment** (`coredns` or `kube-dns`) behind a **Service always named `kube-dns`** (e.g. `10.96.0.10`, ports 53/UDP, 53/TCP, 9153/TCP). All live in `kube-system` with label `k8s-app=kube-dns`.

```bash
kubectl get pods   -n kube-system -l k8s-app=kube-dns
kubectl get deploy -n kube-system -l k8s-app=kube-dns
kubectl get svc    -n kube-system -l k8s-app=kube-dns
```

### Service registration (automatic)

1. Give the Service a name (developer), e.g. `valkyrie-web`.
2. Kubernetes checks the name is unique and assigns a **ClusterIP**.
3. The cluster DNS, which **watches the API server for new Services**, registers DNS **A and SRV records**.

The full flow: post Deployment + Service YAML → authN/authZ → ClusterIP allocated, config persisted to the cluster store → cluster DNS registers records → EndpointSlices created with matching healthy Pod IPs → **kube-proxy** on every node creates local routing rules (**IPVS**) so ClusterIP traffic reaches Pods. Apps need no registration logic.

### Service discovery

&#91;embedded content: service discovery · enterprise app calling the cer Service\]

Only step 1 involves the app; the DNS lookup, the gateway hops and the kernel redirect are invisible to it.

- The **kubelet** configures every container's `/etc/resolv.conf` with the kube-dns IP and **search domains**:

```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10      # ClusterIP of the cluster DNS
options ndots:5
```

- An **unqualified** short name (`ent`) gets a search domain appended to form an **FQDN** (`ent.default.svc.cluster.local`).
- **ClusterIP routing:** ClusterIPs are virtual IPs on the **service network**, which has **no routes**. The container sends the packet to its default gateway → the node → the node's default gateway, so the **node kernel** processes it. **kube-proxy** watches for Services and EndpointSlices and programs kernel rules that intercept ClusterIP traffic and redirect it to a healthy Pod IP.
- A **default gateway** is where a system sends traffic when it doesn't know where else to send it.

### Namespaces and the cluster domain

- The **cluster domain** (usually `cluster.local`) is the cluster's address space; names must be unique within it.
- FQDN format: **`<object-name>.<namespace>.svc.cluster.local`**.
- Namespaces partition the address space (`<svc>.dev.svc.cluster.local`, `<svc>.prod.svc.cluster.local`), so names need only be unique **within** a Namespace. Identical dev and prod setups can share one cluster.
- **Short names resolve to the local Namespace; use the FQDN to reach another Namespace.**

### Hands-on (`sd-example.yml`)

- Creates Namespaces `dev` and `prod`, a Deployment `enterprise` and ClusterIP Service `ent` (port 8080) in **each** (images `text-dev` / `text-prod`), plus a `jump` Pod (ubuntu) in `dev`.
- `kubectl get all --namespace dev` / `prod` → two `ent` Services with different ClusterIPs.

```bash
kubectl exec -it jump --namespace dev -- bash
cat /etc/resolv.conf                    # search dev.svc.cluster.local ...
apt-get update && apt-get install curl -y
curl ent:8080                           # Hello from the DEV Namespace!
curl ent.prod.svc.cluster.local:8080    # Hello from the PROD Namespace!
```

### Troubleshooting checklist

1. **Deployment and Pods up:** `kubectl get deploy` / `get pods -n kube-system -l k8s-app=kube-dns`.
2. **Logs healthy:** `kubectl logs <coredns-pod> -n kube-system` (shows `.:53`, CoreDNS version).
3. **Service up:** `kubectl get svc kube-dns -n kube-system` — its ClusterIP **must match** the nameserver in every container's `/etc/resolv.conf`.
4. **EndpointSlice populated:** `kubectl get endpointslice -n kube-system -l k8s-app=kube-dns` lists coredns Pod IPs on 53.
5. **Test from a tools Pod:** `kubectl run -it dnsutils --image registry.k8s.io/e2e-test-images/jessie-dnsutils:1.7`, then `nslookup kubernetes` → should return `kubernetes.default.svc.cluster.local` and its ClusterIP (e.g. 10.96.0.1). "can't resolve kubernetes" = DNS broken.
6. **Fix attempt:** `kubectl delete pod -n kube-system -l k8s-app=kube-dns` (the Deployment recreates them), then retest.

### Deeper dive

**How search domains expand a short name.** For the `jump` Pod in `dev`, `/etc/resolv.conf` lists `search dev.svc.cluster.local svc.cluster.local cluster.local`. Asking for `ent` tries, in order:

1. `ent.dev.svc.cluster.local` ← matches the local `ent` Service, so you reach DEV
2. `ent.svc.cluster.local`
3. `ent.cluster.local`

That's why the short name always lands in the **local** Namespace and you need the FQDN to cross Namespaces.

**`options ndots:5` (going further, not explained in the book):** a name with fewer than 5 dots is treated as "relative", so the search domains are tried **first**. Even `ent.prod.svc.cluster.local` has only 4 dots, so lookups may try a few suffixed names before the exact one; it still resolves correctly. Adding a trailing dot (`ent.prod.svc.cluster.local.`) marks a name as absolute and skips the search list.

**Name forms that all work from the `dev` Namespace:**

| You ask for | Resolves to | Reaches |
| --- | --- | --- |
| `ent` | `ent.dev.svc.cluster.local` | dev |
| `ent.dev` | `ent.dev.svc.cluster.local` (via `svc.cluster.local`) | dev |
| `ent.prod` | `ent.prod.svc.cluster.local` | prod |
| `ent.prod.svc.cluster.local` | itself | prod |
| `kubernetes.default` | `kubernetes.default.svc.cluster.local` | the API server Service |

**What gets registered.** For each Service the cluster DNS creates an **A record** (name → ClusterIP; AAAA for IPv6) and **SRV records** (which add port and protocol). For headless Services (Ch 13) the A records point at each Pod's IP instead of a single virtual IP.

**ClusterIP routing, step by step:**

1. The app resolves `cer` → ClusterIP `192.168.200.217` and sends a packet there.
2. The container has no route to the **service network**, so the packet goes to its default gateway (the node).
3. The node has no route either and sends it toward its own default gateway, so the **node kernel** handles it.
4. **kube-proxy** has already programmed kernel rules (IPVS or iptables) for every Service and EndpointSlice.
5. The rule rewrites the destination to a healthy **Pod IP** from the Service's EndpointSlice, and the packet goes out over the pod network.

ClusterIPs are never assigned to any interface; they exist only as these kernel rules, which is why you can't ping one from outside the cluster.

**Symptom → likely cause:**

| Symptom | Check |
| --- | --- |
| `nslookup: can't resolve kubernetes` | CoreDNS Pods down or crashing → `kubectl logs`, then delete the Pods to recreate them |
| Every lookup times out | `nameserver` in `/etc/resolv.conf` doesn't match the `kube-dns` ClusterIP, or the kube-dns EndpointSlice is empty |
| Short name fails, FQDN works | You're in a different Namespace from the Service |
| Name resolves but connection fails | DNS is fine; the Service has no healthy endpoints, or a Network Policy blocks traffic |
| Wrong environment answers | Short name resolved to the local Namespace's Service (e.g. dev instead of prod) |

### Takeaways

- Cluster DNS (CoreDNS behind the `kube-dns` Service) is the registry; registration is automatic because it watches the API server.
- Containers resolve names via `/etc/resolv.conf`; ClusterIPs are unroutable virtual IPs that kube-proxy kernel rules redirect to Pods.
- Short names stay in the local Namespace; cross-Namespace needs `<svc>.<ns>.svc.cluster.local`.

## Chapter 11 — Kubernetes storage

Apps request storage with a **PersistentVolumeClaim**; a **StorageClass** dynamically creates the external volume through the provider's **CSI plugin** and maps it to a **PersistentVolume** the Pod mounts. Hands-on examples use Linode (LKE) and won't work unchanged on other clouds.

### The big picture: three layers

1. **Storage providers** (left): external systems with block, file and object storage and advanced services (backup, replication, snapshots, encryption) — on-prem (EMC, NetApp) or cloud.
2. **Plugin layer** (middle): the **Container Storage Interface (CSI)**, the industry-standard interface between orchestrators and storage. Plugins are developed **out-of-tree**: no need to be open source, released on their own schedule. (Old **in-tree** plugins lived in the Kubernetes codebase and were tied to its release cycle.)
3. **Persistent volume subsystem** (right): standard API objects — **PV** maps an external volume, **PVC** grants access to a PV, **SC** makes it all automatic and dynamic.

&#91;embedded content: dynamic provisioning · Pod, PVC, StorageClass, CSI plugin, PV\]

The Pod never talks to the storage system; it only references a PVC, and the StorageClass and CSI plugin do the rest.

- Kubernetes prevents multiple Pods writing to the same PV, and enforces **1 external volume = 1 PV** (no splitting a 50GB volume into 2 × 25GB PVs).

### Providers and CSI plugins

- Each provider (a.k.a. provisioner) ships its own CSI plugin (a.k.a. driver), usually as a **Helm chart or YAML installer**; it runs as Pods in **`kube-system`**. Cloud platforms pre-install plugins for their native storage.
- Obvious limits: no AWS EBS on an Azure cluster; Pods may need to be in the same region/zone as their storage.
- Size, protection, snapshots, replication, encryption are configured through the plugin; features differ, so read its docs. Driver list: `kubernetes-csi.github.io/docs/drivers.html`.

### StorageClasses

- Resource `StorageClass` in **`storage.k8s.io/v1`**, short name `sc`. Each class represents a **tier** of storage (e.g. fast block, fast encrypted block, slow block, NFS file). Example request: an app needing 100GB fast encrypted block references a PVC on `sc-fast-encrypted`.

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fast-local
provisioner: ebs.csi.aws.com       # AWS EBS CSI plugin
parameters:                        # plugin-specific
  encrypted: true
  type: io1                        # AWS SSD
  iopsPerGB: "10"
allowedTopologies:                 # where to provision
- matchLabelExpressions:
  - key: topology.ebs.csi.aws.com/zone
    values:
    - eu-west-1a
```

- StorageClasses are **immutable**; give them meaningful names (other objects refer to them); `parameters` differ for every plugin.
- Workflow: install the CSI plugin → create StorageClasses → deploy Pods with PVCs that reference a class. The **SC controller** watches the API server for PVCs referencing its class.

**Pod + PVC + SC together** (`---` separates objects in one file):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: mypvc
  containers: ...
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fast
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
```

### Access modes and reclaim policies

| Access mode | Meaning |
| --- | --- |
| `ReadWriteOnce` (RWO) | One PVC binds read-write; others fail |
| `ReadWriteMany` (RWM) | Many PVCs bind read-write; usually file/object storage, rarely block |
| `ReadOnlyMany` (ROM) | Many PVCs bind read-only |

A PV can only be opened in **one** mode at a time.

| Reclaim policy | When the PVC is released |
| --- | --- |
| `Delete` (**default** for dynamic PVs) | PV **and the external volume are deleted** — dangerous |
| `Retain` | PV and external storage are kept; clean up manually — safer |

| Volume binding mode | Behaviour |
| --- | --- |
| `Immediate` | PV and backend volume created as soon as the PVC exists |
| `WaitForFirstConsumer` | **Topology-aware provisioning** (provision on demand): waits for a Pod to use the PVC, so the volume lands in the Pod's region/zone |

### Hands-on 1: use an existing StorageClass (LKE)

- `kubectl get sc` → `linode-block-storage` (Delete, Immediate) and `linode-block-storage-retain` (**default**, Retain). PVCs without a class get the **default class**, which varies between clusters, so name a class explicitly for production apps. (GKE Autopilot shows 8 classes: Filestore CSI, PD CSI, and the legacy in-tree `kubernetes.io/gce-pd`.)
- `kubectl describe sc linode-block-storage` → provisioner, AllowVolumeExpansion, ReclaimPolicy Delete, VolumeBindingMode Immediate.
- Apply `lke-pvc-test.yml` (PVC `pvc-test`, RWO, 10Gi, class `linode-block-storage`) → `kubectl get pvc` shows **Bound** immediately; `kubectl get pv` shows a PV named `pvc-...`, reclaim Delete; the 10GB volume appears in the Linode console.
- `kubectl delete pvc pvc-test` → PV and backend volume disappear (Delete policy).

### Hands-on 2: create a StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-wait-keep
provisioner: linodebs.csi.linode.com
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer   # provision on demand
reclaimPolicy: Retain                     # keep volume when PVC released
```

- Apply `lke-sc-wait-keep.yml`, then PVC `pvc-wait-keep` (20Gi) → PVC stays **Pending**, no PV yet.
- Deploy Pod `volpod` (`lke-app.yml`) that mounts the claim:

```yaml
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-wait-keep
  containers:
  - name: ubuntu-ctr
    image: ubuntu:latest
    command: ["/bin/bash", "-c", "sleep 60m"]
    volumeMounts:
    - name: data
      mountPath: /tkb
```

- Now PVC is **Bound**, a 20Gi PV exists with reclaim **Retain**; `kubectl describe pod volpod` shows `/tkb from data (rw)`.
- **Clean-up order with Retain:** delete Pod → delete PVC → delete PV manually (`kubectl delete pv <name>`) → delete the backend volume in the cloud console (or keep paying) → delete the SC.

### Deeper dive

**The book's AWS workflow in seven steps** (a Pod needing 50GB):

1. The Pod requests the volume through a **PVC**.
2. The PVC asks its **StorageClass** for a new PV and backend volume.
3. The SC calls the AWS backend through the **AWS CSI plugin**.
4. The plugin creates a **50GB EBS volume** on AWS.
5. The plugin reports the new volume back to the SC.
6. The SC creates a **PV** and maps it to the EBS volume.
7. The Pod **mounts** the PV and uses it.

**Storage tiers as StorageClasses** (the book's example for one external array):

| External tier | StorageClass | CSI plugin |
| --- | --- | --- |
| Flash | `sc-fast-block` | `csi.xyz.block` |
| Flash, encrypted | `sc-fast-encrypted` | `csi.xyz.block` |
| Mechanical disk | `sc-slow` | `csi.xyz.block` |
| NFS filestore | `sc-file` | `csi.xyz.file` |

An app needing 100GB of fast encrypted block just names `sc-fast-encrypted` in its PVC; the SC controller, plugin and array do the rest.

**What each `kubectl get sc` column means:**

| Column | Meaning |
| --- | --- |
| PROVISIONER | CSI plugin that creates volumes (e.g. `linodebs.csi.linode.com`, `pd.csi.storage.gke.io`, legacy in-tree `kubernetes.io/gce-pd`) |
| RECLAIMPOLICY | `Delete` or `Retain` when the PVC goes away |
| VOLUMEBINDINGMODE | `Immediate` or `WaitForFirstConsumer` |
| ALLOWVOLUMEEXPANSION | Whether PVCs from this class can be grown later |
| `(default)` after the name | The class used by PVCs that don't name one |

**Immediate vs WaitForFirstConsumer, as you saw it:**

|  | `linode-block-storage` (Immediate) | `block-wait-keep` (WaitForFirstConsumer) |
| --- | --- | --- |
| After `apply` of the PVC | Bound at once; PV and cloud volume exist | **Pending**; no PV yet |
| After the Pod is created | (already there) | Volume created in the Pod's zone, PVC Bound |
| After deleting the PVC | PV and cloud volume deleted (Delete) | PV and cloud volume kept (Retain) |

**Going further (not in the book):**

- **PV status lifecycle:** `Available` (unclaimed) → `Bound` (claimed by a PVC) → `Released` (PVC deleted, Retain policy, data still there) → you delete or clean it manually. A Released PV isn't automatically reused by a new PVC.
- **Expanding a volume:** if the class has `allowVolumeExpansion: true`, edit the PVC's `resources.requests.storage` to a larger value; the CSI driver grows the volume. Shrinking isn't supported.
- **Static provisioning:** instead of a StorageClass, an admin can create a PV by hand pointing at an existing volume, and a PVC binds to it if size, access mode and class match. Dynamic provisioning is the norm, but this is how you attach pre-existing data.
- **Ephemeral vs persistent:** an `emptyDir` volume (used for the shared `html` volume in Ch 4) lives and dies with its Pod; only PVs outlive Pods.

### Takeaways

- PV = external volume in Kubernetes, PVC = a claim on it, SC = dynamic provisioning of both via a CSI plugin.
- Default reclaim for dynamic volumes is **Delete**: deleting a PVC can delete your data. Use Retain for anything precious.
- `WaitForFirstConsumer` keeps volumes in the same zone as the Pods that use them.

## Chapter 12 — ConfigMaps and Secrets

ConfigMaps and Secrets decouple apps from their configuration: build one image, store config separately, and inject it at run time. ConfigMaps hold non-sensitive data; Secrets hold sensitive data but are **not secure by default** (base64, not encrypted).

### Why decouple config from code

Bundling app + config is an **anti-pattern** (seems good, turns out bad). With dev, test and prod each needing their own config, you'd build, store and redeploy **three images** for every change, even a typo fix.

Decoupled: one hardened base image (e.g. NGINX with no embedded config), one repository everyone can pull (no secrets in it), and per-environment configs applied at run time. Benefits: **reuse**, simpler dev and test, smaller less-disruptive changes (fix a typo in config and push it to running containers without replacing them).

### ConfigMap theory

- `ConfigMap` (CM) is in the **core `v1` API group**: stable, long-established, YAML-definable, kubectl-manageable.
- Use for: environment variables, config files (web server, database), hostnames, Service names/ports, account names. **Never** for certificates or passwords.
- A map of **key-value pairs**: keys = alphanumerics, `-`, `.`, `_`; values = anything, including whole multi-line files; max **1 MiB**.
- ConfigMaps have a **`data`** block instead of `spec`/`status`: there is no desired vs observed state.

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: epl
data:
  Competition: Premier League
  Season: 2024-2025
  Champions: Liverpool
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: test-config
data:
  test.conf: |            # key; the pipe makes everything below one value
    env = plex-test
    endpoint = 0.0.0.0:31001
    char = utf8
    vault = PLEX/test
    log-size = 512M
```

- **Kubernetes-native apps** can read ConfigMaps straight from the API server and see live updates, but that creates Kubernetes lock-in. The book's examples use env vars and volumes so ordinary apps work unchanged.

### Injecting config into containers

| Method | How | Sees updates? |
| --- | --- | --- |
| Environment variables | `env[].valueFrom.configMapKeyRef` (name + key) | No — static, set at container start |
| Startup command arguments | Env vars referenced in `command`, e.g. `$(FIRSTNAME)` | No — same limitation |
| Volume (**preferred**) | `volumes[].configMap` + `volumeMounts`; each key becomes a file | **Yes** — updates appear in a minute or two |

**Environment variables** (`podenv.yml`):

```yaml
spec:
  containers:
  - name: ctr1
    env:
    - name: FIRSTNAME
      valueFrom:
        configMapKeyRef:
          name: multimap
          key: firstname
    - name: LASTNAME
      valueFrom:
        configMapKeyRef:
          name: multimap
          key: lastname
```

`kubectl exec envpod -- env | grep NAME` → `FIRSTNAME=Nigel`, `LASTNAME=Poulton`.

**Startup command** (`podstartup.yml`): same `env` block plus `command: ["/bin/sh", "-c", "echo First name $(FIRSTNAME) last name $(LASTNAME)"]`. `kubectl logs startup-pod -c args1` prints the line; the Pod then shows Completed.

**Volume** (`podvol.yml`):

```yaml
spec:
  volumes:
  - name: volmap
    configMap:
      name: multimap
  containers:
  - name: ctr
    image: nginx
    volumeMounts:
    - name: volmap
      mountPath: /etc/name
```

`kubectl exec cmvol -- ls /etc/name` → `firstname`, `lastname`. After `kubectl edit cm multimap` changes the data to `City`/`Country`, the files update in the live container.

### ConfigMap commands

```bash
kubectl create configmap testmap1 \
  --from-literal shortname=SAFC \
  --from-literal longname="Sunderland Association Football Club"
kubectl create cm testmap2 --from-file cmfile.txt   # key = file name, value = contents
kubectl get cm
kubectl describe cm testmap1
kubectl get cm testmap2 -o yaml
kubectl apply -f fullname.yml                       # declarative
kubectl edit cm multimap
```

Don't delete the system ConfigMap `kube-root-ca.crt`.

### Secrets: are they secure?

**No, not on their own.** A real secrets solution must cover:

- Encryption **at rest** in the cluster store (configure `EncryptionConfiguration`)
- Encryption **in flight** (e.g. a service mesh)
- Protection when surfaced on nodes, Pods and containers
- **Least-privilege RBAC** on API access to Secrets
- Restricted access to etcd / control plane nodes
- Keeping privileged containers away from Secrets
- Never committing Secret manifests to GitHub or similar
- Secure deletion when no longer needed

Most new clusters do **none** of this: Secrets are stored unencrypted, sent in plain text and mounted as plain text. Production setups often use a service mesh plus an external vault (**HashiCorp Vault** or a cloud equivalent) via the **Secrets Store CSI Driver**.

**Default Secret workflow:** create Secret (stored unencrypted) → schedule a Pod using it → sent unencrypted to the node → kubelet starts the Pod → runtime mounts it via an in-memory **tmpfs** and decodes base64 to plain text → app reads it → deleting the Pod deletes the node's copy (the cluster store keeps its copy).

Use case: a TLS termination proxy image with a different certificate Secret per environment.

### Secrets hands-on

```bash
kubectl create secret generic creds \
  --from-literal user=nigelpoulton \
  --from-literal pwd=Password123
kubectl get secret creds -o yaml          # values are base64
echo UGFzc3dvcmQxMjM= | base64 -d         # Password123 -- no key needed
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tkb-secret
  labels:
    chapter: configmaps
type: Opaque
data:                     # use stringData for plain-text values
  username: bmlnZWxwb3VsdG9u
  password: UGFzc3dvcmQxMjM=
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  volumes:
  - name: secret-vol
    secret:
      secretName: tkb-secret
  containers:
  - name: secret-ctr
    image: nginx
    volumeMounts:
    - name: secret-vol
      mountPath: "/etc/tkb"
```

- Secret volumes are mounted **read-only** automatically. `ls /etc/tkb` → `password`, `username`; `cat` shows **plain text**.
- Like ConfigMaps, Secrets can be injected as env vars, command args or volumes; volumes are most flexible and receive updates.

### Deeper dive

**The dev/test/prod problem in numbers.** Coupled: 1 app × 3 environments = **3 images**, 3 repositories, 3 builds, tests and deployments per change, and every repo holds secrets so access must be tight. Decoupled: **1 image**, 1 repository everyone can pull, 3 small configs applied at run time; a typo fix is a config change pushed to running containers.

**ConfigMap data rules at a glance:**

| Rule | Detail |
| --- | --- |
| Keys | Alphanumerics, `-`, `.`, `_` (e.g. `test.conf`, `firstname`) |
| Values | Any string, including whole multi-line files (use `\|` in YAML) |
| Size | 1 MiB per ConfigMap |
| Structure | `data` (and `binaryData`) instead of `spec`/`status` |
| `--from-literal k=v` | One entry per flag |
| `--from-file f` | Key = file name, value = file contents |

**How each injection method looks inside the container:**

| Method | Inside the container | Update behaviour |
| --- | --- | --- |
| Env vars | `FIRSTNAME=Nigel` in the environment (`env \| grep NAME`) | Frozen at container start |
| Startup command | `$(FIRSTNAME)` substituted into `command` before it runs | Frozen at container start |
| Volume | A file per key under the mount path (`/etc/name/firstname`) | Files refresh after the CM changes, typically within a minute or two |

**The edit-and-watch experiment explained.** `kubectl edit cm multimap` replaced the keys `firstname`/`lastname` with `City`/`Country`. The `cmvol` container's `/etc/name` directory changed to match (old files gone, new ones appear), without restarting the Pod. The `envpod` container still shows the old `FIRSTNAME` and `LASTNAME`, because env vars never update.

**Secrets on the node, step by step:**

1. Created → stored **unencrypted** in etcd (unless you configure encryption at rest).
2. A Pod using it is scheduled → sent to that node's kubelet (plain text unless traffic is encrypted).
3. The runtime mounts it into the container through an in-memory **tmpfs**, never written to the node's disk, **read-only**, base64-decoded to plain text.
4. Pod deleted → the node's copy is removed; etcd keeps its copy.

**Base64 is not encryption.** `echo UGFzc3dvcmQxMjM= | base64 -d` → `Password123`, with no key. Anyone who can read the YAML or `kubectl get secret -o yaml` can read the secret, so RBAC on Secrets matters as much as encryption.

**Going further (not in the book):**

- **Load every key at once** as env vars with `envFrom` instead of one `configMapKeyRef` per key:

```yaml
containers:
- name: ctr1
  image: busybox
  envFrom:
  - configMapRef:
      name: multimap        # creates firstname and lastname env vars
  - secretRef:
      name: tkb-secret
```

- **Immutable ConfigMaps/Secrets:** setting `immutable: true` blocks changes (you replace the object instead), protects against accidental edits, and reduces API server load on big clusters.
- **`subPath` gotcha:** mounting a single key with `subPath` gives you a file that **does not** receive updates, unlike a full ConfigMap volume.
- **Secret types:** besides `Opaque`, there are built-in types such as `kubernetes.io/tls` (cert + key, used by Ingress TLS) and `kubernetes.io/dockerconfigjson` (registry credentials for pulling private images).

### Takeaways

- Separate config from images; inject with ConfigMaps (non-sensitive) and Secrets (sensitive).
- Prefer volume injection: it's the only method that pushes updates to running containers.
- Base64 is encoding, not encryption. Secure Secrets with encryption at rest, mTLS, RBAC, etcd lockdown, or an external vault.

## Chapter 13 — StatefulSets

StatefulSets run **stateful** apps (databases, key-value stores, apps saving session data). Like Deployments they self-heal, scale and roll out, but each Pod gets a **sticky identity**: a predictable name, DNS name and its own volume that survive failures, rescheduling and scaling.

### StatefulSets vs Deployments

|  | Deployment | StatefulSet |
| --- | --- | --- |
| Pod names | Random (`hello-deploy-54f5d46964-x7k2p`) | **Predictable**: `<sts-name>-<ordinal>` from 0 (`tkb-sts-0`, `-1`, `-2`) |
| Volumes | Shared/none per Pod | **Per-Pod PVC** created from `volumeClaimTemplates`, reattached to the same-named Pod |
| Startup / scale-up | All Pods at once (race conditions possible) | **One at a time**, each running and ready before the next |
| Scale-down | Any Pods, in parallel | **Highest ordinal first**, waiting for full termination |
| Self-healing & scaling done by | ReplicaSet controller | StatefulSet controller itself |
| Networking | Regular Service (load-balanced ClusterIP) | **Headless Service** giving each Pod its own DNS record |
| Rollout order | Batches per maxSurge/maxUnavailable | Highest ordinal down, one at a time, each ready first |

- Name + volume binding = the Pod's **sticky ID**. A failed `tkb-sts-3` is replaced by a new `tkb-sts-3` with the same DNS name and volume, even on another node; same after scale-down then scale-up.
- Ordered, never-parallel termination matters: clustered apps can lose data if several replicas stop at once.
- Use valid DNS characters in the StatefulSet name; it's the base for Pod and DNS names.

### Volumes

- The controller creates each Pod's volumes with names linking them to the Pod: **`<claim-template>-<sts-name>-<ordinal>`** (e.g. `webroot-tkb-sts-0`).
- Volumes stay **decoupled** via normal PVCs with their own lifecycle: Pod failure, termination or scale-down **does not delete PVCs**, so replacements reconnect to the surviving data. A lifesaver if you delete the last replica by mistake.

### Failures

- **Pod failure:** controller recreates the same-named Pod and attaches the same PVC.
- **Node failure** is trickier: if a "failed" node was only rebooting, you could end up with two identical Pods writing to one volume (corruption). Older Kubernetes needed manual intervention; newer versions handle this much better.

### Headless and governing Services

- A **headless Service** is a normal Service with **`clusterIP: None`** (no "head", just the tail of Pods).
- Listing it in the StatefulSet's **`spec.serviceName`** makes it the **governing Service**: it owns the StatefulSet's DNS subdomain and creates **DNS SRV and A records for every Pod**.
- Clients query DNS for the list of Pods and connect **directly** to a specific Pod. Developers must code apps to do this.
- Pod DNS name format: **`<pod-name>.<service-name>.<namespace>.svc.cluster.local`**, e.g. `tkb-sts-0.dullahan.default.svc.cluster.local`.

### Deleting a StatefulSet

- Deleting it does **not** terminate Pods in order, so **scale to 0 first**: `kubectl scale sts tkb-sts --replicas=0`, then `kubectl delete sts tkb-sts`.
- Set `terminationGracePeriodSeconds` (commonly ≥ 10) so apps can flush buffers and commit in-flight writes.

### Hands-on

Three objects (can be combined in `app.yml`): StorageClass, headless Service, StatefulSet.

```yaml
# lke-sc.yml (Docker Desktop: dd-kind-sc.yml; other clouds: your own class named block)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block
provisioner: linodebs.csi.linode.com
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
---
# headless-svc.yml
apiVersion: v1
kind: Service
metadata:
  name: dullahan
  labels:
    app: web
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None          # headless
  selector:
    app: web
---
# sts.yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tkb-sts
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  serviceName: "dullahan"  # governing Service
  template:
    metadata:
      labels:
        app: web
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: ctr-web
        image: nginx:latest
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: webroot
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:    # one PVC per Pod
  - metadata:
      name: webroot
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "block"
      resources:
        requests:
          storage: 10Gi
```

- `kubectl get svc` shows `dullahan` with CLUSTER-IP **None**.
- `kubectl get sts --watch` climbs 0/3 → 1/3 → 2/3 → 3/3 about **30 s apart** (ordered startup).
- `kubectl get pvc` → `webroot-tkb-sts-0/1/2`, each Bound, 10Gi RWO.

**Peer discovery:** from a jump Pod with `dig`: `dig SRV dullahan.default.svc.cluster.local` → ANSWER section lists the three Pod DNS names, ADDITIONAL section maps each to its IP.

**Scaling:** set replicas 3 → 2 and apply: `tkb-sts-2` (highest ordinal) is deleted but **all 3 PVCs remain** (`webroot-tkb-sts-2` shows Used by `<none>`). Back to 3: a new `tkb-sts-2` mounts `webroot-tkb-sts-2` (`kubectl describe pod tkb-sts-2 | grep ClaimName`).

- Scale-down is **paused if any Pod is failed**, protecting data.
- **`spec.podManagementPolicy`**: `OrderedReady` (default) or `Parallel` (create/delete together, like a Deployment). Naming rules still apply; it affects scaling only, not rollouts.
- **Rollouts:** change the image and apply; replaced from the highest ordinal down, one at a time, each ready first. See `kubectl explain sts.spec.updateStrategy`. **Rollbacks need manual attention.**

**Failure tests:**

- `kubectl delete pod tkb-sts-0` → Terminating → Pending → ContainerCreating → Running with the same name and the same PVC `webroot-tkb-sts-0`.
- Node failure (recycle an LKE node, or delete a kind node container in Docker Desktop): the Pod terminates, is rescheduled to a surviving node, may sit in ContainerCreating while the old PVC attachment is released, then runs. Kubernetes does **not** rebalance existing replicas onto a replacement node.

**Clean up:** scale to 0, delete the StatefulSet, jump Pod, headless Service, the three PVCs (deletes PVs and cloud volumes via the Delete policy; check your backend), and the StorageClass.

### Deeper dive

**Ordering, worked through** for `tkb-sts` with the default `OrderedReady` policy:

| Operation | What the controller does |
| --- | --- |
| Create, 3 replicas | Start `-0`, wait until running and ready → start `-1`, wait → start `-2` (≈30 s apart in the book) |
| Scale 3 → 5 | Start `-3`, wait until ready → start `-4`; new PVCs `webroot-tkb-sts-3` and `-4` are created |
| Scale 5 → 2 | Terminate `-4`, wait until fully gone → `-3` → `-2`; **all 5 PVCs remain** |
| Scale 2 → 3 | Start a new `-2`, which reattaches to the existing `webroot-tkb-sts-2` (its data is still there) |
| Rolling update | Replace `-2` → wait until ready → `-1` → `-0` (highest ordinal first) |
| Any Pod Failed | Scale-downs pause until it recovers |
| Delete the StatefulSet | **No ordering guarantee** → scale to 0 first |

With `podManagementPolicy: Parallel`, the create and scale rows happen all at once (like a Deployment); names and volume bindings stay the same, and rollouts are unaffected.

**Reading the `dig SRV` output:**

- **QUESTION:** `dullahan.default.svc.cluster.local. IN SRV` — "which Pods sit behind this headless Service?"
- **ANSWER:** one SRV record per Pod, each pointing at a Pod's DNS name (`tkb-sts-0.dullahan.default.svc.cluster.local.` ...).
- **ADDITIONAL:** an A record per Pod name with its current IP (`10.60.0.5` ...).

A client (say, a database replica looking for its peers) queries once, gets every member, then connects to specific Pods by stable name. If `tkb-sts-0` is rescheduled and gets a new IP, its **name stays the same** and DNS is updated.

**Regular vs headless Service DNS:**

|  | Regular (ClusterIP) Service | Headless Service (`clusterIP: None`) |
| --- | --- | --- |
| `nslookup <svc>` returns | One virtual ClusterIP | The IP of every matching Pod |
| Load balancing | kube-proxy spreads traffic | None: the client picks a Pod |
| Per-Pod DNS names | No | Yes, for StatefulSet Pods: `<pod>.<svc>.<ns>.svc.cluster.local` |

**Why the node-failure test can hang in ContainerCreating.** The replacement `tkb-sts-0` lands on a new node, but its `ReadWriteOnce` volume is still attached to the old (dead) node. Kubernetes must first detach it, which takes time when the old node can't respond. This caution is deliberate: attaching the same volume to two writers would corrupt data.

**Going further (not in the book):**

- **Update strategies** (`kubectl explain sts.spec.updateStrategy`): `RollingUpdate` (default; highest ordinal first) and `OnDelete` (Pods are only replaced when you delete them yourself, for manual, controlled upgrades). `RollingUpdate` also takes a **`partition`**: only ordinals ≥ the partition are updated, which lets you canary a new version on, say, `-2` while `-0` and `-1` stay on the old one.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2     # only tkb-sts-2 (and higher) get the new template
```

- **Automatic PVC clean-up:** `spec.persistentVolumeClaimRetentionPolicy` (`whenDeleted` / `whenScaled`, each `Retain` or `Delete`) lets you choose whether PVCs are removed when the StatefulSet is deleted or scaled down. The default `Retain` matches the behaviour shown in the book.
- **Operators** for databases (PostgreSQL, MongoDB, Kafka...) build on StatefulSets and add the app-specific logic — leader election, backups, failover — that StatefulSets leave to you.

### Takeaways

- StatefulSets give each Pod a stable name, DNS identity and volume, plus ordered start, scale and stop.
- Pair them with a headless governing Service for per-Pod DNS records; apps must be written to use them.
- PVCs outlive Pods; always scale to zero before deleting a StatefulSet. StatefulSets are a framework: apps must be designed to exploit it.

## Chapter 14 — API security and RBAC

Every request to the API server, from kubectl, Pods, kubelets, control plane services or Kubernetes-native apps, travels over TLS and must pass **authentication**, then **authorization**, then **admission control** before it runs.

&#91;embedded content: API request path · authentication, authorization, admission control\]

Any gate can stop the request; read requests skip admission control entirely.

The book's airport analogy: passport check (authN) → do you have a ticket for this flight (authZ) → airline policies such as hand-luggage and no-weapons rules (admission) → take your seat. Example: user `grant-ward` runs `kubectl apply` to create Deployment `hive` in Namespace `terran`.

### Authentication (authN, "auth en")

- Proves identity. Every request carries credentials; failure returns **HTTP 401**.
- **Pluggable:** client certificates, webhooks, external identity systems (Active Directory, cloud IAM).
- Kubernetes has **no built-in identity database** — deliberately, to avoid another identity silo. Most clusters support client certs out of the box; production should integrate with corporate or cloud IAM (hosted services often do this automatically).
- Credentials live in the **kubeconfig** (`~/.kube/config`): `clusters` (name, API server URL, CA cert), `users` (name + token, often an X.509 cert signed by the cluster CA), `contexts` (cluster + user \[+ namespace\]), `current-context`.

### Authorization (authZ, "auth zee") with RBAC

- Pluggable, multiple modules allowed; the first module that authorizes a request ends authZ. Most clusters use **RBAC** (Role-Based Access Control).
- RBAC = **which users** can perform **which actions** on **which resources** (e.g. Bao can create Pods; Kalila can list Deployments).
- **Least-privilege, deny-by-default.** There are **only allow rules, no deny rules**, which keeps it simple to reason about and troubleshoot.

**Roles define permissions; RoleBindings grant them to users:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: shield
  name: read-deployments
rules:
- verbs: ["get", "watch", "list"]   # allowed actions
  apiGroups: ["apps"]               # on resources
  resources: ["deployments"]        # of this type
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-deployments
  namespace: shield
subjects:
- kind: User
  name: sky                           # must match an authenticated user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: read-deployments
  apiGroup: rbac.authorization.k8s.io
```

Now `sky` can run `kubectl get deployments -n shield`.

**Rules = verbs + apiGroups + resources:**

| apiGroup | Resource | API path |
| --- | --- | --- |
| `""` (core group) | pods | `/api/v1/namespaces/{namespace}/pods` |
| `""` | secrets | `/api/v1/namespaces/{namespace}/secrets` |
| `"storage.k8s.io"` | storageclasses | `/apis/storage.k8s.io/v1/storageclasses` |
| `"apps"` | deployments | `/apis/apps/v1/namespaces/{namespace}/deployments` |

| Kubernetes verb | HTTP method | Common responses |
| --- | --- | --- |
| create | POST | 201 Created, 403 |
| get, list, watch | GET | 200 OK, 403 |
| update | PUT | 200 OK, 403 |
| patch | PATCH | 200 OK, 403 |
| delete | DELETE | 200 OK, 403 |

- `kubectl api-resources --sort-by name -o wide` lists every resource with its API group and supported verbs, handy when writing rules.
- `"*"` means all verbs / resources / API groups (demonstration only; avoid in practice).

**Four RBAC objects** (all in `rbac.authorization.k8s.io`, near-identical YAML):

| Object | Scope |
| --- | --- |
| Role | One Namespace |
| RoleBinding | One Namespace |
| ClusterRole | Cluster-wide (no `metadata.namespace`) |
| ClusterRoleBinding | Cluster-wide |

**Powerful pattern:** define a **ClusterRole once**, then apply it to specific Namespaces with one **RoleBinding per Namespace**.

### Real-world example: Docker Desktop admin access

1. Docker Desktop puts a **client certificate** (signed by the cluster CA) in your kubeconfig under the friendly name `docker-desktop`.
2. Decode it: `kubectl config view --raw -o json | jq '.users[] | select(.name=="docker-desktop")' | jq -r '.user["client-certificate-data"]' | base64 -d | openssl x509 -text | grep "Subject:"` → `O=kubeadm:cluster-admins, CN=kubernetes-admin`. Certificates encode the **username in CN** and **groups in O**.
3. `kubectl describe clusterrole cluster-admin` → all verbs on all resources (`*.*`): the equivalent of **root**.
4. `kubectl get clusterrolebindings | grep cluster-admin` and `kubectl describe clusterrolebindings kubeadm:cluster-admins` → binds the **group** `kubeadm:cluster-admins` to ClusterRole `cluster-admin`.
5. Result: you authenticate as `kubernetes-admin`, a member of that group, so you're a full cluster admin.

### Admission control

- Runs after authN and authZ; it's all about **policy**. Applies only to requests that **modify** the cluster; reads skip it.
- **Mutating** controllers check and can **modify** requests; they always run **first**. **Validating** controllers check and can only **accept or reject**.
- Example policy "every object needs `env=prod`": a mutating controller adds the label if missing; a validating controller rejects the request.
- Docker Desktop's API server: `kubectl describe pod kube-apiserver-desktop-control-plane -n kube-system | grep admission` → `--enable-admission-plugins=NodeRestriction`. Real clusters run many more, e.g. **AlwaysPullImages** sets every Pod's `imagePullPolicy: Always` (no cached images; nodes need registry credentials).
- **One rejection = request denied** immediately; all controllers must approve.

### Deeper dive

**The `grant-ward` example, gate by gate:**

| Gate | Question asked | Pass | Fail |
| --- | --- | --- | --- |
| TLS | Is the connection secure? | Encrypted channel to the API server | Connection refused |
| Authentication | Is this really `grant-ward`? | Identity confirmed (cert, IAM token...) | **401 Unauthorized** |
| Authorization (RBAC) | May `grant-ward` **create** **Deployments** in **`terran`**? | A Role/ClusterRole bound to them allows it | **403 Forbidden** |
| Mutating admission | Does the object need policy changes? | Object adjusted (e.g. label added, `imagePullPolicy: Always`) | Rejected |
| Validating admission | Does the final object meet policy? | Accepted | Rejected with a reason |
| Execute | — | `hive` Deployment stored in etcd | — |

**How a rule reads:** `verbs` (what) + `apiGroups` and `resources` (on what) + the Role's Namespace (where). The binding adds **who**. So the `read-deployments` Role + RoleBinding reads: *"sky may get, watch and list Deployments in the `shield` Namespace."* Nothing else is allowed, because RBAC has no deny rules: anything not explicitly allowed is denied.

**The reuse pattern in full.** One ClusterRole defined once, granted in two Namespaces by two RoleBindings. Each binding gives the cluster-wide role's permissions only **inside its own Namespace**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-deployments
rules:
- verbs: ["get", "watch", "list"]
  apiGroups: ["apps"]
  resources: ["deployments"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-deployments
  namespace: dev
subjects:
- kind: User
  name: sky
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole            # a RoleBinding can reference a ClusterRole
  name: read-deployments
  apiGroup: rbac.authorization.k8s.io
---
# ...same RoleBinding again with namespace: prod
```

A **ClusterRoleBinding** to the same ClusterRole would instead grant it in **every** Namespace.

**Which binding to use:**

| Want | Use |
| --- | --- |
| Permissions in one Namespace, defined once for that Namespace | Role + RoleBinding |
| The same permissions in several chosen Namespaces | ClusterRole + one RoleBinding per Namespace |
| Permissions in all Namespaces, or on cluster-scoped resources (Nodes, PVs, StorageClasses) | ClusterRole + ClusterRoleBinding |

**The Docker Desktop admin chain, as one line:** client cert `CN=kubernetes-admin, O=kubeadm:cluster-admins` → signed by the cluster CA (authN passes) → ClusterRoleBinding `kubeadm:cluster-admins` binds that **group** → ClusterRole `cluster-admin` (`*.*`, all verbs) → you're root on the cluster. Binding groups rather than individual users is the norm: add someone to the group and they inherit the access.

**Going further (not in the book):**

- **Subject kinds:** `User` and `Group` (identities from your authenticator) and **`ServiceAccount`** (identities for Pods, in a Namespace):

```yaml
subjects:
- kind: ServiceAccount
  name: build-bot
  namespace: ci
```

- **Test permissions without guessing** using `kubectl auth can-i`:

```bash
kubectl auth can-i create deployments -n terran
kubectl auth can-i list deployments -n shield --as sky     # impersonate a user (needs rights to impersonate)
kubectl auth can-i --list -n shield                         # everything you may do there
```

- **Built-in ClusterRoles** worth knowing: `cluster-admin` (everything), `admin` and `edit` (broad rights, meant to be bound per Namespace), and `view` (read-only, excluding Secrets).

### Takeaways

- Request path: TLS → authN (401) → authZ (403) → mutating admission → validating admission → execute.
- Kubernetes outsources identity; use enterprise IAM in production.
- RBAC is allow-only and deny-by-default: Roles/ClusterRoles define permissions, RoleBindings/ClusterRoleBindings grant them. Reuse ClusterRoles via per-Namespace RoleBindings.

## Chapter 15 — The Kubernetes API

Kubernetes is **API-centric**: every resource is defined in the API and every create, read, update and delete goes through the API server as a RESTful HTTPS request. The API is split into the original **core group** (`/api/v1`) and **named groups** (`/apis/<group>/<version>`), and you can extend it with **CustomResourceDefinitions**.

### Big picture and jargon

- Clients (kubectl, code, API tools like Postman) send requests to the API server → authN/authZ → for creates, the object is persisted to the cluster store **serialized** and scheduled.
- **Resource vs object:** the API definition is a resource; running instances are objects ("the Pod resource is in the core group; five Pod objects run in default"). Used interchangeably.
- **Serialization** = converting an object to a string/byte stream for the network or storage (deserialization = reverse). Kubernetes supports **JSON** (used with external clients) and **Protobuf** (faster, scales better, harder to inspect; used between internal components). Clients state what they accept via headers such as `Content-Type: application/json`.
- **Amazon analogy:** stuff = resources, warehouse = API, website = API server, browser = kubectl. Third parties can sell through Amazon just as CRDs add resources. If it isn't in the API, you can't deploy it.

### The API server

- Exposes the API over **RESTful HTTPS**, usually port **443 or 6443** (`kubectl cluster-info`).
- "Grand Central station": kubectl commands, kubelets (watch for tasks, report status) and control plane services all talk through it.
- Often runs as Pods in `kube-system`. Self-managed clusters need an HA, performant control plane; hosted Kubernetes hides and manages this.
- TLS for all connections; internal and external requests face the **same authN and authZ**.
- **REST** (REpresentational State Transfer) request = **verb + path**. kubectl turns `kubectl get pods --namespace shield` into `GET /api/v1/namespaces/shield/pods`.

| CRUD verb | HTTP method | kubectl example |
| --- | --- | --- |
| create | POST | `kubectl create -f <file>` |
| get, list, watch | GET | `kubectl get pods` |
| update | PUT / PATCH | `kubectl edit deployment <name>` (uses PATCH) |
| delete | DELETE | `kubectl delete ingress <name>` |

Verb names, HTTP methods and kubectl subcommands don't always match.

### Hands-on: talking REST directly

```bash
kubectl proxy --port 9000 &        # exposes the API on localhost and handles auth
curl -X GET http://localhost:9000/api/v1/namespaces/shield/pods   # PodList, items: []
curl -X GET http://localhost:9000/api/v1/namespaces
curl -v -X GET http://localhost:9000/api/v1/namespaces/shield/pods
#  > lines = sent: GET path, Accept: */*
#  < lines = received: HTTP/1.1 200 OK, Content-Type: application/json,
#    X-Kubernetes-Pf-* (priority and fairness headers)
```

**POST and DELETE a Namespace** (`ns.json`):

```json
{
  "kind": "Namespace",
  "apiVersion": "v1",
  "metadata": {
    "name": "shield",
    "labels": { "chapter": "api" }
  }
}
```

```bash
curl -X POST -H "Content-Type: application/json" \
  --data-binary @ns.json http://localhost:9000/api/v1/namespaces
kubectl get ns                                  # shield Active
curl -X DELETE -H "Content-Type: application/json" \
  http://localhost:9000/api/v1/namespaces/shield  # status.phase: Terminating
```

"POSTing a config to the API server" = creating an object. `kubectl apply -f ns.json` would send the same POST.

### API groups

The API began monolithic and was split into groups for scalability, navigation and extensibility.

| Group type | Contents | REST path |
| --- | --- | --- |
| **Core** (`""` in RBAC) | Original resources: Pods, Nodes, Services, Secrets, ServiceAccounts, Namespaces. **No new resources added.** | `/api/v1/...` (singular *api*, no group name) |
| **Named** (sub-groups) | All new resources, grouped by function: `apps` (Deployments, StatefulSets, DaemonSets), `networking.k8s.io` (Ingress, IngressClass, NetworkPolicy), `storage.k8s.io`, `rbac.authorization.k8s.io`, `batch`, `autoscaling`... | `/apis/<group>/<version>/...` (plural *apis*) |

- Namespaced objects add `/namespaces/{namespace}/`: `GET /api/v1/namespaces/shield/pods/` vs cluster-scoped `GET /api/v1/nodes/`.
- **GVR** (group, version, resource) is the shape of a path, e.g. `/apis/storage.k8s.io/v1/storageclasses`.
- Examples: Ingress `/apis/networking.k8s.io/v1/namespaces/{ns}/ingresses/`, ClusterRole `/apis/rbac.authorization.k8s.io/v1/clusterroles/`.
- If invented today, Services would likely live in `networking.k8s.io` and Pods in `apps`.

**Inspecting the API:**

```bash
kubectl api-resources          # names, short names, APIVERSION, NAMESPACED, KIND
kubectl api-versions           # enabled group/versions (spot alpha, multiple versions)
kubectl explain <kind>         # KIND and VERSION of a resource
curl http://localhost:9000/api     # core group versions: ["v1"]
curl http://localhost:9000/apis    # APIGroupList with preferredVersion per group
```

### Alpha, beta and GA

| Stage | Track | Example path | Notes |
| --- | --- | --- | --- |
| Alpha | Experimental | `/apis/apps/v1alpha1/tkb` | Buggy, may drop features, changes a lot; **disabled by default** on most clusters |
| Beta | Pre-release | `/apis/apps/v1beta1/tkb` | Close to final; minor changes on GA; **enabled by default**; use in production at your own judgement |
| GA (stable) | Production | `/apis/apps/v1/tkb` | Strong long-term commitment |

- A v2 goes through the same alpha → beta → GA cycle (`v2alpha1` → `v2beta1` → `v2`). Real stable paths: `/apis/networking.k8s.io/v1/ingresses`, `/apis/batch/v1/cronjobs`, `/apis/autoscaling/v2/horizontalpodautoscalers`.
- You can create an object through one version (e.g. `v1beta2`) and manage it later through a newer one (`v1`).

**Deprecation policy:**

- **Beta:** 9 months to ship a newer beta or graduate (prevents stagnation — Ingress sat in beta for 15+ releases).
- **GA:** deprecated only once a newer stable version exists; then still served for **12 months or 3 releases, whichever is longer**, before removal.
- Using a deprecated API triggers: (1) a CLI warning, (2) a `k8s.io/deprecated:true` audit annotation, (3) the `apiserver_requested_deprecated_apis` metric — useful when planning upgrades.

### Extending the API: CRDs

- Pattern: **custom resource + custom controller**. Vendors use it to expose features such as snapshot schedules or IP address management through standard kubectl/YAML.
- A **CustomResourceDefinition** makes a resource that looks and behaves like a native one, with its own REST path.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: books.nigelpoulton.com
spec:
  group: nigelpoulton.com      # named API group
  scope: Cluster               # or Namespaced
  names:
    plural: books
    singular: book
    kind: Book
    shortNames:
    - bk
  versions:
  - name: v1
    served: true               # false = not served on v1
    storage: true              # store objects as this version
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties: ...
---
apiVersion: nigelpoulton.com/v1
kind: Book
metadata:
  name: ai
spec:
  bookTitle: "AI Explained"
  subTitle: "Facts, Fiction, and Future"
  topic: "Artificial Intelligence"
  edition: 1
  salesUrl: https://www.amazon.com/dp/1916585388
```

- `kubectl apply -f crd.yml` → served at `apis/nigelpoulton.com/v1/books/`. Check with `kubectl api-resources | grep books`, `kubectl explain book`.
- `kubectl apply -f book.yml`, then `kubectl get bk`; `curl http://localhost:9000/apis/nigelpoulton.com/v1/` lists the resource and its verbs.
- Custom resources do nothing useful until a **custom controller** acts on them (out of scope for the book).
- Clean up: kill the proxy (`ps | grep kubectl`, `kill -9 <pid>`; Windows `tasklist` / `taskkill /F /PID`), `kubectl delete book ai`, `kubectl delete crd books.nigelpoulton.com`.

### Deeper dive

**Anatomy of a REST path**, segment by segment:

| Path | Prefix | Group | Version | Namespace part | Resource |
| --- | --- | --- | --- | --- | --- |
| `/api/v1/namespaces/shield/pods` | `/api` (core) | (none) | `v1` | `namespaces/shield` | `pods` |
| `/api/v1/nodes` | `/api` | (none) | `v1` | (cluster-scoped) | `nodes` |
| `/apis/apps/v1/namespaces/default/deployments` | `/apis` (named) | `apps` | `v1` | `namespaces/default` | `deployments` |
| `/apis/storage.k8s.io/v1/storageclasses` | `/apis` | `storage.k8s.io` | `v1` | (cluster-scoped) | `storageclasses` |
| `/apis/nigelpoulton.com/v1/books` | `/apis` | `nigelpoulton.com` (CRD) | `v1` | (cluster-scoped) | `books` |

Add `/<name>` to address one object, e.g. `/api/v1/namespaces/shield` (the target of the book's DELETE). The **apiVersion** you write in YAML is the group/version part: `v1` for core, `apps/v1`, `storage.k8s.io/v1`.

**Headers in the curl exchange:**

| Header | Direction | Meaning |
| --- | --- | --- |
| `Content-Type: application/json` | Client → server | "My request body is JSON" (needed on POST) |
| `Accept: */*` | Client → server | "I'll take any serialization" |
| `Content-Type: application/json` | Server → client | "My response is JSON" |
| `X-Kubernetes-Pf-Flowschema-Uid`, `-Prioritylevel-Uid` | Server → client | Which API Priority and Fairness rules handled the request |

**Why `kubectl proxy`.** It listens on localhost and adds your kubeconfig credentials and TLS to every request, so plain `curl http://localhost:9000/...` works without handling certificates or tokens yourself. Anything on your machine can use it while it runs, so stop it when finished.

**Resources vs objects, one more time:** "The Pod **resource** is defined in the core group; there are five Pod **objects** running in `default`." Same idea as a class vs its instances.

**CRD fields explained:**

| Field | Purpose |
| --- | --- |
| `metadata.name` | Must be `<plural>.<group>`: `books.nigelpoulton.com` |
| `spec.group` | The new named API group |
| `spec.scope` | `Namespaced` or `Cluster` |
| `names.plural / singular / kind / shortNames` | URL name, CLI name, YAML `kind`, kubectl short name (`bk`) |
| `versions[].served` | Whether this version is reachable through the API |
| `versions[].storage` | The one version objects are stored as in etcd |
| `versions[].schema.openAPIV3Schema` | Field names and types, so the API server can validate objects (e.g. `bookTitle`, `edition`) |

**Going further (not in the book):**

- **Watch kubectl's HTTP calls.** Raise the verbosity to see the REST requests behind any command:

```bash
kubectl get pods -n shield -v=6     # prints GET https://<server>/api/v1/namespaces/shield/pods ... 200 OK
kubectl get pods -n shield -v=8     # also prints request and response bodies
```

- **Raw API calls without a proxy:** `kubectl get --raw /apis/apps/v1` returns the API resource list for that group, using your kubeconfig auth directly.
- **Plan upgrades** by checking for deprecated APIs before you upgrade a cluster: the deprecation warnings, audit annotation and `apiserver_requested_deprecated_apis` metric from this chapter show which clients still use them.
- **Operators** = CRD + custom controller. The controller watches your custom objects and reconciles real resources to match them, exactly like built-in controllers do for Deployments.

### Takeaways

- Everything is an API resource reached through the API server via REST verbs and paths; keep the control plane HA and fast.
- Core group at `/api/v1`, everything newer in named groups at `/apis/<group>/<version>`.
- New APIs mature alpha → beta → GA; GA deprecations give at least 12 months / 3 releases. CRDs make Kubernetes the de facto cloud API.

## Chapter 16 — Threat modeling Kubernetes

Threat modeling identifies vulnerabilities so you can prevent or mitigate them. The chapter applies **STRIDE**'s six threat categories to Kubernetes; no model covers every threat, and one threat often leads to another, so **defense in depth** is key. It ends with **Pod Security Standards / Admission** as the preferred way to enforce Pod security.

### STRIDE at a glance

| Threat | Meaning | Main Kubernetes defences |
| --- | --- | --- |
| **S**poofing | Pretending to be someone else to gain privileges | mTLS between components, a protected CA, separate trust for internal/external clients, ServiceAccount token hygiene |
| **T**ampering | Malicious change causing DoS or privilege gain | TLS in transit; restricted access to nodes, configs, registries; checksums; audit daemons; read-only filesystems |
| **R**epudiation | Creating doubt about who did what | API server audit logs, node/runtime/app logs shipped to a secure central store |
| **I**nformation disclosure | Leaking sensitive data | Lock down etcd nodes, encrypt Secrets with **KMS v2** + external KEK (HSM/cloud KMS), keys in Secrets |
| **D**enial of service | Making something unavailable | HA control plane and etcd across zones, resource limits, ResourceQuotas, PID limits, Network Policies, firewalls |
| **E**levation of privilege | Gaining more access than granted | RBAC + Node authorizers, non-root, capabilities, seccomp, AppArmor/SELinux, `allowPrivilegeEscalation: false`, PSS/PSA |

### Spoofing

**Cluster components.** All components authenticate with signed certificates (**mTLS**), which Kubernetes rotates automatically. But:

- The auto-generated **self-signed CA** is not enough on its own for production.
- mTLS is only as secure as the CA — **protect the CA**. Approve certificate signing requests carefully and never trust the cluster CA outside the cluster.
- Use **two trusted key pairs**: the cluster CA for internal systems, trusted third-party CA(s) for external ones.

**Pods.** Every Pod has a **ServiceAccount** identity whose token is auto-mounted as a Secret and grants API server access. Most Pods don't need it:

```yaml
spec:
  serviceAccountName: some-service-account
  automountServiceAccountToken: false
```

If a Pod does need the API, limit the token's lifetime and audience with a projected volume:

```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
  serviceAccountName: my-pod
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: vault-token
          expirationSeconds: 3600   # expires after 1 hour
          audience: vault           # only valid for vault
```

### Tampering

- Hard to prevent, so make it **evident** (like tamper-proof seals on medicine).
- Targets: etcd; config files of the API server, controller manager, scheduler, etcd and kubelet; runtime binaries; container images; Kubernetes binaries.
- **In transit:** TLS gives integrity guarantees.
- **At rest:** restrict access to servers running Kubernetes (especially control plane), to config repositories and to image registries; bootstrap remotely only over SSH (guard the keys); verify downloads with **SHA-2 checksums**; audit and alert on key binaries and configs, e.g. `auditctl -w /usr/bin/docker -p wxa -k audit-docker`.
- **Apps:** make filesystems read-only. The book's example:

```yaml
spec:
  securityContext:
    readOnlyRootFilesystem: true
    allowedHostPaths:
    - pathPrefix: "/test"
      readOnly: true
```

Note: in the current Pod API, `readOnlyRootFilesystem` is set in each **container's** `securityContext`, and `allowedHostPaths` came from the old PodSecurityPolicy (removed in v1.25).

### Repudiation

- **Non-repudiation** = proving what happened, when, who, where, why and how (the last two usually need correlated events over time).
- Enable **API server auditing**. An audit event records `verb` (e.g. list), `requestURI`, `user.username` and groups, `sourceIPs`, `objectRef`, timestamps and `stage`.
- Also collect logs from container runtimes, kubelets, apps, and non-Kubernetes infrastructure such as firewalls.
- Centralise with an agent deployed as a **DaemonSet** shipping logs to a **secure** central store; an insecure store makes logs repudiable.
- File audit daemons (like the `auditctl` rule) produce SYSCALL records (e.g. a `docker run`); correlated with Kubernetes audit logs they form a trustworthy picture.

### Information disclosure

- **etcd** holds the whole cluster config (network, storage, passwords, the CA): a prime target. Limit and audit access to the nodes hosting it.
- Secret encryption arrived in v1.7 but isn't on by default, and the **data encryption key (DEK)** sits on the same node as the Secret, so node access bypasses it.
- **Envelope encryption:** a **key encryption key (KEK)** stored outside the cluster encrypts the DEKs. KMS v1 (beta in v1.11) was replaced by **KMS v2, GA since v1.29**:
  - New DEK for every write, encrypted with the KEK
  - DEK cache limited only by control plane resources (faster)
  - Zero-downtime re-encryption after key rotation (no API server downtime or Pod restarts)
  - KMS plugins run on control plane nodes, talk to the API server over Unix sockets and to an external KMS provider
- Attackers now need both the control plane and the external KMS; a node snapshot isn't enough. Keep KEKs in an **HSM** or cloud KMS; verify encryption works and plan for a lost KEK or lost KMS connectivity.
- **In Pods:** store keys and passwords in **Secrets**, not plain files or env vars; if PVs or ConfigMaps hold encrypted data, keep their keys in Secrets. Don't lock the house and leave the key in the door.

### Denial of service

**Cluster resources**

- Run multiple **HA control plane nodes across availability zones**, and multiple worker nodes across zones.
- Set **limits** on memory, CPU and storage.
- Limit **object counts** per Namespace with a ResourceQuota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-quota
  namespace: skippy
spec:
  hard:
    pods: "100"
```

- **`podPidsLimit`** caps processes per Pod, confining a **fork bomb** to one Pod and protecting the node's PID range (and the kubelet). It's a kubelet setting and needs a reasonable estimate of Pods per node.

**API server:** HA replicas across zones; monitoring and alerting on request thresholds; firewalls to limit internet exposure; guard highly privileged accounts against spoofing.

**etcd:** an HA cluster of **3 or 5** nodes; monitoring and alerting (start with etcd logs); network isolation so only the control plane can reach it. Large production clusters should run a **dedicated etcd cluster**: etcd is the most common choke point at scale, and a slow etcd is as bad as one under attack.

**Applications:** resource requests/limits, **Network Policies** for Pod-to-Pod and Pod-to-external traffic, **mTLS and API-token authentication** (reject unauthenticated requests), and least-privilege application-level authorization.

### Elevation of privilege

**Protecting the API server** with authorizers (run several together, commonly RBAC + Node):

- **RBAC** — restrict operations to subsets of users and services ("can Mia create Pods?").
- **Webhook** — offload to an external REST policy engine; extra work and a potential single point of failure for every API request, so vet it rigorously.
- **Node** — authorizes kubelet requests, which differ from user requests.

**Protecting Pods**

1. **Don't run as root (UID 0).** Container root may have unrestricted root on the host.

```yaml
spec:
  securityContext:
    runAsUser: 1000          # all containers in the Pod
  containers:
  - name: demo
    image: example.io/simple:1.0
    securityContext:
      runAsUser: 2000        # overrides for this container
```

Pods sharing a UID share a security context (risk of uncoordinated writes and cross-tampering). Mitigate with per-container UIDs, a UID usage map, or **user namespaces** (root inside the container mapped to e.g. UID 1000 on the host; check support).

2. **Drop capabilities.** Root = 30+ small **capabilities** (e.g. `SYS_TIME` sets the clock, `NET_ADMIN` changes routing/interfaces). Runtime defaults are sensible but often not enough for production. Find the minimum set by starting with all dropped, reading the failures, adding back, repeating — and retest every release.

```yaml
securityContext:
  capabilities:
    add: ["NET_ADMIN", "CHOWN"]
```

3. **Filter syscalls with seccomp** (GA in v1.19):

| Profile | Behaviour |
| --- | --- |
| Non-blocking | Allows everything, logs every syscall: use in test to build a custom profile |
| Blocking | Blocks all syscalls: secure but useless |
| Runtime default | The runtime's tested, balanced profile: a good starting point |
| Custom (preferred) | Only the syscalls the app needs; risk of missing edge cases |

4. **Mandatory access control:** **AppArmor** and **SELinux** (Linux-only), configured on nodes and applied via the Pod security context. Unlike seccomp and capabilities, they're **mandatory** once enabled; some runtimes apply a default SELinux profile. Test thoroughly, as misconfiguration breaks apps.
5. **Block privilege escalation:** Linux lets a child process claim more privileges than its parent by default.

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

### Pod Security Standards (PSS) and Pod Security Admission (PSA)

- **PSS** = policies of required Pod settings; **PSA** = enforces them when Pods are created. You pick policies, PSA enforces.

| PSS policy | Meaning |
| --- | --- |
| Privileged | Wide-open, allow-all |
| Baseline | Sensible defaults |
| Restricted | Gold standard of current best practice; many Pods fail it |

Policies are community-maintained and **can't be modified, imported or custom-built**.

- PSA works **per Namespace** as a **validating admission controller** (can't modify Pods, doesn't affect running Pods).
- Modes: **warn** (allow + user warning), **audit** (allow + audit event), **enforce** (reject).
- Label format: `pod-security.kubernetes.io/<mode>: <policy>`, e.g. `pod-security.kubernetes.io/warn: baseline`.
- Recommended path: every Namespace at least **baseline in warn/audit** → then **enforce baseline** and **warn/audit restricted**. A Namespace with no policy is a gap.

**Hands-on (`psa` folder):**

```bash
kubectl create ns psa-test
kubectl label --overwrite ns psa-test pod-security.kubernetes.io/enforce=baseline
kubectl apply -f psa-pod.yml      # privileged: true -> Forbidden: violates PodSecurity "baseline:latest"
# set privileged: false
kubectl apply -f psa-pod.yml      # pod/psa-pod created
kubectl label --dry-run=server --overwrite ns psa-test pod-security.kubernetes.io/enforce=restricted
#  warns: allowPrivilegeEscalation != false, unrestricted capabilities,
#         runAsNonRoot != true, seccompProfile
kubectl label --overwrite ns psa-test pod-security.kubernetes.io/enforce=restricted
kubectl get pods -n psa-test      # psa-pod still Running: PSA only acts on create/modify
kubectl label --overwrite ns psa-test \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

**Alternatives** when you need custom or mutating policy: **OPA Gatekeeper, Kubewarden, Kyverno**.

### Kubernetes security keeps improving

- **v1.26+:** all binaries and images used to build clusters are **cryptographically signed**.
- **v1.27+:** official auto-refreshing JSON/RSS **CVE feed**; containers can inherit the runtime's **default seccomp profile** (needs `--seccomp-default` on every kubelet).
- **Confidential computing** (confidential VMs/containers, memory encryption) available on many clouds.
- Third-party **security audit** of Kubernetes 1.24 (April 2023, following the 2019 one) and the **CNCF Cloud Native Security Whitepaper** are recommended reading.

### Deeper dive

**Threats chain together.** The chapter stresses that one threat often enables another. Examples drawn from its sections:

- **Spoofing → elevation of privilege:** steal a Pod's auto-mounted ServiceAccount token and act as that Pod against the API server.
- **Tampering → denial of service:** alter a kubelet or etcd config file so the node or cluster stops working.
- **Information disclosure → everything else:** read etcd and you have every Secret, the cluster CA and all configuration.
- **Elevation of privilege → repudiation:** an attacker with node root can bypass API audit logs unless node-level auditing ships logs off the box.

**From the PSA test to a restricted-compliant Pod.** The dry run listed four reasons `psa-pod` fails `restricted`: `allowPrivilegeEscalation != false`, unrestricted capabilities, `runAsNonRoot != true` and `seccompProfile`. Fixing all four looks like this (the fixes follow the book's warnings; the exact YAML is going further):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: psa-pod
  namespace: psa-test
spec:
  securityContext:
    runAsNonRoot: true            # fix 3
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault        # fix 4: use the runtime's seccomp profile
  containers:
  - name: psa-ctr
    image: nginxinc/nginx-unprivileged   # an NGINX image that runs as non-root
    securityContext:
      allowPrivilegeEscalation: false    # fix 1
      capabilities:
        drop: ["ALL"]                    # fix 2
      readOnlyRootFilesystem: true       # tampering defence (may need writable emptyDir mounts)
```

The stock `nginx` image runs as root, so it can't meet `runAsNonRoot` on its own. Picking images built to run as non-root is part of passing `restricted`.

**PSA modes, compared:**

| Mode | Violating Pod created? | Who notices | Good for |
| --- | --- | --- | --- |
| `warn` | Yes | The user running kubectl sees a warning | Educating teams |
| `audit` | Yes | An event in the API audit log | Gathering data quietly |
| `enforce` | **No** | Request rejected (Forbidden) | Real protection |

All three can be set on one Namespace at different levels, e.g. enforce `baseline` + warn/audit `restricted`, which is the recommended stepping stone.

**Per-threat checklist:**

- [ ] **S:** cluster CA protected and trusted only inside the cluster; separate trust for external clients; `automountServiceAccountToken: false` where the API isn't needed; short-lived, audience-bound tokens where it is
- [ ] **T:** TLS everywhere; restricted access to control plane servers, config repos and registries; SHA-2 checksums; file auditing on binaries and configs; read-only root filesystems
- [ ] **R:** API server auditing on; node, runtime and app logs shipped by a DaemonSet to a secured central store
- [ ] **I:** etcd node access limited and audited; Secrets encrypted with KMS v2 and a KEK in an HSM or cloud KMS; keys for encrypted volumes kept in Secrets
- [ ] **D:** 3 or 5 control plane and etcd nodes across zones; dedicated etcd for large clusters; resource limits, ResourceQuotas and PID limits; Network Policies; API server not exposed to the internet
- [ ] **E:** RBAC + Node authorizers; non-root UIDs; capabilities dropped; seccomp profiles; AppArmor/SELinux; `allowPrivilegeEscalation: false`; PSA enforcing at least `baseline` on every Namespace

### Takeaways

- Use STRIDE as a checklist, but expect threats to chain; layer your defences.
- Biggest wins: protect the CA and etcd, encrypt Secrets with KMS v2, audit centrally, run HA across zones, and lock down Pods (non-root, minimal capabilities, seccomp, no privilege escalation).
- Enforce Pod security per Namespace with PSA: baseline enforced, restricted warned/audited.

## Chapter 17 — Real-world Kubernetes security

A security architect's high-level view (not a cookbook) of four areas: the **software delivery pipeline**, **workload isolation**, **identity and access management**, and **monitoring and auditing**. The headline: Kubernetes does **not** support secure hard multi-tenancy, so hostile workloads need separate clusters.

### 1. Securing the software delivery pipeline

**Image registries**

- Public registries are convenient: protect what you push there and **don't trust what you pull**. **Official images** (vendor-provided, vetted, scanned, patched, sometimes supported) are safer than **community images** (unvetted: extreme caution).
- Give developers a standard, frictionless way to get images so they don't bypass it.

**Approved base images**

- Images = base layer (OS/filesystem) + libraries/dependencies + app. Maintain a **small set of approved, hardened base images** derived from official ones (e.g. hardened Alpine, Alpine+NGINX).
- Benefits: standard drivers, known patches, standard audit settings, less sprawl, simpler testing, patching and troubleshooting, fewer support contracts; developers focus on apps.
- For non-standard needs: find out why an approved image won't do, whether one can be extended (e.g. add a GPU driver) and the support implications of anything new. Usually **extend an existing base** rather than add a new one.

**Controlling access to images**

- Best: **private registries inside your firewall** (you control deployment, replication, patching, policies, AD integration). Otherwise private repos on a carefully chosen public registry.
- Host only **approved** images; RBAC on repositories (e.g. devs push/pull dev and test repos; ops push/pull prod).
- Restrict which nodes have internet access (limit to registry addresses/ports); only build nodes or automated build systems may push.

**Promoting images from non-production to production**

- **Vulnerability scanning** in CI/CD: binary-level checks against CVE databases; auto-fail builds and quarantine images with e.g. critical vulnerabilities. Prefer scanners with custom policies and false-positive marking.
- **Configuration as code review:** scan Dockerfiles, Kubernetes YAML and Helm charts too. Cautionary tale: an IBM data science experiment shipped **private TLS keys inside images**, giving attackers root on hosts. Policy-as-code tools automate this.
- **Sign images:** developers sign, consumers (the runtime) verify on pull/run, proving the image is genuine and untampered. Enforce signing policy enterprise-wide, not per user.

**Image promotion checklist:**

- [ ] Policies forcing signed images
- [ ] Network rules on which nodes can push/pull
- [ ] RBAC on image repositories
- [ ] Approved base images
- [ ] Vulnerability scanning
- [ ] Promote or quarantine based on scan results
- [ ] Review and scan infrastructure-as-code files

### 2. Workload isolation

**Cluster level**

- Namespaces are the only way to divide a cluster and only group resources for limits, quotas and RBAC. They **don't stop a compromised workload affecting other Namespaces** — use them, but not as security boundaries.
- **Soft multi-tenancy** (trusted workloads, e.g. an e-commerce front end and recommendation back end managed by different teams): one cluster, a Namespace each.
- **Hard multi-tenancy** (untrusted/hostile workloads): **not possible** today → separate clusters for prod vs non-prod, different customers, sensitive projects. The Multitenancy Working Group is working on this.

**Node isolation:** apps needing non-standard privileges (root, unusual syscalls) can run on a **ring-fenced subset of nodes** (via labels, affinity/anti-affinity, taints) with stricter auditing and runtime defences, limiting blast radius to those nodes.

**Runtime isolation** — VMs win on isolation: each has its own kernel with hardware enforcement, whereas **namespaced containers** (Docker, containerd, CRI-O) share the host kernel and rely on namespaces and cgroups, never designed as strong boundaries. Harden containers with AppArmor, SELinux, seccomp, capabilities and user namespaces.

| Isolation option | Trade-off |
| --- | --- |
| Virtual machines | Dedicated kernel, excellent isolation; slower and resource-heavy |
| Namespaced containers | Fast and light; need extra hardening |
| Every container in its own lightweight VM | VM security, but loses much of the container appeal; not popular |
| Different runtime classes (e.g. **gVisor**, **Kata Containers** via CRI + RuntimeClass) | Containers everywhere, stronger runtimes for workloads that need them |
| Wasm containers | OCI packaging only; run in a deny-by-default Wasm host (Chapter 9) |

**Network isolation**

- The pod network comes from **CNI** plugins, broadly **overlay** or **BGP**.
- **Overlay** (most common): a flat Layer-2 network on top of Layer-3 infrastructure using **VXLAN** encapsulation. Encapsulation **hides real source/destination IPs** from firewalls.
- **BGP** (the internet's routing protocol; peers share routes, like passing a birthday card via friends of friends) **doesn't encapsulate**, so it's easier for firewalls.
- If your firewalls can't inspect encapsulated packets, consider a BGP pod network. Combine **physical firewalls** (central team; beware slow change processes) and **host-based firewalls** (OS teams) for the most security.
- **Packet capture:** Pod IPs are obscured and churn (re-used by other Pods), so tools that map IPs to Pod IDs, Service names and container IDs are very useful.

### 3. Identity and access management

- Kubernetes RBAC (stable since **v1.8**) plugs into existing IAM (Active Directory, LDAP, cloud IAM) tied to HR systems: joiners get access through group membership; leavers lose it automatically.
- Admin is done via the API, so **SSH to nodes should be rare**: only for node tasks the API can't do, "break the glass" when the API server is down, and deep troubleshooting.
- Protect root-level API and node accounts with **MFA** (something you know + something you have, e.g. a one-time password). Secure workstations and profiles with kubectl installed.

### 4. Security monitoring and auditing

Assume you'll be breached. You must (1) **detect** the breach and (2) build a **non-repudiable timeline** (what, how, when, who — possibly for court). Don't go live without auditing and monitoring.

- **Baseline best practice:** the **CIS Kubernetes Benchmark**, run with Aqua Security's **kube-bench** (can't inspect hosted control planes). Run it during node provisioning (pass/fail), and keep reports to compare before vs after an incident.
- **Pod/container lifecycle events:** keep logs of terminated Pods; runtimes may log lifecycle events too.
- **Forensic checkpointing:** **CRIU** (Checkpoint/Restore in Userspace) silently captures a running container to restore in a sandbox for analysis; alpha in Kubernetes, supported only by **CRI-O** at the time of writing.
- **Application logs:** some apps log to stdout/stderr, others to bespoke files; check each app so nothing is missed.
- **User actions:** API requests are logged, but attackers with SSH to control plane nodes can bypass the API (even reach etcd). Limit SSH, require MFA, log all SSH to a secure aggregator, consider a **two-person rule** for control plane SSH.
- **Log volume:** container density creates huge log volumes; use specialised tools for storage plus proactive and reactive analysis.

**Alert immediately on:**

- **Privileged Pod creation by a human** (they can often get node root; normally only automation with service accounts should do this).
- **Exec sessions by humans** that aren't troubleshooting (consider deleting those containers to rule out tampering).
- **Control plane access attempts from the internet**; a successful one usually means a misconfiguration.

### Migrating apps: crawl, walk, run

1. **Crawl:** threat-model existing apps to know their posture (e.g. which use TLS).
2. **Walk:** migrate **without changing** security posture, so issues aren't misdiagnosed as migration vs security change.
3. **Run:** improve security after migration, starting with non-critical apps and stepping up (e.g. one-way TLS, then mutual TLS).

### Real-world example: CVE-2019-5736 (February 2019)

A container process running as **root** could gain root on the worker node and every container on it. Three practices from this chapter would each have prevented it: **image vulnerability scanning** (it had a CVE), **not running as root**, and **SELinux** default policies.

### Deeper dive

**The pipeline as one path**, from laptop to production, with the control the chapter puts at each step:

1. **Base image:** developer starts from an approved, hardened base (e.g. company Alpine or Alpine+NGINX).
2. **Build:** only build nodes / automated systems may push; config files (Dockerfile, YAML, Helm) are reviewed and scanned, catching e.g. embedded private keys.
3. **Scan:** vulnerability scanning fails the build or quarantines the image on, say, critical CVEs; known false positives are marked.
4. **Sign:** the image is cryptographically signed.
5. **Store:** pushed to a private registry with RBAC (devs → dev/test repos, ops → prod repos).
6. **Promote:** only scanned, signed images move from non-prod to prod repos.
7. **Run:** nodes pull only from allowed registry addresses; the runtime verifies signatures under an enterprise-wide policy.

**Choosing an isolation level:**

| Situation | Book's guidance |
| --- | --- |
| Trusted parts of one app, different teams (web front end + recommendations) | One cluster, a Namespace each (soft multi-tenancy) |
| Prod vs non-prod | Separate clusters |
| Different customers | Separate clusters (hard multi-tenancy isn't possible on one) |
| Sensitive projects or business functions | Separate clusters |
| A workload needing root or unusual syscalls | Ring-fenced node pool (labels, affinity, taints) with stricter audit and runtime defences |
| A workload needing a stronger sandbox but not its own cluster | Stronger runtime via RuntimeClass (gVisor, Kata Containers) |
| Event-driven or edge workloads that suit it | Wasm containers (deny-by-default) |

**Overlay vs BGP pod networks:**

|  | Overlay (VXLAN) | BGP |
| --- | --- | --- |
| How it works | Encapsulates Pod packets inside node-to-node packets: a flat Layer-2 network over Layer-3 | Shares routes between peers; packets travel unencapsulated |
| What a firewall sees | Node IPs; real Pod source/destination hidden | Real Pod IPs |
| Firewall needs | Must open and inspect packets to decide | Ordinary address-based rules work |
| Popularity | Most common default | Choose it when firewalls can't inspect encapsulated traffic |

**IAM lifecycle in practice.** Joiner: HR creates the person in AD/IAM → added to a group → a RoleBinding to that group grants Kubernetes access automatically. Leaver: HR removes them → the identity disappears → Kubernetes access stops, with no cluster-side clean-up. This is why Kubernetes deliberately has no user database of its own.

**Why alert on those three events:**

| Event | Why it's suspicious | Expected source |
| --- | --- | --- |
| Privileged Pod created by a human | Privileged Pods can often get root on the node | Automation using a ServiceAccount, rarely |
| Exec session by a human | Shell-like access inside a container; possible tampering | Troubleshooting only |
| Control plane access from the internet | The control plane should not be internet-reachable | None: a success means misconfiguration |

**Going further (not in the book): tools that implement these practices.** The book stays vendor-neutral; common open-source choices include:

| Practice | Example tools |
| --- | --- |
| Image vulnerability scanning | Trivy, Grype |
| Image signing and verification | Sigstore cosign, Notation |
| Policy as code / admission | OPA Gatekeeper, Kyverno, Kubewarden (Ch 16) |
| CIS benchmark checks | kube-bench (named in the book) |
| Runtime threat detection and alerting | Falco |
| BGP-capable CNI | Calico, Cilium |
| Sandboxed runtimes | gVisor, Kata Containers (named in the book) |

### Takeaways

- Secure the supply chain: approved base images, private registries with RBAC, scanning, config review and signed images.
- Separate clusters for hostile or high-sensitivity workloads; use node pools, stronger runtimes and network controls for the rest.
- Reuse corporate IAM with MFA, minimise SSH, and build auditing and alerting in before production.

## Quick reference

The commands and terms used most across the book, gathered in one place for revision.

### kubectl cheat sheet

| Task | Command | Ch |
| --- | --- | --- |
| Show contexts / switch | `kubectl config get-contexts` / `kubectl config use-context <ctx>` | 3 |
| Default Namespace for context | `kubectl config set-context --current --namespace <ns>` | 5 |
| Deploy / update from YAML | `kubectl apply -f <file>` | 4 |
| List with detail | `kubectl get <kind> [-o wide \| -o yaml] [--watch] [-n <ns> \| --all-namespaces]` | 4 |
| Inspect incl. events | `kubectl describe <kind> <name>` | 4 |
| Logs | `kubectl logs <pod> [--container <c>]` | 4 |
| Run command / shell | `kubectl exec <pod> -- <cmd>` / `kubectl exec -it <pod> -- sh` | 4 |
| Explore schema | `kubectl explain pods --recursive` / `kubectl explain pod.spec.restartPolicy` | 4 |
| Scale | `kubectl scale deploy <name> --replicas N` (prefer editing YAML) | 6 |
| Rollouts | `kubectl rollout status \| pause \| resume \| history \| undo --to-revision=N deploy <name>` | 6 |
| Expose as Service | `kubectl expose deployment <name> --type=LoadBalancer` | 7 |
| EndpointSlices | `kubectl get endpointslices` | 7 |
| Label a node | `kubectl label nodes <node> key=value` | 9 |
| ConfigMap / Secret | `kubectl create cm <n> --from-literal k=v --from-file f` / `kubectl create secret generic <n> --from-literal k=v` | 12 |
| Resources, groups, verbs | `kubectl api-resources [-o wide]` / `kubectl api-versions` | 5, 14, 15 |
| Raw API access | `kubectl proxy --port 9000 &` then `curl localhost:9000/api/v1/...` | 15 |
| Test PSS change | `kubectl label --dry-run=server --overwrite ns <ns> pod-security.kubernetes.io/enforce=restricted` | 16 |
| Delete | `kubectl delete <kind> <name>` / `kubectl delete -f <file>` | 4 |

### Key numbers and rules

- Control plane nodes: **3 or 5** for HA (odd numbers avoid etcd split brain); Linux only.
- kubectl within **one minor version** of the cluster.
- NodePort range **30000–32767**.
- ConfigMap max size **1 MiB**.
- Docker runtime removed in **v1.24**; native sidecars beta in v1.29 (enabled by default); KMS v2 GA in **v1.29**; seccomp GA in v1.19; Ingress GA in v1.19; RBAC stable since v1.8.
- GA API deprecations: served **12 months or 3 releases**, whichever is longer; beta APIs have **9 months** to move on.
- REST paths: core `/api/v1/...`, named groups `/apis/<group>/<version>/...`; Pod/Service DNS: `<name>.<namespace>.svc.cluster.local`; StatefulSet Pod DNS: `<pod>.<governing-svc>.<namespace>.svc.cluster.local`.

### Glossary (from the book's terminology section)

| Term | Meaning |
| --- | --- |
| Admission controller | Validates or mutates requests to enforce policy, after authN and authZ |
| Annotation | Object metadata, used for alpha/beta features or third-party integration |
| API group | Set of related resources, e.g. `networking.k8s.io` |
| Cloud controller manager | Control plane service integrating with the cloud (load balancers, storage) |
| Cloud native | Apps that self-heal, scale on demand, roll updates and roll back |
| Cluster store | etcd-based store of cluster and app state; the only stateful control plane part |
| CNI / CRI / CSI | Pluggable interfaces for networking, container runtimes and storage |
| containerd | Industry-standard runtime donated by Docker; most clusters' default |
| Controller / reconciliation loop | Process making observed state match desired state |
| CRD | Custom Resource Definition: adds your own resource types to the API |
| Data plane | The worker nodes that run user apps |
| Desired / observed state | What you asked for vs what currently exists |
| EndpointSlice | Live list of healthy Pods behind a Service (replaced Endpoints) |
| Headless Service | Service with `clusterIP: None`; creates per-Pod DNS records for StatefulSets |
| Ingress / IngressClass | Layer-7 host/path routing to many Services via one load balancer / choose among controllers |
| Init container / sidecar | Runs to completion before the app / runs alongside it (init container with `restartPolicy: Always`) |
| kubelet / kube-proxy | Node agent talking to the API server / node networking that routes Service traffic to Pods |
| Label / label selector | Grouping metadata / the matching that ties Services and controllers to Pods |
| Manifest | YAML file describing one or more objects |
| Namespace | Virtual cluster for quotas and RBAC; not a security boundary |
| PV / PVC / StorageClass | External volume in Kubernetes / a claim on it / a tier that provisions both dynamically |
| RBAC | Allow-only, deny-by-default authorization: Roles, ClusterRoles, RoleBindings, ClusterRoleBindings |
| ReplicaSet | Controller that does scaling and self-healing for Deployments |
| RuntimeClass | Selects a runtime handler (e.g. a Wasm shim) and the nodes that have it |
| Service mesh | Encryption, telemetry and routing for Pod traffic (Istio, Linkerd, Consul...) |
| Wasm | Sandboxed, portable binary format; runs on Kubernetes via containerd shims |
| YAML | The usual format for Kubernetes config; a superset of JSON |
