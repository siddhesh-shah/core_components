Here is a complete, GitHub-ready Markdown file containing all the consolidated notes, commands, workflows, and troubleshooting steps for OpenShift's Operator Lifecycle Manager (OLM). You can copy and paste this directly into a `README.md` or a personal knowledge-base repository.

---

# OpenShift Operator Lifecycle Manager (OLM) - Master Notes

## 📌 1. Core Concepts

* **Operator:** A Kubernetes software extension that uses Custom Resources (CRs) and controllers to automate the lifecycle (deployment, updates, day-2 operations) of applications.
* **OLM (Operator Lifecycle Manager):** A native OpenShift framework that acts as a management engine and enterprise app store. It handles the installation, upgrades, and Role-Based Access Control (RBAC) for all operators.
* **OperatorHub:** The graphical UI in OpenShift, acting as an "App Store" fed by OLM's backend catalogs.

## ⚙️ 2. Core OLM Custom Resources (CRs)

OLM relies on a specific hierarchy of resources to function.

| Custom Resource | Function |
| --- | --- |
| **CatalogSource** | A backend repository pointing to an index image of operators. It feeds data into OLM. |
| **PackageManifest** | Represents a single operator and all its versions/channels (generated from the CatalogSource). |
| **OperatorGroup** | Manages RBAC and scoping. Defines which namespaces the operator is allowed to watch. |
| **Subscription** | Subscribes the cluster to an operator's specific channel and tracks updates. |
| **InstallPlan** | A generated pre-installation checklist. Shows exactly what OLM intends to create. |
| **ClusterServiceVersion (CSV)** | Represents the single, installed version of the operator. Manages the actual deployment of controllers. |

---

## 🚀 3. Installation Workflow (CLI)

When you click "Install" in the UI, OpenShift does this in the background. Here is how to do it manually via the CLI.

### Step 1: Discover the Operator

```bash
# List all available operators
oc get packagemanifests -n openshift-marketplace

# Find specific channels and package names
oc describe packagemanifest <operator-name> -n openshift-marketplace

```

### Step 2: Create the Target Namespace

```bash
oc create namespace <target-namespace>

```

### Step 3: Create the OperatorGroup (Required for RBAC)

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: my-operator-group
  namespace: <target-namespace>
spec:
  targetNamespaces:
  - <target-namespace> # Leave blank if Cluster-wide

```

```bash
oc apply -f operator-group.yaml

```

### Step 4: Create the Subscription

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-operator-sub
  namespace: <target-namespace>
spec:
  channel: stable
  name: <operator-package-name>
  source: redhat-operators 
  sourceNamespace: openshift-marketplace
  installPlanApproval: Manual # 'Automatic' or 'Manual'

```

```bash
oc apply -f subscription.yaml

```

### Step 5: Approve the InstallPlan (If set to Manual)

```bash
# Find the InstallPlan
oc get installplan -n <target-namespace>

# Review what it will do
oc describe installplan <installplan-name> -n <target-namespace>

# Approve it
oc patch installplan <installplan-name> -n <target-namespace> --type merge --patch '{"spec":{"approved":true}}'

```

### Step 6: Verify Installation

```bash
# Watch the CSV transition to 'Succeeded'
oc get csv -n <target-namespace> -w

# Check operator pods
oc get pods -n <target-namespace>

```

---

## 🔄 4. Upgrades & Polling Behavior

* **Registry Polling:** The `catalog-operator` polls the upstream container registry (e.g., every 15 or 240 minutes) to check if the CatalogSource index image has been updated.
* **In-Channel Upgrades:** If a new version is found in the *current* channel, OLM executes it instantly (if `Automatic`) or generates a pending InstallPlan (if `Manual`).
* **Cross-Channel Upgrades:** OLM **never** changes channels automatically. You must patch the subscription:
```bash
oc patch subscription <sub-name> -n <namespace> --type merge --patch '{"spec":{"channel":"new-channel-name"}}'

```


* **Namespace-Wide Sync:** If *any* operator in a namespace has its approval strategy set to `Manual`, OLM forces all other operators in that namespace to wait for manual approval as well.
* **Downgrades/Older Versions:** If you specifically install an older version of an operator from a channel, OLM will force the Subscription to `Manual` to prevent it from immediately auto-upgrading to the latest version.

---

## 🛠️ 5. Troubleshooting Guide

### Issue 1: Operator is missing from the OperatorHub UI

**Where to check:**

1. **CatalogSource:** Check if the default catalogs are disabled or failing.
```bash
oc get catalogsource -n openshift-marketplace
oc get pods -n openshift-marketplace

```


2. **PackageManifest:** If the catalog pod is running, check if OLM parsed the operator.
```bash
oc get packagemanifest -n openshift-marketplace | grep <operator-name>

```


3. **Architecture Compatibility:** If the PackageManifest exists but it's hidden in the UI, the operator might not support the cluster's underlying CPU architecture (e.g., ARM vs. x86).

### Issue 2: Installation or Upgrade is stuck / not triggering

**Where to check:**

1. **Check for Pending InstallPlans:**
```bash
oc get installplan -n <target-namespace>

```


*Fix: Approve the pending InstallPlan.*
2. **Check the Subscription status and events:**
```bash
oc describe subscription <sub-name> -n <target-namespace>

```


*Look for errors like "no operator found from the catalog" (meaning the CatalogSource went down or the channel name is typo'd).*
3. **Check for Namespace Conflicts:** Does another operator in the same namespace have a failing or unapproved InstallPlan? OLM bundles namespace updates. Fix the broken operator to unblock the others.

### Issue 3: Custom CatalogSource is failing to load

**Where to check:**

1. **Check the Catalog Pod:**
```bash
oc get pods -n openshift-marketplace

```


If it shows `ImagePullBackOff` or `ErrImagePull`:
```bash
oc describe pod <catalog-pod-name> -n openshift-marketplace

```


*Fix: Usually caused by missing registry credentials, invalid DNS resolution to the custom registry, or an incorrect image tag.*

### Issue 4: Where to find OLM System Logs

If the entire OLM system seems broken, check the core controller pods located in the `openshift-operator-lifecycle-manager` and `openshift-marketplace` namespaces.

```bash
# View logs for the catalog operator (handles registry polling and PackageManifests)
oc logs deployment/catalog-operator -n openshift-operator-lifecycle-manager

# View logs for the OLM operator (handles CSVs and actual deployments)
oc logs deployment/olm-operator -n openshift-operator-lifecycle-manager

```

---

## ⚠️ 6. Administration & Edge Cases

* **Disabling Default Catalogs:**
```bash
# Empties the OperatorHub entirely
oc patch operatorhub cluster --type merge --patch '{"spec":{"disableAllDefaultSources":true}}'

```


*Note: Disabling a catalog does **not** impact currently running operators. It only prevents discovery and future upgrades.*
* **Uninstalling an Operator:** Deleting an operator (CSV and Subscription) removes the controller pods. **It does NOT delete the workloads (Custom Resources) managed by that operator.** The workloads will simply run unmanaged until the operator is reinstalled or the CRs are manually deleted.
