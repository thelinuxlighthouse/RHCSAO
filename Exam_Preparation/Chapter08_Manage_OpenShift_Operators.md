# Chapter 8: Manage OpenShift Operators

> **Exam baseline:** EX280 on OpenShift Container Platform 4.18.  
> **Reviewed:** 2026-07-19.  
> This chapter concentrates on the classic Operator Lifecycle Manager (OLM) resources used by OpenShift OperatorHub: `CatalogSource`, `PackageManifest`, `OperatorGroup`, `Subscription`, `InstallPlan`, and `ClusterServiceVersion`.

## What You Must Be Able to Do

The EX280 objectives require you to install, uninstall, and delete an Operator. You must be able to do that through the web console and by understanding the resources that the console creates.

At the end of this chapter, you should be able to:

- distinguish a cluster Operator from an optional OLM-installed Operator;
- find an Operator package, its channels, and its install modes;
- create a suitable namespace and `OperatorGroup`;
- create and verify a `Subscription`;
- approve a manual `InstallPlan` with a real supported command;
- identify the installed `ClusterServiceVersion` (CSV);
- remove the Subscription and CSV in the correct order; and
- decide which custom resources and CRDs must be retained or cleaned up.

---

## 1. What an Operator Is

An Operator is a controller plus operational knowledge. A controller repeatedly compares desired state with observed state and acts to reduce the difference. For example, a database Operator might create a StatefulSet, Services, Secrets, backups, and upgrades after a user creates one database custom resource.

The main pattern is:

```text
CustomResourceDefinition (new API type)
              |
user creates a Custom Resource (desired database, broker, and so on)
              |
Operator controller watches the resource
              |
Operator creates/updates ordinary Kubernetes and external resources
```

### Cluster Operators are different

`oc get clusteroperators` shows platform Operators managed as part of OpenShift itself, such as authentication, ingress, network, and monitoring. Do not try to uninstall these using an OperatorHub Subscription.

`oc get csv -A` shows Operators installed through classic OLM. The two lists answer different questions.

---

## 2. OLM Objects and Their Relationships

```text
CatalogSource
    |
    +-- publishes packages, channels, bundles, and dependencies
            |
       PackageManifest (discovery view)
            |
       Subscription  <---- desired package + channel + catalog
            |
       InstallPlan   <---- exact resources OLM plans to install
            |
       ClusterServiceVersion (CSV)
            |
       Operator Deployment, RBAC, services, CRDs

OperatorGroup in the Subscription namespace defines the target namespace set
and must match an install mode supported by the CSV.
```

| Object | Scope | Purpose |
|---|---|---|
| `CatalogSource` | Namespaced | Makes an Operator catalog available. Default catalogs live in `openshift-marketplace`. |
| `PackageManifest` | Discovery resource | Shows packages, channels, current CSV names, and catalog details. |
| `OperatorGroup` | Namespaced | Defines the namespaces an Operator installed in that namespace may watch. |
| `Subscription` | Namespaced | Selects package, channel, catalog, approval mode, and optional starting CSV. |
| `InstallPlan` | Namespaced | Records a resolved set of installation steps. Manual plans wait for approval. |
| `ClusterServiceVersion` | Namespaced despite its name | Describes one installed Operator version, permissions, deployment, CRDs, install modes, and status. |
| `CustomResourceDefinition` | Cluster-scoped | Adds an API type used by the Operator. |
| `Custom Resource` | Depends on its CRD | A desired-state instance managed by the Operator. |

“Cluster” in `ClusterServiceVersion` does not make the CSV cluster-scoped. Always use the installation namespace.

---

## 3. OperatorGroup and Install Modes

A namespace used for an OLM installation must have a compatible `OperatorGroup`. There can be only one OperatorGroup in an installation namespace.

CSV install modes are:

| Install mode | Meaning |
|---|---|
| `OwnNamespace` | Watch the namespace in which the Operator is installed. |
| `SingleNamespace` | Watch one selected namespace. |
| `MultiNamespace` | Watch more than one selected namespace. |
| `AllNamespaces` | Watch all namespaces. |

The CSV declares which modes it supports. Do not assume every Operator supports every mode.

### One-namespace OperatorGroup

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: database-operators
  namespace: database-operators
spec:
  targetNamespaces:
  - database-operators
```

### All-namespaces OperatorGroup

Omit both `spec.targetNamespaces` and `spec.selector`:

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: global-example
  namespace: global-example-operators
```

Do **not** put `"*"` in `targetNamespaces`. For all namespaces, the resolved target set is represented to the Operator as an empty string.

The standard `openshift-operators` namespace already has a global OperatorGroup for common all-namespaces installs. Inspect it before creating anything:

```bash
oc get operatorgroup -n openshift-operators
oc get operatorgroup -n openshift-operators -o yaml
```

### Verify resolution

```bash
oc get operatorgroup -n database-operators
oc get operatorgroup/database-operators -n database-operators \
  -o jsonpath='{.status.namespaces}'
echo
```

An unsupported target set can leave a CSV in a failed state with an `UnsupportedOperatorGroup` reason.

---

## 4. Discover the Exact Package Before Installing

Do not guess package names, channel names, or catalog sources.

```bash
oc get catalogsource -n openshift-marketplace
oc get packagemanifest -n openshift-marketplace
oc get packagemanifest -n openshift-marketplace | grep -i '<search-text>'
oc describe packagemanifest/<package-name> -n openshift-marketplace
```

From the package details, record:

- package name for `spec.name`;
- source name for `spec.source`;
- source namespace, normally `openshift-marketplace`;
- available channel names and the default channel;
- supported install modes; and
- the current CSV name for the chosen channel.

OperatorHub in the web console presents the same information more visually. Use **Operators → OperatorHub**, open the Operator, read its capability level, channels, install modes, and prerequisites, and then choose **Install**.

---

## 5. Install an Operator with Classic OLM

The examples below use placeholders deliberately. Package and channel names vary by catalog and time; discover them first.

### Step 1: Create installation namespace

```bash
oc create namespace database-operators
```

### Step 2: Create a compatible OperatorGroup

Save the one-namespace manifest from the previous section as `operatorgroup.yaml`:

```bash
oc apply -f operatorgroup.yaml
oc get operatorgroup -n database-operators
```

### Step 3: Create the Subscription

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: example-operator
  namespace: database-operators
spec:
  channel: <verified-channel>
  installPlanApproval: Automatic
  name: <verified-package-name>
  source: <verified-catalog-source>
  sourceNamespace: openshift-marketplace
```

```bash
oc apply -f subscription.yaml
oc get subscription -n database-operators
oc get installplan -n database-operators
oc get csv -n database-operators
```

### Step 4: Prove successful installation

Do not use a made-up Subscription `phase`. The most useful evidence is the CSV phase and conditions:

```bash
oc get csv -n database-operators \
  -o custom-columns=NAME:.metadata.name,VERSION:.spec.version,PHASE:.status.phase

oc get subscription/example-operator -n database-operators -o yaml
oc get csv/<installed-csv-name> -n database-operators -o yaml
oc get pods -n database-operators
```

A successfully installed CSV normally reports `Succeeded`. If the Operator requires an operand custom resource, successful CSV installation does not mean an operand has been created; create and verify that resource separately only when the task requests it.

---

## 6. Manual InstallPlan Approval

With `installPlanApproval: Manual`, OLM creates a plan but waits.

```bash
oc get installplan -n database-operators
oc describe installplan/<plan-name> -n database-operators
```

Approve it with a patch:

```bash
oc patch installplan/<plan-name> \
  -n database-operators \
  --type=merge \
  -p '{"spec":{"approved":true}}'
```

There is no standard `oc approve installplan` command.

After approval:

```bash
oc get installplan/<plan-name> -n database-operators
oc get csv -n database-operators
oc get pods -n database-operators
```

Manual approval also affects later updates. Inspect what the plan installs before approving it.

---

## 7. Read Operator Status Correctly

### Catalog source

```bash
oc get catalogsource -n openshift-marketplace
oc get catalogsource/<source> -n openshift-marketplace \
  -o jsonpath='{.status.connectionState.lastObservedState}{"\n"}{.status.connectionState.lastConnect}{"\n"}'
```

For a gRPC catalog, `READY` is the expected connection state.

### Subscription

```bash
oc describe subscription/<name> -n <namespace>
oc get subscription/<name> -n <namespace> \
  -o jsonpath='{.status.currentCSV}{"\n"}{.status.installedCSV}{"\n"}'
```

Read `.status.conditions[*].type`, `status`, `reason`, and `message`. Useful failure condition types include catalog-health and resolution failures; the message is more useful than memorizing a state diagram.

### InstallPlan

```bash
oc get installplan -n <namespace>
oc describe installplan/<name> -n <namespace>
```

Confirm the approval mode, approved value, CSV names, and failed steps.

### CSV

```bash
oc get csv -n <namespace>
oc describe csv/<name> -n <namespace>
```

Common CSV phases include `Pending`, `InstallReady`, `Installing`, `Succeeded`, `Failed`, `Replacing`, and `Deleting`. These are CSV phases—not a universal ten-stage Operator lifecycle. Read `.status.reason` and `.status.message` when the phase is not `Succeeded`.

---

## 8. Remove an OLM-Installed Operator Safely

There are three different cleanup layers:

1. **Operand data and custom resources** managed by the Operator.
2. **OLM installation objects**, primarily Subscription and CSV.
3. **Shared API definitions and infrastructure**, such as CRDs, namespaces, and off-cluster resources.

Removing the Operator controller does not automatically remove managed custom resources, CRDs, or external data. Plan cleanup before stopping the controller.

### Step 1: Inventory and record current CSV

```bash
NS=<operator-namespace>
SUB=<subscription-name>

oc get subscription/$SUB -n $NS -o yaml
oc get subscription/$SUB -n $NS \
  -o jsonpath='{.status.currentCSV}'
echo
oc get csv -n $NS
oc api-resources --api-group='<operator-api-group>'
```

Record the real CSV name. Do not assume it equals the Subscription name.

### Step 2: Handle operands according to the product procedure

List custom resources and determine whether deleting them destroys data or external infrastructure. Some Operators require a finalizer-driven shutdown while the controller is still running. Do not delete CRDs first.

### Step 3: Delete Subscription, then CSV

```bash
CSV=$(oc get subscription/$SUB -n $NS -o jsonpath='{.status.currentCSV}')

oc delete subscription/$SUB -n $NS
oc delete csv/$CSV -n $NS
```

Deleting only the Subscription stops future updates but can leave the current CSV and Operator deployment installed. Deleting the CSV removes the installed Operator version's OLM-managed deployment and related installation resources.

### Step 4: Verify

```bash
oc get subscription,csv,installplan -n $NS
oc get pods -n $NS
oc get crd | grep -i '<operator-keyword>'
```

CRDs and custom resources can remain by design. Delete them only if the task explicitly requires it, no other Operator/operand uses them, and data-loss consequences are understood.

### Web console path

Go to **Operators → Installed Operators**, open the Operator, choose **Actions → Uninstall Operator**, and confirm. Then verify resources with the CLI. The console warning that managed resources and CRDs can remain is important, not optional reading.

### “Uninstall” versus “delete” in a task

The classic OpenShift documentation often calls the full Subscription-plus-CSV procedure “deleting an Operator.” Exam wording can distinguish stopping updates, uninstalling the controller, and deleting remaining resources. Read the required end state:

- no updates only: remove or change the Subscription as requested;
- no installed Operator: remove Subscription and current CSV;
- no operands or APIs: additionally follow the Operator-specific cleanup order, but never delete shared CRDs blindly.

---

## 9. Custom CatalogSource: Correct Schema

Custom catalogs are not required for every EX280 task, but the schema is useful for troubleshooting. A gRPC catalog uses `spec.sourceType`, not `spec.type`, and does not use invented `grpcPodName` fields.

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: lab-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: registry.example.com/olm/lab-index:v4.18
  displayName: Lab Catalog
  publisher: Lab Team
  updateStrategy:
    registryPoll:
      interval: 30m
```

```bash
oc apply -f catalogsource.yaml
oc get catalogsource/lab-catalog -n openshift-marketplace -o yaml
oc get pod -n openshift-marketplace
```

Do not replace default Red Hat catalog images with guessed internal registry paths.

---

## 10. Troubleshooting Decision Tree

### No package appears

```bash
oc get catalogsource -n openshift-marketplace
oc describe catalogsource/<source> -n openshift-marketplace
oc get packagemanifest -n openshift-marketplace | grep -i '<name>'
oc get pods -n openshift-marketplace
```

Check catalog connection state, catalog pod image pulls, proxy/trust configuration, and whether the package exists in that catalog and cluster version.

### No InstallPlan appears

```bash
oc describe subscription/<name> -n <namespace>
oc get operatorgroup -n <namespace> -o yaml
oc get events -n <namespace> --sort-by=.lastTimestamp
```

Likely causes include an incorrect package/channel/source, an unhealthy catalog, dependency resolution failure, or missing/incompatible OperatorGroup.

### InstallPlan is not progressing

```bash
oc get installplan -n <namespace>
oc describe installplan/<name> -n <namespace>
```

If approval is `Manual` and `APPROVED` is false, inspect then patch `spec.approved: true`. If already approved, read failed plan steps and events.

### CSV is `Pending` or `Failed`

```bash
oc describe csv/<name> -n <namespace>
oc get operatorgroup -n <namespace> -o yaml
oc auth can-i create deployments -n <namespace>
oc get events -n <namespace> --sort-by=.lastTimestamp
```

Check unsupported install mode, conflicting CRD ownership, missing APIs, insufficient permissions, or failed deployment creation.

### CSV is `Succeeded`, but the Operator pod is unhealthy

```bash
oc get deployment,pod -n <namespace>
oc describe pod/<pod-name> -n <namespace>
oc logs deployment/<operator-deployment> -n <namespace> --all-containers
```

Check image pulls, SCC admission, node scheduling, resource limits, configuration Secrets/ConfigMaps, and runtime errors.

---

## 11. Hands-On Exam Lab

Use an Operator that is available in your lab's catalog and is safe to install.

1. Find its package manifest and record package, source, source namespace, channel, current CSV, and supported install mode.
2. Create a dedicated namespace.
3. Create exactly one compatible OperatorGroup.
4. Install with `installPlanApproval: Manual`.
5. Find and approve the InstallPlan.
6. Prove the CSV is `Succeeded` and the Operator pod is ready.
7. Record any CRDs introduced by the Operator.
8. Remove the Subscription and current CSV.
9. Prove the Operator deployment is gone and report which CRDs/CRs remain.

Do not use an Operator that manages valuable data for this lab.

---

## 12. Exam Traps

- `oc get clusteroperators` is not the inventory of optional OperatorHub installations.
- Do not guess package or channel names; inspect `PackageManifest`.
- Only one OperatorGroup is allowed in an installation namespace.
- An all-namespaces OperatorGroup omits selectors; `targetNamespaces: ["*"]` is wrong.
- `Subscription.status.phase` is not the success test. Inspect conditions and the CSV phase.
- There is no standard `oc approve installplan`; patch `spec.approved`.
- Deleting a Subscription alone can leave the installed CSV running.
- Uninstalling an Operator normally leaves CRDs, custom resources, and managed data.
- A CSV is namespaced despite its name.
- Use `spec.sourceType: grpc` in `CatalogSource`, not `spec.type: grpc`.

---

## 13. Quick Reference

| Task | Command |
|---|---|
| List catalogs | `oc get catalogsource -n openshift-marketplace` |
| Discover packages | `oc get packagemanifest -n openshift-marketplace` |
| Inspect package/channels | `oc describe packagemanifest/NAME -n openshift-marketplace` |
| List OperatorGroups | `oc get operatorgroup -A` |
| List Subscriptions | `oc get subscription -A` |
| List plans | `oc get installplan -n NS` |
| Approve plan | `oc patch installplan/NAME -n NS --type=merge -p '{"spec":{"approved":true}}'` |
| List installed versions | `oc get csv -A` |
| Show current CSV | `oc get subscription/NAME -n NS -o jsonpath='{.status.currentCSV}'` |
| Remove update subscription | `oc delete subscription/NAME -n NS` |
| Remove installed version | `oc delete csv/CSV-NAME -n NS` |

---

## 14. Review Questions and Answers

1. **What does a Subscription select?** A package, channel, catalog source, source namespace, approval mode, and optionally a starting CSV/configuration.
2. **What proves classic OLM installation succeeded?** The intended CSV in the correct namespace reaches `Succeeded`, its deployment/pods are healthy, and required APIs exist.
3. **How do you approve a manual plan?** Patch `InstallPlan.spec.approved` to `true`.
4. **What happens if you delete only the Subscription?** Future updates stop, but the current CSV and Operator can remain installed.
5. **Does uninstall delete CRDs and custom resources?** Normally no. Inventory and clean them separately only when safe and requested.
6. **How do you target all namespaces in an OperatorGroup?** Omit both `spec.targetNamespaces` and `spec.selector`, provided the Operator supports `AllNamespaces`.
7. **Why can an OperatorGroup fail an installation?** Its target set can be incompatible with install modes supported by the CSV.
8. **What resource shows package channels before installation?** `PackageManifest`.

---

## Verified References

- EX280 objectives: <https://www.redhat.com/en/services/training/red-hat-certified-openshift-administrator-exam>
- OpenShift 4.18 Operators guide: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/operators/index>
- OLM concepts and OperatorGroups: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/operators/understanding-operators>
- Installing and removing Operators: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/operators/administrator-tasks>
