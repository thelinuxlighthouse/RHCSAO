# Chapter 8: Manage OpenShift Operators

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand the Operator Framework and Operator Lifecycle Manager (OLM)
- Install operators using OperatorHub
- Uninstall and delete operators from the cluster
- Configure operator installation and subscription parameters
- Monitor operator status and health
- Troubleshoot common operator installation and runtime issues
- Manage operator subscriptions and updates
- Understand operator channel selection and version management
- Apply operator management best practices in production

---

## OpenShift Concepts

### What is an Operator?

An Operator is a Kubernetes-native application that automates the installation, configuration, and management of complex software on Kubernetes. Operators encapsulate domain-specific knowledge into custom controllers that continuously reconcile the cluster state with the desired state.

**Examples of Operators:**
- **Database Operators**: PostgreSQL, MySQL, MongoDB management
- **Message Queue Operators**: Kafka, RabbitMQ, ActiveMQ
- **Cache Operators**: Redis, Memcached
- **Storage Operators**: Ceph, NFS, GlusterFS
- **Monitoring Operators**: Prometheus, Grafana, Elasticsearch
- **CI/CD Operators**: Jenkins, Tekton

### Operator Lifecycle Manager (OLM)

OLM is the framework that manages operators in OpenShift and Kubernetes. It provides:

- **OperatorHub**: Catalog of available operators
- **CatalogSource**: Source of operator packages
- **OperatorGroup**: Defines which namespaces an operator manages
- **Subscription**: Tracks operator installation and updates
- **CatalogSource**: Points to operator package repository
- **InstallPlan**: Records the steps to install an operator
- **ClusterServiceVersion (CSV)**: Operator package manifest

### Operator Lifecycle

An operator goes through several lifecycle states:

1. **Pending**: Operator package is downloading
2. **Installing**: Operator is being installed
3. **Installed**: Operator is installed but not yet running
4. **Running**: Operator is running and managing resources
5. **Upgradeable**: New version is available
6. **Upgrading**: Operator is upgrading
7. **Succeeded**: Operator completed successfully
8. **Failed**: Operator encountered an error
9. **Uninstalling**: Operator is being removed
10. **Uninstalled**: Operator has been removed

### OperatorHub

OperatorHub is the OpenShift catalog of available operators. It provides:

- **Search**: Find operators by name, description, or tags
- **Filter**: Filter by category, provider, or status
- **Preview**: View operator details before installation
- **Install**: One-click operator installation

Operators in OperatorHub are organized into categories:

- **Database**: PostgreSQL, MySQL, Redis
- **Message Queue**: Kafka, RabbitMQ
- **Cache**: Memcached, Redis
- **Monitoring**: Prometheus, Grafana
- **CI/CD**: Jenkins, Tekton
- **Storage**: Ceph, NFS
- **Networking**: OpenShift SDN, OVN
- **Enterprise**: Red Hat, VMware, SAP

### OperatorGroup

An OperatorGroup defines which namespaces an operator can manage. It specifies:

- **Target namespaces**: Namespaces the operator can manage
- **Selector**: Filter operators by label
- **Scope**: Cluster-wide or namespace-specific

OperatorGroups are required for operators to manage resources in namespaces.

---

## Architecture and Internal Operation

### Operator Lifecycle Manager Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Operator Lifecycle Manager                  │
├─────────────────────────────────────────────────────────────┤
│  CatalogSource → Catalog → ClusterServiceVersion (CSV)      │
│         │                    │                              │
│         ▼                    ▼                              │
│  ┌──────────────┐    ┌───────────────────┐                  │
│  │  OperatorHub │    │  CatalogSource    │                  │ 
│  │ (Registry)   │◄───┼────┐              │                  │
│  └──────────────┘    │    │              │                  │
│                      │    ▼              │                  │
│                      │  ┌──────────────┐ │                  │
│                      │  │  Subscription│ │                  │
│                      │  └──────┬───────┘ │                  │
│                      │         │         │                  │
│                      │         ▼         │                  │
│                      │  ┌──────────────┐ │                  │
│                      │  │  InstallPlan │ │                  │
│                      │  └──────┬───────┘ │                  │
│                      │         │         │                  │
│                      │         ▼         │                  │
│                      │  ┌──────────────┐ │                  │
│                      │  │  Operator    │ │                  │
│                      │  │  Deployment  │ │                  │
│                      │  └──────┬───────┘ │                  │
│                      │         │         │                  │
│                      └─────────┴─────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

### Operator Installation Flow

```
1. User selects operator from OperatorHub
        │
        v
2. Create CatalogSource pointing to operator repository
        │
        v
3. Create OperatorGroup for target namespace(s)
        │
        v
4. Create Subscription to install operator
        │
        v
5. OLM validates Subscription against OperatorGroup
        │
        v
6. OLM creates InstallPlan
        │
        v
7. Cluster approves InstallPlan
        │
        v
8. OLM creates Operator Deployment
        │
        v
9. Operator becomes Running
```

### Subscription State Machine

```
Subscription: Pending → InstallPlan → Approved → Operator → Running
                                  ↓
                               Failed → Reconciliation
```

When a subscription is created:
1. OLM creates an InstallPlan with required permissions and resources
2. The cluster approves the InstallPlan
3. OLM creates the Operator Deployment
4. The Operator starts managing resources

---

## Components and Terminology

| Term | Description |
|------|-------------|
| Operator | Software that automates application lifecycle management |
| OLM | Operator Lifecycle Manager framework |
| OperatorHub | Catalog of available operators |
| CatalogSource | Reference to operator package repository |
| OperatorGroup | Defines namespaces an operator can manage |
| Subscription | Tracks operator installation and updates |
| ClusterServiceVersion (CSV) | Operator package manifest |
| InstallPlan | Records installation steps |
| ApprovalMode | Automatic or manual approval of InstallPlans |
| Channel | Operator version stream (alpha, stable, etc.) |
| Package | Group of related operators |
| StartOfWeek | Schedule for automatic operator updates |
| ImageDigestMirror | Mirror operator images to different registry |
| MirrorSelector | Filter which operators to mirror |
| ImageContentSourcePolicy | References external image sources |

---

## Administration Tasks

### Installing Operators Using OperatorHub

#### Accessing OperatorHub via Web Console

1. Navigate to **Operators → OperatorHub**
2. Search for desired operator
3. Select **Install** button
4. Configure installation parameters
5. Click **Install Operator**

#### Installing Operators via CLI

```bash
# Search for operators
oc get pods -A | grep -i kafka

# Install operator via CLI
oc apply -f operator-install.yaml
```

#### Installing via OperatorHub CLI Plugin

```bash
# Search for operators
operatorhub.io search kafka

# Install operator
operatorhub.io install kafka --namespace=openshift-kafka
```

### Configuring CatalogSources

#### Creating a CatalogSource

```bash
oc apply -f catalogsource.yaml
```

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: community-operators
  namespace: openshift-marketplace
spec:
  type: grpc
  image: registry-proxy.engineering.redhat.com/rh-osbs/openshift-operators-operatorhub:v4.18
  grpcPodName: openshift-marketplace
  grpcServiceName: openshift-marketplace
```

#### Listing CatalogSources

```bash
oc get catalogsources -n openshift-marketplace
oc get catalogsources -A
```

#### Editing CatalogSource

```bash
oc edit catalogsources -n openshift-marketplace
```

### Creating OperatorGroups

#### Creating an OperatorGroup

```bash
oc apply -f operatorgroup.yaml
```

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: my-operators
  namespace: myapp-project
spec:
  targetNamespaces:
    - myapp-project
```

#### Creating OperatorGroup for Multiple Namespaces

```yaml
spec:
  targetNamespaces:
    - myapp-project
    - database-project
    - monitoring-project
```

#### Creating Cluster-Wide OperatorGroup

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cluster-operators
  namespace: openshift-operators
spec:
  targetNamespaces:
    - '*'
```

#### Listing OperatorGroups

```bash
oc get operatorgroups
oc get operatorgroups -A
```

### Installing Operators with Subscription

#### Installing PostgreSQL Operator

```bash
oc apply -f subscription-postgresql.yaml
```

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: postgresql
  namespace: openshift-postgresql
spec:
  channel: community-4.5
  name: postgresql
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

#### Installing Kafka Operator

```bash
oc apply -f subscription-kafka.yaml
```

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kafka
  namespace: openshift-kafka
spec:
  channel: stable-2.4
  name: kafka
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

#### Installing with Custom Parameters

```bash
oc apply -f subscription-with-params.yaml
```

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kafka
  namespace: openshift-kafka
spec:
  channel: stable-2.4
  name: kafka
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
  config:
    params:
      - name: REPLICATION_FACTOR
        value: "3"
      - name: KAFKA_HEAP_OPTS
        value: "-Xmx2G -Xms2G"
```

#### Viewing Operator Status

```bash
oc get subscriptions -A
oc get subscriptions -A -o wide
oc get subscriptions postgresql -n openshift-postgresql
oc describe subscription postgresql -n openshift-postgresql
```

### Managing Operator Updates

#### Checking Operator Update Status

```bash
oc get subscriptions postgresql -n openshift-postgresql -o jsonpath='{.status.phase}'
oc get subscriptions postgresql -n openshift-postgresql -o jsonpath='{.status.installedCSV}'
```

#### Approving InstallPlan Manually

```bash
oc get installplans -n openshift-postgresql
oc approve installplan <installplan-name> -n openshift-postgresql
```

#### Uninstalling an Operator

```bash
# Delete the subscription
oc delete subscription postgresql -n openshift-postgresql

# Verify uninstallation
oc get subscriptions -n openshift-postgresql
```

#### Deleting an Operator

```bash
# Delete the operator deployment
oc delete deployment -n openshift-postgresql | grep -i postgresql

# Delete operator-related resources
oc delete all -n openshift-postgresql
```

---

## YAML Examples

### CatalogSource

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: redhat-operators
  namespace: openshift-marketplace
spec:
  type: grpc
  image: registry-proxy.engineering.redhat.com/rh-osbs/openshift-operators-operatorhub:v4.18
  grpcPodName: openshift-marketplace
  grpcServiceName: openshift-marketplace
  secrets:
    - name: operator-source-credentials
```

### CatalogSource with Mirror

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: redhat-operators-mirror
  namespace: openshift-marketplace
spec:
  type: grpc
  image: registry.example.com/mirrored-operators:v4.18
  grpcPodName: openshift-marketplace
  grpcServiceName: openshift-marketplace
```

### ImageContentSourcePolicy

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: operator-images
spec:
  imageDigests:
    - mirrorSources:
        - image: registry.example.com/mirrored-openshift-postgresql:v4.5.0
      sources:
        - image: registry.redhat.io/openshift4/ose-postgresql@sha256:abc123...
```

### OperatorGroup

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: my-operators
  namespace: myapp-project
spec:
  targetNamespaces:
    - myapp-project
```

### OperatorGroup for Multiple Namespaces

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: multi-namespace-operators
  namespace: openshift-operators
spec:
  targetNamespaces:
    - myapp-project
    - database-project
    - monitoring-project
```

### Subscription

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: postgresql
  namespace: openshift-postgresql
spec:
  channel: community-4.5
  installPlanApproval: Automatic
  name: postgresql
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

### Subscription with Configuration

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kafka
  namespace: openshift-kafka
spec:
  channel: stable-2.4
  installPlanApproval: Automatic
  name: kafka
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  config:
    params:
      - name: REPLICATION_FACTOR
        value: "3"
      - name: KAFKA_HEAP_OPTS
        value: "-Xmx2G -Xms2G"
      - name: CLUSTER_ID
        value: "kafka-cluster"
```

### Subscription with Custom Channel

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: redis
  namespace: openshift-redis
spec:
  channel: fast
  installPlanApproval: Automatic
  name: redis
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

### InstallPlan

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: InstallPlan
metadata:
  name: installplan-abc123
  namespace: openshift-postgresql
spec:
  approved: true
  clusterServiceVersions:
    - name: postgresql
      namespace: openshift-postgresql
      installedCSV: postgresql.v4.5.0
      startNode: 0
      status: InProgress
      version: 4.5.0
  generation: 1
```

---

## Explanation of YAML

### CatalogSource

The CatalogSource points to an operator package repository:

- **`spec.type: grpc`**: Uses gRPC for communication with the OperatorHub registry
- **`spec.image`**: Registry address of the catalog source
- **`spec.grpcPodName`** and **`spec.grpcServiceName`**: Names of the pods and services that serve the catalog
- **`spec.secrets`**: Credentials for private registries

### OperatorGroup

The OperatorGroup defines which namespaces an operator can manage:

- **`spec.targetNamespaces`**: List of namespaces to manage
- **`'*'`**: Wildcard for all namespaces (cluster-wide)
- Required for operators to install resources in namespaces

### Subscription

The Subscription tracks operator installation and updates:

- **`spec.channel`**: Version stream (alpha, stable, fast)
- **`spec.name`**: Name of the operator package
- **`spec.source`**: CatalogSource name
- **`spec.sourceNamespace`**: Namespace containing the CatalogSource
- **`spec.installPlanApproval`**: Automatic or manual approval
- **`spec.config.params`**: Custom operator configuration parameters

---

## Verification Procedures

### Verify CatalogSource

```bash
oc get catalogsources -n openshift-marketplace
oc describe catalogsources redhat-operators -n openshift-marketplace
```

Expected: CatalogSource shows healthy status.

### Verify OperatorGroup

```bash
oc get operatorgroups -n myapp-project
oc describe operatorgroups my-operators -n myapp-project
```

Expected: OperatorGroup shows target namespaces.

### Verify Operator Subscription

```bash
oc get subscriptions -n openshift-postgresql
oc describe subscription postgresql -n openshift-postgresql
```

Expected: Subscription shows `phase: Installed` or `Running`.

### Verify Operator Deployment

```bash
oc get deployments -n openshift-postgresql
oc get pods -n openshift-postgresql
```

Expected: Operator pods are Running.

### Verify Operator Status

```bash
oc get subscriptions -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

Expected: All subscriptions show healthy phases.

### Verify OperatorHub Access

```bash
oc get pods -n openshift-marketplace
```

Expected: Marketplace pods are Running.

---

## Troubleshooting

### Operator Installation Fails

**Symptom**: Subscription stuck in `InstallPlan` phase.

**Diagnosis**:

```bash
oc get subscriptions <name> -n <namespace> -o yaml
oc get installplans -n <namespace>
oc describe installplan <name> -n <namespace>
```

**Resolution**:
- Check InstallPlan for errors
- Verify OperatorGroup exists in target namespace
- Check cluster permissions
- Review operator logs

### Operator Not Running

**Symptom**: Subscription shows `Installed` but operator pods not Running.

**Diagnosis**:

```bash
oc get pods -n <namespace>
oc logs -n <namespace> deploy/<operator-name>
oc describe pod -n <namespace> <operator-pod>
```

**Resolution**:
- Check operator deployment configuration
- Verify resource limits are not exceeded
- Check SCC (Security Context Constraints)
- Review operator logs for errors

### Operator Update Fails

**Symptom**: Subscription stuck in `Upgrading` phase.

**Diagnosis**:

```bash
oc get subscriptions <name> -n <namespace>
oc get installplans -n <namespace>
oc describe subscription <name> -n <namespace>
```

**Resolution**:
- Check InstallPlan for errors
- Approve InstallPlan manually if needed
- Review operator upgrade compatibility
- Check for breaking changes in new version

### OperatorHub Not Working

**Symptom**: Cannot search or install operators.

**Diagnosis**:

```bash
oc get pods -n openshift-marketplace
oc logs -n openshift-marketplace deploy/openshift-marketplace
```

**Resolution**:
- Check marketplace pod status
- Verify CatalogSource is configured
- Check network connectivity to registry
- Review operator logs

### Subscription Requiring Manual Approval

**Symptom**: InstallPlan not automatically approved.

**Diagnosis**:

```bash
oc get subscriptions <name> -n <namespace> -o yaml | grep installPlanApproval
oc get installplans -n <namespace>
```

**Resolution**:
- Set `installPlanApproval: Automatic` in subscription
- Or manually approve: `oc approve installplan <name>`

### Operator Channel Not Available

**Symptom**: Channel selection fails or shows no versions.

**Diagnosis**:

```bash
oc get subscriptions <name> -n <namespace> -o yaml | grep channel
oc get catalogsources -n openshift-marketplace
```

**Resolution**:
- Check available channels in OperatorHub
- Use supported channel for your cluster version
- Verify CatalogSource is up to date

---

## Production Best Practices

### Operator Installation Best Practices

- **Use Automatic InstallPlan approval**: Reduces manual intervention
- **Test in staging first**: Validate operator functionality before production
- **Monitor operator health**: Set up alerts for operator failures
- **Document operator dependencies**: Track required OperatorGroups
- **Plan version upgrades**: Schedule operator updates during maintenance windows

### Subscription Configuration Best Practices

- **Select appropriate channel**: Use stable channel for production
- **Configure custom parameters carefully**: Test parameter combinations
- **Monitor resource usage**: Operators consume cluster resources
- **Set resource limits**: Prevent operator resource monopolization
- **Enable logging**: Monitor operator activity for troubleshooting

### Operator Management Best Practices

- **Regular audits**: Review installed operators and subscriptions
- **Update regularly**: Keep operators current with security patches
- **Document operator configurations**: Maintain operator parameter documentation
- **Monitor operator metrics**: Track operator health and performance
- **Plan rollback procedures**: Know how to revert failed upgrades

### OperatorHub Best Practices

- **Filter by provider**: Prefer Red Hat operators for production
- **Check operator documentation**: Review before installation
- **Verify compatibility**: Ensure operator works with your cluster version
- **Review operator dependencies**: Check required namespaces and permissions
- **Monitor operator updates**: Subscribe to operator release notifications

---

## Certification Exam Notes

### Exam-Relevant Commands You Must Know

```bash
# OperatorHub commands
oc get catalogsources -n openshift-marketplace
oc get operatorgroups
oc get subscriptions -A
oc get installplans

# Operator installation
oc apply -f subscription.yaml
oc apply -f operatorgroup.yaml
oc apply -f catalogsource.yaml

# Operator monitoring
oc get subscriptions <name> -n <namespace>
oc describe subscription <name> -n <namespace>
oc get pods -n <namespace> | grep <operator>

# Operator updates
oc get installplans -n <namespace>
oc approve installplan <name> -n <namespace>

# Operator removal
oc delete subscription <name> -n <namespace>
oc delete operatorgroup <name> -n <namespace>

# Operator configuration
oc edit subscription <name> -n <namespace>
oc edit operatorgroup <name> -n <namespace>
```

### Common Exam Scenarios

1. **"Install an operator from OperatorHub"** → Create CatalogSource, OperatorGroup, Subscription
2. **"Check operator subscription status"** → `oc get subscriptions <name> -n <namespace>`
3. **"View operator installation plan"** → `oc get installplans -n <namespace>`
4. **"Approve an install plan manually"** → `oc approve installplan <name>`
5. **"Uninstall an operator"** → `oc delete subscription <name> -n <namespace>`
6. **"Create an OperatorGroup for a namespace"** → `oc apply -f operatorgroup.yaml`
7. **"Check operator pods"** → `oc get pods -n <namespace>`
8. **"View operator description"** → `oc describe subscription <name>`
9. **"List all operators"** → `oc get subscriptions -A`
10. **"Configure subscription parameters"** → Add `config.params` to Subscription

### Important Exam Details

- OperatorGroup must exist **before** Subscription
- Subscription references CatalogSource **by name and namespace**
- Installation fails if OperatorGroup doesn't exist in target namespace
- `installPlanApproval: Automatic` enables auto-approving InstallPlans
- Operator packages are identified by ClusterServiceVersion (CSV)
- Channels include: alpha, beta, stable, fast, latest
- Operators manage resources in namespaces specified by OperatorGroup

---

## Common Mistakes

### Mistake: Creating Subscription Before OperatorGroup

Creating a Subscription before the OperatorGroup exists causes installation to fail. Always create the OperatorGroup first.

### Mistake: Using Wrong CatalogSource Namespace

CatalogSources must be referenced with their correct namespace. Using `openshift-marketplace` when the CatalogSource is in `openshift-operators` causes failures.

### Mistake: Not Checking Operator Logs

When operators fail, checking logs is essential for diagnosis. Skipping log review leads to extended troubleshooting time.

### Mistake: Ignoring InstallPlan Approval

Manual InstallPlan approval delays operator installation. Use Automatic approval for smoother deployments.

### Mistake: Using Alpha Channel in Production

Alpha channels are unstable and may break. Use stable channels for production environments.

### Mistake: Forgetting to Create Namespaces

Operators create resources in namespaces specified by OperatorGroup. Ensure namespaces exist or configure operators to create them.

### Mistake: Not Monitoring Operator Health

Operators can silently fail. Regular health checks prevent issues from going unnoticed.

### Mistake: Ignoring Operator Dependencies

Some operators have dependencies on other operators. Review operator documentation for required dependencies.

---

## Chapter Summary

This chapter covered managing OpenShift operators. You learned the Operator Lifecycle Manager (OLM) framework and its components including CatalogSources, OperatorGroups, Subscriptions, and InstallPlans. You installed operators using OperatorHub and configured subscriptions with custom parameters. You monitored operator status and health, checking subscription phases, operator pods, and install plans. You troubleshooted common operator installation failures, update issues, and runtime problems. You managed operator updates and uninstalled operators when needed. Production best practices emphasized testing in staging, monitoring operator health, configuring appropriate channels, and planning version upgrades. You understood the importance of OperatorGroups for namespace management and Automatic InstallPlan approval for streamlined deployments.

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| List CatalogSources | `oc get catalogsources -n openshift-marketplace` |
| Create CatalogSource | `oc apply -f catalogsource.yaml` |
| List OperatorGroups | `oc get operatorgroups` |
| Create OperatorGroup | `oc apply -f operatorgroup.yaml` |
| List Subscriptions | `oc get subscriptions -A` |
| Describe Subscription | `oc describe subscription <name> -n <namespace>` |
| Install Operator | `oc apply -f subscription.yaml` |
| List InstallPlans | `oc get installplans -n <namespace>` |
| Approve InstallPlan | `oc approve installplan <name> -n <namespace>` |
| Uninstall Operator | `oc delete subscription <name> -n <namespace>` |
| Delete OperatorGroup | `oc delete operatorgroup <name> -n <namespace>` |
| Check Operator Pods | `oc get pods -n <namespace>` |
| View Operator Status | `oc get subscriptions <name> -n <namespace> -o jsonpath='{.status.phase}'` |

### Operator Lifecycle Phases

| Phase | Description |
|-------|-------------|
| Pending | Operator package being downloaded |
| Installing | Operator being installed |
| Installed | Operator installed but not running |
| Running | Operator running and managing resources |
| Upgradeable | New version available |
| Upgrading | Operator upgrading |
| Succeeded | Operator completed successfully |
| Failed | Operator encountered error |
| Uninstalling | Operator being removed |
| Uninstalled | Operator removed |

### Subscription Channels

| Channel | Description |
|---------|-------------|
| alpha | Early access, unstable |
| beta | Testing phase |
| stable | Production-ready (recommended) |
| fast | Latest tested version |
| latest | Most recent version |

### OperatorGroup Target Namespaces

| Value | Description |
|-------|-------------|
| Specific namespace | Operator manages only that namespace |
| `*` (wildcard) | Operator manages all namespaces |
| Multiple namespaces | Operator manages listed namespaces |

### InstallPlan Approval Modes

| Mode | Description |
|------|-------------|
| Automatic | InstallPlans approved automatically |
| Manual | Admin must approve each InstallPlan |

---

## Review Questions

1. What is the purpose of an OperatorGroup?

2. How do you install an operator using OperatorHub?

3. What is the difference between a CatalogSource and a Subscription?

4. What happens when you create a Subscription before an OperatorGroup?

5. How do you check the status of an operator subscription?

6. What is an InstallPlan and when is it created?

7. How do you approve an InstallPlan manually?

8. What is the purpose of the `channel` field in a Subscription?

9. How do you uninstall an operator from the cluster?

10. What is the difference between `installPlanApproval: Automatic` and `Manual`?

11. How do you view the pods belonging to an operator?

12. What is a ClusterServiceVersion (CSV)?

13. How do you configure custom parameters for an operator?

14. What namespace should CatalogSources typically be created in?

15. How do you list all installed operators across the cluster?

---

## Answers

1. An OperatorGroup defines which namespaces an operator can manage. It specifies the target namespaces and is required before a Subscription can install an operator in those namespaces.

2. Navigate to **Operators → OperatorHub**, search for the operator, click **Install**, configure installation parameters, and click **Install Operator**.

3. A **CatalogSource** is a reference to an operator package repository (like OperatorHub). A **Subscription** tracks the installation and updates of a specific operator from a CatalogSource.

4. The Subscription will fail because the OperatorGroup doesn't exist. Operators require an OperatorGroup to know which namespaces to manage.

5. `oc get subscriptions <name> -n <namespace>` or `oc describe subscription <name> -n <namespace>`

6. An InstallPlan is created when a Subscription is approved. It records the steps and resources needed to install or update the operator.

7. `oc approve installplan <name> -n <namespace>`

8. The `channel` field specifies which version stream to subscribe to (alpha, stable, fast, etc.). It controls which operator versions are installed.

9. Delete the Subscription: `oc delete subscription <name> -n <namespace>`. The operator will uninstall when the InstallPlan is approved.

10. **Automatic** approves InstallPlans immediately without admin intervention. **Manual** requires an admin to approve each InstallPlan with `oc approve`.

11. `oc get pods -n <namespace>` and filter by the operator name, or `oc get pods -n <namespace> | grep <operator-name>`

12. A ClusterServiceVersion (CSV) is the operator package manifest that describes the operator's capabilities, resources, and requirements.

13. Add a `config.params` section to the Subscription YAML with parameter name-value pairs.

14. CatalogSources are typically created in the `openshift-marketplace` namespace.

15. `oc get subscriptions -A` lists all subscriptions across all namespaces, showing installed operators.
