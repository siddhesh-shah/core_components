Here is a complete, comprehensive production handbook designed to be saved directly into a GitHub repository (e.g., `MCO-CHEAT_SHEET.md`). It consolidates everything we discussed into a clean, technical, markdown-compatible format complete with internal architectures, troubleshooting flows, commands, and expected behaviors.

---

# 🛠️ OpenShift Machine Config Operator (MCO) Production Handbook

## 📌 Core Concept: Why MCO & RHCOS?

Red Hat Enterprise Linux CoreOS (RHCOS) is an **immutable, container-optimized operating system**.

* **The Problem:** The core filesystem is read-only. Manual modifications via SSH (like editing files in `/etc` or installing packages via RPMs) will cause configuration drift, degrade nodes, or be completely wiped out during a cluster upgrade.
* **The Solution:** The **Machine Config Operator (MCO)** brings the operating system under the control of the Kubernetes control plane. It enables declarative configuration management using standard Kubernetes manifests (`oc apply`).

---

## 🏗️ Architecture & Core Components

```
                  +-----------------------------------+
                  |  MachineConfig Controller (MCC)   |  <-- Watches MCs/MCPs, 
                  +-----------------------------------+      Renders blueprints
                                    |
                                    v
                  +-----------------------------------+
                  |     MachineConfigPool (MCP)       |  <-- Target grouping
                  +-----------------------------------+
                                    |
       +----------------------------+----------------------------+
       | (Master Pool)                                           | (Worker Pool)
       v                                                         v
+-------------------------------+                         +-------------------------------+
| MachineConfig Daemon (MCD)    |                         | MachineConfig Daemon (MCD)    |
| (Runs on Master Node Pod)     |                         | (Runs on Worker Node Pod)     |
+-------------------------------+                         +-------------------------------+
       |                                                         |
       v (Applies to Host Disk)                                  v (Applies to Host Disk)
[ Immutable RHCOS OS Disk ]                               [ Immutable RHCOS OS Disk ]

```

### 1. Component Breakdown

* **MachineConfig (MC) [CRD]:** The "Source of Truth" blueprint containing the targeted Ignition payload (`storage` files, `systemd` units, `passwd` user keys, kernel arguments).
* **MachineConfigPool (MCP) [CRD]:** The matching engine that pools nodes together based on roles (`master`, `worker`, or custom roles like `infra`).
* **MachineConfig Controller (MCC) [Control Plane Pod]:** The brain. It watches for changes, collects labeled MCs, sorts them alphanumerically, handles overrides, and bakes the final compiled manifest (**Rendered MachineConfig**).
* **MachineConfig Daemon (MCD) [DaemonSet Pod]:** The worker bee running on **every** node. It monitors the MCP status, cordons/drains its host node, applies the rendered blueprint, and reboots the hardware.
* **MachineConfig Server (MCS) [Control Plane Pod]:** An internal web server that supplies the initial Ignition configuration exclusively during the provisioning/bootstrapping phase of a brand-new node.

---

## 🔄 The Rendering & Rollout Workflow

```
[Admin applies MC] 
       │
       ▼
[MCC matches labels] ──► [Sorts alphanumerically (00 -> 99)] ──► [Generates rendered-worker-xyz]
                                                                             │
                                                                             ▼
[MCD detects change] ──► [Cordon & Drain Node] ──► [Write to Disk/Reboot] ──► [Uncordon Node]

```

1. **Submit Manifest:** Admin applies a custom `MachineConfig` with a selector label (e.g., `machineconfiguration.openshift.io/role: worker`).
2. **Merge & Sort:** The MCC intercepts the manifest, pulls all configs for that pool, and sorts them alphanumerically by `metadata.name`.
> 💡 **Conflict Rule:** **Highest prefix wins.** If `01-worker-config` defines a setting, but `99-worker-custom` modifies that exact same setting, the `99` configuration overwrites the `01` configuration inside the final render.


3. **Generate Render:** The MCC outputs a single `rendered-worker-<hash>` file and updates the MCP's `desiredConfig`.
4. **Node Actions:** The MCD running on the node recognizes the new `desiredConfig`. It handles the transition one node at a time (controlled by `maxUnavailable`):
* **Cordon:** Marks node unschedulable.
* **Drain:** Safely evicts running pods.
* **Write:** Applies Ignition changes directly onto the underlying host filesystem.
* **Reboot:** Powers the host cycle down and up into the new immutable state.
* **Uncordon:** Rejoins the active compute fleet.



---

## 📋 Essential Command Sheet

### Cluster Status & Inspection

```bash
# Get high-level status of all MachineConfigPools
oc get mcp

# Inspect a specific pool to verify current vs desired configurations
oc get mcp worker -o yaml

# List all available MachineConfigs (including system defaults and custom configurations)
oc get mc

# View the actual compiled blueprint that is being pushed to worker nodes
oc get mc $(oc get mcp worker -o jsonpath='{.status.configuration.name}') -o yaml

```

### Managing Rollout Behavior

```bash
# Pause updates/rollouts on a pool (useful for setting a maintenance window)
oc patch mcp/worker --type=merge -p='{"spec":{"paused":true}}'

# Resume updates/rollouts on a pool
oc patch mcp/worker --type=merge -p='{"spec":{"paused":false}}'

# Change concurrency limits (set maxUnavailable dynamically via absolute int or percentage)
oc patch mcp/worker --type=merge -p='{"spec":{"maxUnavailable": "10%"}}'

```

### Node Operations

```bash
# Add a custom role to a node (Pre-requisite for custom pools)
oc label node <node-name> node-role.kubernetes.io/infra=""

# Safely check node drift or logs directly from the node via host debug session
oc debug node/<node-name>

```

---

## 🛠️ Production Scenarios & Troubleshooting

### 1. Manual File Creation (Drift Behavior)

* **Modifying a Managed File:** If you modify a file tracked by an Ignition payload, the MCD detects file checksum differences. It flags the pool as `DEGRADED: True` and **immediately overwrites your changes** with the cluster's defined source of truth.
* **Creating Untracked Files:** Creating a random file in `/etc` or `/var` will be ignored by the MCO, but it will cause stealth configuration drift. New or scaled-up nodes will lack this file, breaking environment parity.
* **Modifying Read-Only Areas:** Attempting to alter paths like `/usr`, `/bin`, or `/sbin` returns a `Read-only file system` error. Any forced remount modifications are completely erased during the next OS upgrade cycle.

### 2. How to Safely Undo/Rollback a Change

* **If Paused:** If your pool was `paused: true` when you applied a broken MachineConfig, simply delete the custom MachineConfig: `oc delete mc <bad-config-name>`. The MCC will recalculate a clean render. When you unpause, the nodes will match the old state and skip any reboots.
* **If Actively Rolling Out:** 1. Immediately pause the pool to stop subsequent nodes from breaking: `oc patch mcp/worker --type=merge -p='{"spec":{"paused":true}}'`
2. Delete the offending configuration: `oc delete mc <bad-config-name>`
3. Unpause the pool: The MCO will auto-recalculate the clean render, drain the degraded nodes, roll back the changes, and reboot them to full health.

### 3. Log Inspection & Troubleshooting Deep-Dive

When an MCP reports `DEGRADED: True`, use this exact troubleshooting flow to isolate and fix the root cause:

```
                  ┌───────────────────────────────────────────┐
                  │          MCP shows DEGRADED: True         │
                  └─────────────────────┬─────────────────────┘
                                        │
                                        ▼
                  ┌───────────────────────────────────────────┐
                  │ Run: oc describe mcp <pool>               │
                  │ Check "Messages" section for error string │
                  └─────────────────────┬─────────────────────┘
                                        │
                                        ▼
                  ┌───────────────────────────────────────────┐
                  │ Locate the failing node name from error   │
                  └─────────────────────┬─────────────────────┘
                                        │
                                        ▼
                  ┌───────────────────────────────────────────┐
                  │ Run: oc get pods -n openshift-machine-... │
                  │ Identify target machine-config-daemon pod │
                  └─────────────────────┬─────────────────────┘
                                        │
                         ┌──────────────┴──────────────┐
                         ▼                             ▼
           ┌───────────────────────────┐ ┌───────────────────────────┐
           │ Run: oc logs <pod-name>   │ │ Run: oc debug node/<node> │
           │ Check daemon execution    │ │ Check host systemd unit   │
           │ error state logs          │ │ error state logs          │
           └───────────────────────────┘ └───────────────────────────┘

```

#### Step A: Read the High-Level Object Status

```bash
oc describe mcp worker

```

* **What to look for:** Scroll down to the `Status.Conditions` and `Messages` blocks. The operator explicitly logs the node name and the fault reason here (e.g., *“Node worker-0.example.com is degraded: failed to execute absolute systemd conversion”*).

#### Step B: Pull Logs from the Control Plane Operator

If the rendering process itself is failing or configs aren't matching up, check the controller's logs:

```bash
oc logs -n openshift-machine-config-operator deployment/machine-config-controller -c machine-config-controller

```

#### Step C: Pull Logs from the Target Node Daemon

If a specific node is failing to apply the files or is hanging during its update loop, check the logs of the daemon pod running *on that node*:

```bash
# Find the exact MCD pod name assigned to your failing node
oc get pods -n openshift-machine-config-operator -o wide | grep <failing-node-name>

# Query the daemon container logs
oc logs -n openshift-machine-config-operator machine-config-daemon-<hash> -c machine-config-daemon --tail=200

```

#### Step D: Investigate Local OS Execution Failures

If the MCO applied the configuration but the node is degraded because a custom `systemd` unit you injected failed to start, use a host debug shell to read the system logs:

```bash
oc debug node/<failing-node-name>
# Inside the debug shell:
chroot /host
journalctl -xeu <your-custom-systemd-service-name>

```

---

## 🚀 Manifest Template: Custom MCP + Custom MC

Save this complete block as a single template file to deploy a safe, production-grade custom node pool tracking base configurations and custom files.

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: infra
spec:
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/infra: ""
  machineConfigSelector:
    matchExpressions:
      # Safeguard: Matches both base worker configs AND custom infra configs to avoid config loss
      - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker, infra]}
---
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: infra
  name: 99-infra-custom-ntp
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - path: /etc/chrony.d/custom-ntp.conf
        mode: 0644
        contents:
          # Contents: "server ntp.example.com iburst" encoded in base64
          source: data:text/plain;charset=utf-8;base64,c2VydmVyIG50cC5leGFtcGxlLmNvbSBpYnVyc3QK

```
