You have put in a massive amount of effort to learn this architecture inside and out! Transitioning from high-level concepts down into the bare-metal Linux commands is exactly how senior engineers are forged.

Here is a complete, GitHub-ready Markdown file of everything we discussed. You can copy this directly into a `.md` file in your repository. It is structured specifically for quick scanning during a live troubleshooting scenario.

---

# ☸️ Kubernetes & OpenShift Control Plane Survival Guide

## 1. etcd: The Brain of the Cluster

etcd is a distributed, highly available key-value store. It is the absolute source of truth for the cluster. If etcd dies, the API server loses its memory and the cluster halts.

### The Raft Consensus Algorithm

* **Leader:** The absolute boss. Handles all writes, replicates to followers, and sends constant heartbeats. (Only ONE leader at a time).
* **Follower:** Passive. Receives data and listens for heartbeats.
* **Candidate:** If a follower misses a heartbeat (randomized 150ms-300ms timeout), it assumes the leader is dead and campaigns for votes.
* **Quorum:** The majority vote needed to commit a write or elect a leader.
* *Troubleshooting Note:* If nodes frequently switch to `Candidate` state, the cluster is suffering from network starvation (dropped packets) or disk starvation (slow I/O).

### Storage Architecture

etcd strictly separates its storage into two phases:

1. **WAL (Write-Ahead Log) - `/var/lib/etcd/member/wal/**`
* Raw, append-only binary files (usually 64MB chunks).
* Written **FIRST** (Sequential I/O for blazing speed). Survives sudden power loss.


2. **BoltDB - `/var/lib/etcd/member/snap/db**`
* The actual key-value tree structure.
* Written **AFTER** the quorum commits the data in RAM.



## 2. etcd Maintenance: Compaction & Defragmentation

Because etcd uses MVCC (Multi-Version Concurrency Control), it never overwrites data; it saves every historical revision. This hoarder behavior requires strict maintenance.

* **Compaction:** Deletes old historical revisions. Frees up *logical* space (empty rooms), but the database file stays the same physical size. OpenShift automates this every 5 minutes.
* **Defragmentation:** Rebuilds the database file without the empty gaps. Shrinks the actual physical `DB SIZE` on the hard drive.
* *Golden Rule:* Must be done **one node at a time** (followers first, leader last) because it temporarily locks the database.



## 3. The `etcdctl` Admin Cheat Sheet

*Note: Run inside the etcd pod, or add mTLS `--cacert`, `--cert`, and `--key` flags if running from the host.*

**Cluster Health & Status**

```bash
# Check if nodes are talking and who is the LEADER
etcdctl endpoint status --cluster -w table

# Check if the database is responding fast
etcdctl endpoint health --cluster -w table

# Check for Alarms (e.g., NOSPACE)
etcdctl alarm list

```

**Space & Cleanup**

```bash
# Get exact used vs free space (Look for dbSize vs dbSizeInUse)
etcdctl endpoint status --write-out=json

# Manual Compaction (Find current revision first, then compact)
etcdctl compact <REVISION_NUMBER>

# Manual Defrag (Shrinks the disk file)
etcdctl defrag

# Disarm a NOSPACE alarm after cleaning up
etcdctl alarm disarm

```

**Backups & Data Inspection**

```bash
# Take a snapshot backup
etcdctl snapshot save /tmp/backup.db

# Find what is bloating your database (The Top 10 worst namespaces)
# Replace 'events' with 'pods' or 'secrets' to check other objects
etcdctl --command-timeout=60s get --prefix --keys-only /kubernetes.io/events | awk -F/ '{ print $4 }' | sort | uniq -c | sort -n

```

## 4. Node Architecture: Kubelet vs. Controllers

* **The Kubelet:** The "dumb" node agent. Its only job is to talk to the container runtime and ensure containers stay running. It enforces OOMKills, runs liveness/readiness probes, and reports node health to the API.
* **Static Pods:** The Kubelet watches `/etc/kubernetes/manifests/`. If you put a YAML file there, the Kubelet starts the pod locally, completely bypassing the API Server. This is how the API Server and etcd boot up.
* **etcd Operator/Controller:** The "Automated DBA". It handles the logic the Kubelet cannot do: Raft scaling, certificate rotation, and automated backups/compaction.
* **Kube-Proxy:** The networking agent. Writes `iptables` or `IPVS` rules so pods can route traffic to each other.

## 5. CRI-O & System-Level Operations

Modern Kubernetes uses lightweight OCI runtimes (like CRI-O) instead of Docker.

* **gRPC over Unix Domain Sockets:** The Kubelet talks to CRI-O using a local Linux file (`.sock`). It uses HTTP/2 and Protobuf binaries for lightning-fast, secure, internal communication without touching the network stack.
* **OverlayFS:** Saves disk space by stacking image layers. The base image is Read-Only; the running container gets a tiny, temporary Read-Write layer on top.
* **conmon:** The hidden daemon that monitors the actual `runc` Linux process, grabs stdout logs, and catches exit codes.

**The `crictl` Break-Glass Commands**
Use these when the API server is dead and `oc` commands hang. SSH into the master node and bypass Kubernetes entirely:

```bash
# List all running/dead containers (your alternative to docker ps)
crictl ps -a

# Read raw container logs to find out why the API server crashed
crictl logs <CONTAINER_ID>

# List all Pod Sandboxes (the networking wrapper for the containers)
crictl pods

```

## 6. OpenShift Internal Image Registry

A built-in enterprise registry managed by the Cluster Image Registry Operator.

* **Architecture:** Stateless registry pods + a Service Load Balancer + Shared Object Storage (AWS S3, NFS, etc.).
* **Pruner Pods:** CronJobs that delete orphaned image layers to save disk space.
* **ImageStreams:** OpenShift's webhook system. Pushing an image to the registry triggers the ImageStream, which instantly auto-deploys the new code.
* **Admin Commands:**
```bash
# Expose the registry to the outside world
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge

# Mirror an image directly from an external repo to the internal one
oc image mirror quay.io/repo/app:latest <internal-registry-host>/namespace/app:latest

```



## 7. The Ultimate Control Plane Troubleshooting Workflow

When users complain `oc` commands are hanging, follow this exact order:

1. **Check Physical Node Resources:** Run `oc adm top nodes`. If masters are at 100% CPU, the API is suffocating.
2. **Check etcd Disk Speed (Fsync):** Look in Grafana. `etcd_disk_wal_fsync_duration_seconds` **MUST be < 10ms**. If it's 50ms+, your hard drives are failing the cluster. Test hardware limits with the `fio` utility.
3. **Check etcd Restarts & Logs:** Run `oc get pods -n openshift-etcd`. Look for recent restarts or `grep "took too long"` in the pod logs.
4. **Check API Server Load:** Search API logs for timeouts to see if a rogue operator is spamming the cluster with requests.
5. **Check Webhooks:** If etcd is fast and API is healthy, check `mutatingwebhookconfigurations`. A crashed security scanner webhook will cause the API to hang while waiting for validation.
6. **Check the Load Balancer:** Ensure the external VIP routing traffic to your masters is healthy.
