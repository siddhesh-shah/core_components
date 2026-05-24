Here is a detailed, comprehensive architectural summary of everything we have discussed. It is structured in clean Markdown format with explicit troubleshooting commands and architectural mechanics, making it ready to be stored directly in your documentation repository (e.g., GitHub or GitLab wiki).

---

# OpenShift & Kubernetes Control Plane Architecture: Operations & Troubleshooting Guide

This repository contains core architectural breakdowns, operational "why/how" details, and real-world troubleshooting steps derived from cluster verification processes.

---

## 1. Core API Server Architecture & Routing Pipeline

OpenShift splits its administration into two distinct API layers to keep the native Kubernetes codebase clean while supporting extensive enterprise platform mechanics. To an operator or user client, this abstraction is hidden, and everything functions via one unified API endpoint.

```
                  [ Human Client / CLI ]
                            │
                            ▼ (Port 6443)
                   ┌─────────────────┐
                   │  kube-apiserver │ <─── (AuthN ──> AuthZ ──> Admission Controllers)
                   └────────┬────────┘
                            │
            ┌───────────────┴───────────────┐
            ▼ (Core Objects)                ▼ (Aggregated Proxy)
       ┌─────────┐                 ┌─────────────────────┐
       │  ETCD   │                 │ openshift-apiserver │
       └─────────┘                 └──────────┬──────────┘
                                              ▼ (Custom Logic Execution)
                                         ┌─────────┐
                                         │  ETCD   │
                                         └─────────┘

```

### The API Request Pipeline Order

1. **Authentication (AuthN):** Validates *who* you are (using TLS client certs, OAuth tokens, etc.).
2. **Authorization (AuthZ):** Evaluates *if* you have permissions to perform the action using Role-Based Access Control (RBAC).
3. **Admission Control:** Intercepts objects during creation/mutation to modify or validate payloads (e.g., validating or mutating webhooks).

### API Aggregation Mechanics (The Proxy Action)

When an operator or user connects to `kubernetes.default.svc:443`, it talks strictly to the `kube-apiserver`.

* If a core object (`Pod`, `Secret`, `Namespace`) is requested, `kube-apiserver` processes it directly.
* If an OpenShift-specific object (`Route`, `Build`, `ImageStream`) is requested, the `kube-apiserver` acts as a **reverse proxy (middleman)**. It looks up the registration map (`APIService`), holds the client connection open, forwards the request backend to the `openshift-apiserver` pods, gets the response, and hands it back to the client.

### Operational Proofs & Diagnostics

* **View Handoff Mapping:** `oc get apiservice v1.route.openshift.io -o yaml` (Look for the `spec.service` block routing traffic to the `openshift-apiserver` namespace).
* **When to inspect `kube-apiserver` logs:** Failures on foundational objects (`Pods`, `Nodes`, global auth/RBAC errors, and admission webhook timeouts).
* *Command:* `oc logs -n openshift-kube-apiserver -l apiserver=true --tail=100`


* **When to inspect `openshift-apiserver` logs:** Failures creating projects, sync-ing external LDAP identities, or generating routes/builds.
* *Command:* `oc logs -n openshift-apiserver -l app=openshift-apiserver --tail=100`



---

## 2. Infrastructure Isolation & DNS Architecture

OpenShift separates production access and automated cluster platform mechanics into separate infrastructure doors.

### Dual Endpoint Domain Separation

1. **`api.<cluster-domain>` (External API):** Resolves to an External Load Balancer. It is utilized by human operators (`oc` CLI, Web Console) and external tools (CI/CD engines). Port: `6443`.
2. **`api-int.<cluster-domain>` (Internal API):** Resolves to an Private/Internal Load Balancer. It is locked down to the cluster VPC/subnet and utilized strictly by internal machinery (Nodes reporting heartbeats, MachineConfig server syncing states). Ports: `6443` and `22623` (Machine Config Ignition files).

### The Bootstrap Cutover Mechanic

During User-Provisioned Infrastructure (UPI) or bare-metal setups, `api-int` is mapped in a load balancer across four hosts: the temporary Bootstrap node and the three permanent Master nodes.

* **The Switch:** The cutover does **not** rely on dynamic DNS updates. The Load Balancer uses automated TCP/HTTP **Health Checks** on port 6443.
* At boot, the empty master nodes fail health checks; 100% of traffic goes to the bootstrap.
* Once master components spin up, they pass health checks, and traffic balances across all nodes.
* When the bootstrap VM is shut down manually post-install, health checks fail instantly, and the load balancer permanently routes all internal traffic to the control plane.

### Operational Proofs & Diagnostics

* **Verify hardcoded node configuration:** Log inside a running node via `oc debug node/<node-name>`, run `chroot /host`, and look at `cat /etc/kubernetes/kubeconfig`. The server line will explicitly target `api-int` to keep node communication localized and high-speed.

---

## 3. Webhook Architecture & Network Isolation Troubleshooting

Admission Webhooks require **mutual TLS (mTLS)** for complete cryptographic safety.

### Webhook Mutual TLS Mechanics

* The `clientConfig` block inside a Webhook configuration requires a `caBundle` string.
* When the `kube-apiserver` connects to a webhook pod, the webhook presents its server certificate, which the API server validates using the `caBundle`.
* Concurrently, the webhook pod challenges the API server, demanding its client certificate to verify that the inbound call actually originated from the almighty cluster brain and not a rogue pod.

### Network Policy "Deny All" Webhook Lockouts

Enforcing an aggregate `Deny All` ingress `NetworkPolicy` on a namespace where an admission webhook resides will immediately break the control plane if the webhook's `failurePolicy` is set to `Fail`. The `kube-apiserver` originates from outside the target namespace pod network (often from the master host network) and will be silently blocked by the network policy firewall.

### Resolution (Least-Privilege Path)

Rather than deleting the safety boundary, punch a targeted hole through the network policy allowing traffic onto the specific secure port of your webhook pod:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-apiserver-to-webhook
  namespace: <webhook-namespace>
spec:
  podSelector:
    matchLabels:
      app: my-security-webhook # Target only the webhook pod
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 8443 # Target only the precise application port

```

*Note: If standard namespace/pod labels fail due to OVN-Kubernetes host network mapping, you must use an explicit `ipBlock` targeting the master subnet CIDR ranges.*

---

## 4. The Automation Engine: Core vs. Extension Controllers

The **Kube Controller Manager** (`kube-controller-manager`) acts as the cluster's muscle, running continuous loops to reconcile the actual cluster state with the user's desired state ($\text{Desired} = \text{Actual}$).

### Foundational Core Controllers (Mandatory Lifecycle Operations)

* **Node Controller:** Evaluates node network heartbeats. If a machine remains uncommunicative past a grace threshold, it flags the node as `NotReady`.
* **Replication/ReplicaSet Controller:** Acts as a pod bodyguard. If a pod crashes or a host drops, it instantly spins up a replacement copy.
* **Endpoint Controller:** Tracks pod IP assignment and continuously maps active pod IPs into the corresponding `Endpoints` backend array of a `Service`.
* **Service Account Controller:** Detects namespace creation and immediately injects standard, default namespace security identities and token secrets.

### Extension Feature Controllers: HPA Mechanics

The **Horizontal Pod Autoscaler (HPA)** controller belongs to the extension layer and depends heavily on foundational core controllers.

* It does **not** create pods. It watches cluster metrics data via Prometheus/Metrics APIs, recalculates the replica math using target ratios, and writes the output directly into the `spec.replicas` line of a Deployment object.
* The foundational **Replication Controller** detects that update and coordinates with the API server to scale up the actual pods.

### Operational Guard Pods

OpenShift introduces dummy `*-guard` pods (e.g., `etcd-quorum-guard`) onto master nodes. Because the core database and control plane components run as **Static Pods** (invisible to standard API-level protections), OpenShift schedules these guard pods alongside them. The guard pods are bound to strict **Pod Disruption Budgets (PDBs)**. This ensures that an administrator running a node drain cannot accidentally take down multiple master nodes at once, keeping quorum intact.

---

## 5. OpenShift-Specific Controllers & Common Failures

The **OpenShift Controller Manager** (`openshift-controller-manager`) executes reconciliation loops exclusively for specialized enterprise abstractions.

| Controller | Functional Responsibility | Common Root Cause Failures | Diagnostics & Commands |
| --- | --- | --- | --- |
| **Build Controller** | Controls the S2I lifecycle, building app code into images. | • Invalid Git repository secrets/SSH keys.<br>

<br>• Internal/External registry pull authentication timeouts. | • `oc get builds`<br>

<br>• `oc logs build/<name>`<br>

<br>• `oc describe build/<name>` *(Look for `FailedCreatePodCondition`)* |
| **DeploymentConfig** | Handles advanced rollouts and rollback triggers. | • `ImageStream` tags mismatched or misspelled.<br>

<br>• Pre/Post deployment lifecycle hook containers freezing. | • `oc get rc`<br>

<br>• `oc describe dc/<name>`<br>

<br>• `oc logs dc/<name> --version=<num>` |
| **Route Controller** | Syncs `Route` objects down into HAProxy engines. | • Host URL collision (hostname claimed in another namespace).<br>

<br>• Incompatible or broken custom private TLS key formatting. | • `oc get route <name> -o yaml`<br>

<br>• Check `status.ingress` entries at the bottom for `admitted: False`. |
| **Project Controller** | Auto-provisions namespace components upon request. | • Global template files corrupted via upstream cluster configuration errors.<br>

<br>• Finalizer lockouts (stuck permanently in `Terminating`). | • `oc get project <name> -o json | jq '.status.conditions'` *(Pinpoint lingering objects blocking deletion)* |

---

## 6. The Matchmaking Engine: Kube-Scheduler Mechanics

The **Kube-Scheduler** (`kube-scheduler`) handles workload node assignment through a decoupled, two-phase algorithmic sequence.

### Phase 1: Filtering (Predicates - Binary Hard Rules)

Nodes must pass **all** checks or they are immediately disqualified:

* `PodFitsResources`: Verifies if the node's **Allocated** allocation pool can take on the pod's CPU/Memory request.
* `PodMatchesNodeSelector`: Mandates label verification matches.
* `NodeIsReady`: Evaluates if the host node is healthy and uncordoned.
* `Tolerations & Taints`: Drops any node that contains a hard execution restriction (Taint) unless the pod possesses the precise cryptographic exemption key (Toleration).

### Phase 2: Scoring (Priorities - Variable Soft Rules)

Surviving nodes are assigned an incremental score (0 to 100) across structural plugins:

* `ImageLocalityPriority`: Awards higher priority scores to nodes that **already have the container images pre-cached** on their physical disk, reducing startup latency.
* `SelectorSpreadPriority`: Multiplies the score of nodes that are not currently running active replicas of the same workload, spreading pods evenly across availability zones.
* `LeastRequestedPriority`: Distributes workloads gracefully to keep machine temperatures even cluster-wide.

---

## 7. Crucial Distinction: Allocated vs. Used Resources

Understanding the complete decoupling of scheduling math versus real-time physical performance is essential to prevent false cluster resource assumptions.

```
+-----------------------------------------------------------+
| WORKER NODE CAPACITY: 4 CPUs                              |
+-----------------------------------------------------------+
| [■ POD A REQUEST: 2 CPUs]   [■ POD B REQUEST: 1.5 CPUs]   | <─── ALLOCATED (3.5 / 4 CPUs)
| (Real Usage: 0.1 CPU)       (Real Usage: 0.2 CPU)         |      The Scheduler sees this node as 87.5% FULL.
+-----------------------------------------------------------+
| [ Free Headroom: 0.5 CPU ]                                | <─── Any new pod requesting > 0.5 CPU is REJECTED
+-----------------------------------------------------------+
| PHYSICAL UTILIZATION (oc adm top nodes): 0.3 / 4 CPUs     | <─── REAL CONSUMPTION (7.5% utilized)
+-----------------------------------------------------------+

```

### 1. Allocated Resources (Requests)

This is the amount of resource headroom a pod's manifest "books" upon landing. **The `kube-scheduler` makes decisions based entirely on Allocated numbers.** * If a node's physical CPU utilization is sitting at `5%`, but historical pod requests total `95%` of its total capacity, the scheduler will throw the node away if a newly submitted pod requests more than the remaining `5%` allocation headroom. This triggers a `FailedScheduling: Insufficient cpu/memory` flag.

### 2. Used Resources (`oc adm top`)

This reflects real-time physical telemetry. It shows exactly how many CPU clock cycles the physical processor chip is crunching at that precise instant.

* A massive delta between high allocated requests and minimal real-time usage highlights an over-provisioned cluster environment where developers are excessively hoarding space capacity.

### Diagnostic Command Summary

* **View Node Allocation Ledger:** `oc describe node <node-name>` *(Scroll down directly to the `Allocated resources:` metrics matrix block)*.
* **Inspect Real Usage:** `oc adm top nodes` or `oc adm top pods`
* **Evaluate Pod Rejections:** `oc describe pod <pending-pod> -n <namespace>` *(Look at the `Events` log chunk for explicit filtering failure counts)*.
