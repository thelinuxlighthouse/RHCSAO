# Chapter 1: Manage OpenShift Container Platform

## Learning Objectives

By the end of this chapter, you will be able to:

- Navigate and use the OpenShift Web Console to manage and configure a cluster
- Use the `oc` command-line interface to manage and configure a cluster
- Create and delete projects (namespaces)
- Locate and examine container images in deployed pods and image streams
- Identify images using tags and SHA-256 digests
- Query, format, and filter attributes of Kubernetes resources using the `oc` CLI
- Import, export, and configure Kubernetes resource manifests
- Examine resources and overall cluster status
- Monitor cluster events and alerts
- View pod and container logs
- Assess the health of an OpenShift cluster
- Troubleshoot common container, pod, and cluster-level events and alerts
- Locate and navigate Red Hat product documentation effectively

---

## OpenShift Concepts

OpenShift Container Platform (OCP) is Red Hat's enterprise Kubernetes distribution. It extends upstream Kubernetes with developer tools, integrated CI/CD pipelines, security hardening, and a unified management console. Understanding OCP requires understanding how it layers capabilities on top of Kubernetes.

### OpenShift vs. Kubernetes

While Kubernetes provides the core container orchestration engine, OpenShift adds:

- **Integrated registry**: Every cluster ships with an internal container image registry
- **Web console**: A unified graphical interface for administrators, developers, and cluster administrators
- **Security Context Constraints (SCCs)**: OpenShift's authorization mechanism that controls what actions a pod can perform, replacing Kubernetes RBAC for pod-level security
- **Routes**: OpenShift's Layer 7 load balancer, built on HAProxy, for exposing applications externally
- **Image Streams**: An OpenShift-specific resource that tracks container images by tag and digest
- **Templates**: Server-side parameterized application blueprints
- **Operators**: Packaged, deployable, and manageable operators for automated cluster operations

### The OpenShift Cluster Model

An OpenShift cluster consists of three node categories:

1. **Control Plane Nodes**: Run the Kubernetes API server, etcd, scheduler, and controller manager. In OpenShift 4.18, these nodes are automatically provisioned and managed by the Machine Config Operator (MCO).
2. **Compute (Worker) Nodes**: Run user workloads — pods, containers, and applications.
3. **Infrastructure Nodes**: Dedicated nodes running cluster infrastructure components such as the ingress controller, monitoring stack, and container registry.

OpenShift 4 uses a self-hosted etcd model where the etcd cluster runs as static pods on control plane nodes, managed by the etcd operator.

### The Operator Framework

OpenShift is built on the Operator Framework. Nearly every cluster capability is managed by an operator:

- **Cluster Version Operator (CVO)**: Manages cluster-wide upgrades and component versioning
- **Machine Config Operator (MCO)**: Manages node configuration, kernel parameters, and systemd units
- **Cluster Network Operator**: Manages the CNI plugin (OVNKubernetes by default)
- **Ingress Operator**: Manages the HAProxy-based ingress controller
- **Monitoring Operator**: Deploys and manages Prometheus, Grafana, and Alertmanager

Operators are the primary mechanism by which OpenShift achieves its "self-healing" and "self-upgrading" characteristics. When a component drifts from its desired state, the operator reconciles it back.

---

## Architecture and Internal Operation

### Control Plane Architecture

The OpenShift control plane runs as a set of highly available services. Each critical component runs multiple replicas across control plane nodes:

```
Control Plane Node 1          Control Plane Node 2          Control Plane Node 3
├── kube-apiserver            ├── kube-apiserver            ├── kube-apiserver
├── kube-controller-manager   ├── kube-controller-manager   ├── kube-controller-manager
├── kube-scheduler            ├── kube-scheduler            ├── kube-scheduler
├── etcd (static pod)         ├── etcd (static pod)         ├── etcd (static pod)
├── OpenShift API Server      ├── OpenShift API Server      ├── OpenShift API Server
└── Machine Config Server     └── Machine Config Server     └── Machine Config Server
```

The Kubernetes API server is the central entry point for all cluster operations. It exposes the standard Kubernetes API and the OpenShift-specific API extensions.

### Container Network Interface (CNI)

OpenShift 4.18 uses **OVNKubernetes** as the default CNI plugin. OVNKubernetes provides:

- Software-defined networking (SDN) using Open vSwitch (OVS)
- Encapsulated pod-to-pod communication across nodes
- Network policy enforcement at the OVS level
- Multi-network support through NetworkAttachmentDefinition

Each node runs an OVS daemon that programs flow rules to route pod traffic. Pod IP addresses are allocated from subnets assigned to each node.

### Image Registry

The cluster-internal registry is deployed at `image-registry.default-registry.svc:5000`. It stores application images pulled during deployments and images pushed by developers. The registry persists images to PersistentVolumeClaims provisioned by the cluster's storage class.

### Authentication Flow

When a user authenticates to OpenShift, the following components participate:

1. The **OAuth Proxy** intercepts authentication requests
2. The configured **Identity Provider** (HTPasswd, LDAP, Keystone, GitHub, etc.) validates credentials
3. Upon successful authentication, an **OAuth token** is issued
4. The token is exchanged for a **Kubernetes service account token** for API access

---

## Components and Terminology

| Term | Description |
|------|-------------|
| `oc` | OpenShift CLI tool, a fork of `kubectl` with OpenShift-specific extensions |
| Project | OpenShift's term for a Kubernetes Namespace, with added quota and template support |
| Image Stream | OpenShift-specific resource that tracks container images by tag and digest |
| Route | OpenShift's Layer 7 load balancer for exposing services externally |
| SCC | Security Context Constraint — OpenShift's pod-level security policy |
| Operator | A Kubernetes-native application that extends the API and automates management |
| CVO | Cluster Version Operator — manages upgrades and component versions |
| MCO | Machine Config Operator — manages node-level configuration |
| OVNKubernetes | Default CNI plugin providing software-defined networking |
| HTPasswd | Simple file-based identity provider for authentication |
| ClusterRole | Cluster-wide set of permissions |
| ClusterRoleBinding | Binds a ClusterRole to a user, group, or service account |
| Node | A worker or control plane machine in the cluster |
| MachineSet | Defines the desired number and configuration of nodes |

---

## Administration Tasks

### Authenticating to the Cluster

Before performing any administrative task, you must authenticate to the cluster.

#### Using the `oc` CLI

```bash
oc login --insecure-skip-tls-verify=true https://api.cluster.example.com:6443 \
  -u administrator -p password123
```

The `--insecure-skip-tls-verify` flag is commonly needed in exam environments where certificates use internal DNS names that cannot be resolved externally.

To log in as a system administrator with full cluster privileges:

```bash
oc login -u kubeadmin -p <kubeadmin-password>
```

The `kubeadmin` user is a cluster-admin user created during initial cluster installation.

#### Verifying Authentication

```bash
oc whoami
oc whoami --show-token
```

### Accessing the Web Console

The OpenShift Web Console is accessible at:

```
https://console-openshift-console.apps.cluster.example.com
```

The console has two views:

- **Administrator View**: Full cluster management, node inspection, operator management, and global configuration
- **Developer View**: Project-scoped application development, deployment pipelines, and topology visualization

Switch between views using the toggle in the top-right corner of the console.

### Creating and Deleting Projects

In OpenShift, a **project** is a namespace with additional metadata, quota defaults, and template associations.

#### Creating a Project via CLI

```bash
oc new-project myapp-project
```

This command creates the namespace and automatically adds the current user as the full administrator of that project.

#### Creating a Project via Console

1. Navigate to **Administrator View**
2. Go to **Home → Projects → Create Project**
3. Enter the project name and optional description
4. Click **Create**

#### Deleting a Project

```bash
oc delete project myapp-project
```

Deleting a project removes all resources within that namespace. This is irreversible. In the console, navigate to **Home → Projects**, select the project, and choose **Delete Project** from the kebab menu.

#### Examining Current Project

```bash
oc project
oc project -q
```

The `-q` flag outputs only the project name, useful for scripting.

### Locating and Examining Container Images

#### Finding Images in Running Pods

```bash
oc get pods -o wide
```

The `--wide` output includes the NODE and other extended fields, but image details are in the pod spec:

```bash
oc get pod <pod-name> -o yaml | grep -A 5 "containers:"
```

Or more precisely:

```bash
oc get pod <pod-name> -o jsonpath='{.spec.containers[*].image}'
```

#### Examining Image Streams

Image Streams track images within the cluster registry:

```bash
oc get istag -n openshift
oc describe istag nginx:latest -n openshift
```

The `istag` shorthand retrieves ImageStreamTags, which show the tag-to-digest mapping.

### Identifying Images Using Tags and Digests

A container image can be referenced by:

- **Tag**: A mutable label such as `nginx:1.25`
- **Digest**: An immutable SHA-256 hash such as `nginx@sha256:a1b2c3d4e5f6...`

To view the digest for a tagged image:

```bash
oc get istag nginx:latest -o jsonpath='{.image.dockerImageReference}'
```

To pin a deployment to a specific digest (immutable reference):

```bash
oc set image deployment/myapp myapp=registry.example.com/myapp@sha256:abcdef1234567890
```

Using digests ensures that deployments cannot silently change if a tag is updated in the registry.

### Querying, Formatting, and Filtering Resources

The `oc get` command supports multiple output formats and filtering mechanisms.

#### Output Formats

```bash
# Standard table output
oc get pods

# YAML output
oc get pods -o yaml

# JSON output
oc get pods -o json

# JSONPath templating
oc get pods -o jsonpath='{.items[*].metadata.name}'

# Custom columns
oc get pods -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image,STATUS:.status.phase
```

#### Label Selectors

```bash
# Exact match
oc get pods -l app=nginx

# Set-based selector (key exists)
oc get pods -l app in (nginx,redis)

# Negative selector
oc get pods -l app!=nginx
```

#### Field Selectors

```bash
# Filter by field value
oc get pods --field-selector=status.phase=Running
oc get pods --field-selector=status.phase!=Failed
```

#### Name or Prefix Filtering

```bash
# By name prefix
oc get pods -n openshift-logging -o name

# By partial name
oc get all | grep nginx
```

#### Combining Filters

```bash
oc get pods -l app=nginx --field-selector=status.phase=Running -o wide
```

### Importing, Exporting, and Configuring Resources

#### Exporting Resources

```bash
# Export a single resource
oc get deployment myapp -o yaml > myapp-deployment.yaml

# Export all resources in a project
oc get all -o yaml > all-resources.yaml

# Export with --export flag (deprecated in newer versions, use -o yaml instead)
oc get deployment myapp -o yaml --export
```

#### Importing Resources

```bash
# Apply a manifest
oc apply -f myapp-deployment.yaml

# Apply and create if not exists
oc create -f myapp-deployment.yaml --dry-run=client -o yaml | oc apply -f -
```

#### Modifying Resources

```bash
# Edit a resource interactively
oc edit deployment/myapp

# Scale a deployment
oc scale deployment/myapp --replicas=3

# Update an image
oc set image deployment/myapp myapp=registry.example.com/myapp:v2.0
```

### Examining Resources and Cluster Status

#### Cluster Version and Status

```bash
oc get clusterversion
oc get clusterversion -o yaml
```

The ClusterVersion object shows the current version, desired version, and available updates.

#### Cluster Operators

```bash
oc get clusteroperators
```

Each operator should show `True` for the `AVAILABLE`, `PROGRESSING`, and `DEGRADED` conditions (with `PROGRESSING` and `DEGRADED` showing `False`).

#### Nodes

```bash
oc get nodes
oc get nodes -o wide
oc describe node <node-name>
```

#### All Resources Summary

```bash
oc get all -A
```

The `-A` flag is shorthand for `--all-namespaces`.

### Monitoring Cluster Events and Alerts

#### Events

Events are transient records of changes in the cluster:

```bash
# Events in current project
oc get events

# Events across all namespaces
oc get events -A

# Sort events by timestamp
oc get events -A --sort-by=.lastTimestamp

# Events for a specific resource
oc get events --field-selector involvedObject.name=myapp-pod
```

#### Alerts via Console

1. Navigate to **Administrator View**
2. Go to **Administration → Cluster Monitoring → Alerts**
3. Filter by severity (critical, warning, info)
4. Click an alert for details and recommended actions

### Viewing Logs

#### Pod Logs

```bash
# View logs for a pod
oc logs <pod-name>

# Follow logs in real time
oc logs -f <pod-name>

# View logs from a specific container in a multi-container pod
oc logs <pod-name> -c <container-name>

# View previous crashed container logs
oc logs <pod-name> --previous

# View logs with timestamps
oc logs <pod-name> --timestamps

# View last N lines
oc logs <pod-name> --tail=100
```

#### Container Logs via Console

1. Navigate to **Workloads → Pods**
2. Click the pod name
3. Select the **Logs** tab

### Assessing Cluster Health

#### Quick Health Check

```bash
# Check cluster operators
oc get clusteroperators

# Check nodes
oc get nodes

# Check pods in openshift-* namespaces
oc get pods -A | grep -E "Error|CrashLoopBackOff|Pending|Evicted"

# Check cluster version
oc get clusterversion
```

#### Detailed Health Assessment

```bash
# Check for warning or error events
oc get events -A --field-selector type=Warning

# Check etcd health (requires cluster-admin)
oc get pod -n openshift-etcd -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# Check DNS resolution
oc get pods -n openshift-dns

# Check ingress controller
oc get pods -n openshift-ingress
```

---

## YAML Examples

### Project (Namespace) Manifest

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp-project
  labels:
    name: myapp-project
    purpose: development
```

### Nginx Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: myapp-project
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: registry.redhat.io/ubi9/nginx-124:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
```

### Service Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: myapp-project
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

### Node Export Command

```bash
oc get nodes -o yaml > cluster-nodes.yaml
```

### Resource Export with Labels

```bash
oc get deployment -A -l app=nginx -o yaml > nginx-deployments-all-namespaces.yaml
```

---

## Explanation of YAML

### Namespace Manifest

The Namespace resource is the foundational scoping unit in Kubernetes and OpenShift. The `apiVersion: v1` indicates a core Kubernetes API. The `metadata.name` field is the project identifier. Labels attached to the namespace enable policy enforcement and resource organization.

### Deployment Manifest

The Deployment is a workload resource that manages identical pod replicas. Key fields:

- `spec.replicas`: Desired number of running pod instances
- `spec.selector.matchLabels`: Labels used to identify pods belonging to this deployment
- `spec.template`: The pod template — every pod created by this deployment will match this template
- `spec.template.metadata.labels`: Labels applied to each pod (must match `selector.matchLabels`)
- `spec.template.spec.containers`: Container specifications including image, ports, and resource constraints
- `resources.requests`: Minimum guaranteed resources allocated to the container
- `resources.limits`: Maximum resources the container can consume

### Service Manifest

The Service provides stable network access to a set of pods:

- `spec.selector`: Matches pods by label to include them in the service's endpoint list
- `spec.ports.port`: The port the service listens on
- `spec.ports.targetPort`: The port traffic is forwarded to on the pod
- `type: ClusterIP`: Internal-only access (default type)

---

## Verification Procedures

### Verify Project Creation

```bash
oc get project myapp-project
oc project myapp-project
```

Expected output: The project appears in the list and switching to it succeeds.

### Verify Node Status

```bash
oc get nodes
```

Expected output: All nodes show `STATUS=Ready`.

### Verify Cluster Operators

```bash
oc get clusteroperators
```

Expected output: All operators show `AVAILABLE=True`, `PROGRESSING=False`, `DEGRADED=False`.

### Verify Resource Deployment

```bash
oc get deployment nginx-deployment
oc get pods -l app=nginx
```

Expected output: Deployment shows `READY=3/3` and three pods are in `Running` state.

### Verify Service Connectivity

```bash
oc get svc nginx-service
oc get endpoints nginx-service
```

Expected output: Service has a ClusterIP and endpoints lists pod IPs matching the running pods.

### Verify Image Digest

```bash
oc get istag -n myapp-project
oc get pod <pod-name> -o jsonpath='{.spec.containers[0].image}'
```

Expected output: Image reference includes the digest or tag as specified.

### Verify Logs Are Accessible

```bash
oc logs <pod-name> --tail=5
```

Expected output: Recent log lines from the container are displayed.

---

## Troubleshooting

### Pod Stuck in Pending State

**Cause**: Insufficient resources, node affinity conflicts, or PVC provisioning failures.

**Diagnosis**:

```bash
oc describe pod <pod-name>
oc get events -A --sort-by=.lastTimestamp | tail -20
```

Look for events such as `FailedScheduling` or `FailedMount`. The `describe` output's **Events** section at the bottom reveals the root cause.

**Resolution**:
- If resource-related, check node capacity: `oc adm top nodes`
- If PVC-related, verify storage class: `oc get sc`
- If taint-related, verify node tolerations

### CrashLoopBackOff

**Cause**: The container starts, crashes, and the kubelet restarts it repeatedly.

**Diagnosis**:

```bash
oc logs <pod-name> --previous
oc describe pod <pod-name>
```

Check the **Last State** section in the describe output for the exit code.

**Resolution**:
- Exit code 1: Application error — check logs
- Exit code 137: OOMKilled — increase memory limits
- Exit code 125: Docker/runtime error — check image and SCC
- Exit code 126: Permission denied — check entrypoint permissions

### ImagePullBackOff / ErrImagePull

**Cause**: The container image cannot be pulled from the registry.

**Diagnosis**:

```bash
oc describe pod <pod-name> | grep -A 5 "ImagePullBackOff"
```

**Resolution**:
- Verify image name and tag spelling
- Verify registry connectivity from the node
- Check image pull secrets: `oc get secret -n <project>`
- Verify the SCC allows pulling from the registry

### Node Not Ready

**Cause**: kubelet failure, disk pressure, memory pressure, or network issues.

**Diagnosis**:

```bash
oc describe node <node-name>
oc get events -n openshift-machine-config-operator --sort-by=.lastTimestamp
```

Check the **Conditions** section for `Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`, and `NetworkUnavailable`.

**Resolution**:
- Check system resources on the node directly
- Review kubelet logs: `journalctl -u kubelet --no-pager -n 100`
- Verify OVS and network connectivity

### Cluster Operator Degraded

**Diagnosis**:

```bash
oc get clusteroperators
oc describe clusteroperator <operator-name>
```

**Resolution**:
- Review the operator's related objects for errors
- Check operator pod logs in the operator's namespace
- For transient issues, the operator may self-heal within minutes

### Authentication Failures

**Diagnosis**:

```bash
oc get identityprovider
oc get user
oc get group
```

**Resolution**:
- Verify identity provider configuration
- Check user existence and group membership
- Review OAuth server logs in `openshift-authentication` namespace

### Events Not Showing

**Cause**: Events have a limited retention period (default 1 hour).

**Resolution**:

```bash
# Use extended retention if configured
oc get events -A --field-selector type=Warning

# Alternatively, check persistent alerts
# Navigate to Console → Administration → Cluster Monitoring → Alerts
```

---

## Production Best Practices

### Cluster Access Management

- Use dedicated service accounts for automation rather than personal user tokens
- Rotate `kubeadmin` credentials after initial setup
- Restrict `cluster-admin` role to essential personnel only
- Use project-level roles for day-to-day operations

### Resource Querying Discipline

- Use `-o name` for scripting-friendly output (returns `kind/name` format)
- Prefer `--field-selector` over piping to `grep` for filtering
- Use `--no-headers` when parsing output in scripts
- Cache large `oc get -o yaml` exports; do not run them repeatedly in loops

### Image Management

- Pin production workloads to image digests, not mutable tags
- Use the cluster-internal registry as a mirror for external images
- Enable image pull through for trusted registries to reduce registry load
- Monitor image registry storage consumption

### Project Hygiene

- Apply consistent naming conventions across projects
- Label projects by team, environment, and application tier
- Set resource quotas on all projects to prevent resource starvation
- Review and delete unused projects periodically

### Monitoring and Alerting

- Configure alert notification channels (email, Slack, PagerDuty)
- Set up dashboards for critical metrics: node CPU/memory, pod restart counts, API server latency
- Monitor etcd disk latency — high latency degrades entire cluster performance
- Track operator health as a leading indicator of cluster stability

### Documentation Navigation

Red Hat OpenShift documentation is available at:
- **Product Documentation**: `https://docs.openshift.com/container-platform/4.18/`
- **CLI Reference**: Search for "oc command reference" in the documentation
- **Release Notes**: Review before every upgrade for breaking changes
- **Knowledgebase**: `https://access.redhat.com/solutions/` for troubleshooting articles

In the exam environment, use the **Help** menu in the web console to access context-sensitive documentation.

---

## Certification Exam Notes

### Exam-Relevant Commands You Must Know

```bash
oc login
oc whoami
oc get nodes
oc get pods -A
oc get events -A
oc get clusterversion
oc get clusteroperators
oc new-project <name>
oc delete project <name>
oc project
oc describe <resource> <name>
oc logs <pod-name>
oc get <resource> -o yaml
oc get <resource> -o jsonpath='{...}'
oc get <resource> -l <label-selector>
oc get <resource> --field-selector=<field>=<value>
```

### JSONPath Tips for the Exam

- `'{.items[*].metadata.name}'` — list all names
- `'{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'` — formatted table
- `'{.spec.containers[0].image}'` — first container image
- Practice JSONPath syntax before the exam — it is frequently tested

### Web Console Navigation

The exam will require you to navigate the console efficiently. Key areas:

- **Administrator View → Compute Resources → Nodes**: Node health and configuration
- **Administrator View → Workloads → Pods**: Pod inspection and logs
- **Administrator View → Home → Cluster**: Cluster version and operator status
- **Administrator View → Administration → Cluster Settings**: Global configuration
- **Developer View → Topology**: Visual application topology
- **Developer View → Builds / Deployments**: Application lifecycle

### Common Exam Scenarios

1. **"Find the pod using image X"** → `oc get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'` then inspect each pod's image
2. **"Identify the digest for a tagged image"** → `oc get istag <name>:<tag> -o jsonpath='{.image.dockerImageReference}'`
3. **"Find all Warning events"** → `oc get events -A --field-selector type=Warning`
4. **"Export a deployment"** → `oc get deployment <name> -o yaml > file.yaml`
5. **"Assess cluster health"** → Run `oc get clusteroperators`, `oc get nodes`, and `oc get clusterversion`

---

## Common Mistakes

### Mistake: Confusing `oc delete namespace` with `oc delete project`

`oc delete project` is the OpenShift-recommended command. It performs additional cleanup such as removing finalizers and triggering project-specific hooks. `oc delete namespace` works but bypasses OpenShift project lifecycle management.

### Mistake: Using `grep` Instead of Native Filtering

Piping `oc get` output to `grep` is fragile and breaks with output format changes. Use `-l`, `--field-selector`, and `-o jsonpath` instead.

### Mistake: Forgetting `--all-namespaces` or `-A`

Many exam questions ask about resources across the entire cluster. Forgetting `-A` limits your search to the current project and causes missed resources.

### Mistake: Not Checking Events for Root Cause

When a pod fails, jumping to `oc logs` without first running `oc describe pod` misses scheduling, volume, and SCC-related errors that never reach the container.

### Mistake: Assuming Tags Are Immutable

Tags can be retagged at any time. A deployment referencing `nginx:latest` may silently receive a different image. Always verify the actual digest in production.

### Mistake: Ignoring Cluster Operator Status

A degraded cluster operator can cause cascading failures across unrelated components. Always check `oc get clusteroperators` when diagnosing cluster-wide issues.

---

## Chapter Summary

This chapter covered the foundational skills for managing an OpenShift Container Platform cluster. You learned to navigate both the web console and the `oc` command-line interface. You understood the architecture of OpenShift control plane components, the role of operators in cluster management, and how OVNKubernetes provides networking. You practiced creating and deleting projects, locating and examining container images, and distinguishing between mutable tags and immutable digests. You mastered resource querying using label selectors, field selectors, and JSONPath formatting. You learned to export and import resource manifests, monitor events and alerts, view container logs, and assess overall cluster health. Troubleshooting scenarios addressed common pod failures including Pending, CrashLoopBackOff, and ImagePullBackOff states. Production best practices emphasized access control, image pinning, project hygiene, and proactive monitoring.

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Login to cluster | `oc login -u <user> -p <password> https://api.cluster:6443` |
| Check current user | `oc whoami` |
| List current project | `oc project` |
| Create project | `oc new-project <name>` |
| Delete project | `oc delete project <name>` |
| Switch project | `oc project <name>` |
| List all pods | `oc get pods -A` |
| List nodes | `oc get nodes` |
| Describe resource | `oc describe <kind> <name>` |
| View pod logs | `oc logs <pod-name>` |
| Follow pod logs | `oc logs -f <pod-name>` |
| View previous logs | `oc logs <pod-name> --previous` |
| Export resource | `oc get <kind> <name> -o yaml > file.yaml` |
| Apply manifest | `oc apply -f file.yaml` |
| Filter by label | `oc get pods -l app=nginx` |
| Filter by field | `oc get pods --field-selector=status.phase=Running` |
| JSONPath output | `oc get pods -o jsonpath='{.items[*].metadata.name}'` |
| Cluster version | `oc get clusterversion` |
| Cluster operators | `oc get clusteroperators` |
| All events | `oc get events -A` |
| Warning events | `oc get events -A --field-selector type=Warning` |
| Image stream tags | `oc get istag -n <namespace>` |
| Custom columns | `oc get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase` |

### JSONPath Cheatsheet

| Pattern | Description |
|---------|-------------|
| `'{.metadata.name}'` | Resource name |
| `'{.items[*].metadata.name}'` | All resource names |
| `'{.spec.containers[0].image}'` | First container image |
| `'{.status.phase}'` | Pod phase |
| `'{range .items[*]}{.metadata.name}{"\n"}{end}'` | Loop over items |
| `'{.spec.template.spec.containers[*].image}'` | All container images in template |

### Web Console URLs

| Resource | Console Path |
|----------|-------------|
| API server | `https://api.cluster.example.com:6443` |
| Web console | `https://console-openshift-console.apps.cluster.example.com` |
| Internal registry | `default-route-openshift-image-registry.apps.cluster.example.com` |

---

## Review Questions

1. What is the difference between an OpenShift project and a Kubernetes namespace?

2. Which command displays the current cluster version and upgrade status?

3. What does the `CrashLoopBackOff` pod status indicate, and what is the first command you should run to diagnose it?

4. How do you filter pods by label `app=nginx` that are currently in the `Running` state?

5. What is the difference between a container image tag and a digest? Why does this distinction matter in production?

6. Which OpenShift component manages cluster-wide upgrades and component versioning?

7. How do you view the logs from a container that previously crashed in a pod?

8. What command exports a deployment configuration to a YAML file?

9. What does `oc get clusteroperators` reveal, and what values indicate a healthy operator?

10. How do you find all Warning-level events across all namespaces, sorted by timestamp?

11. What is the purpose of the Machine Config Operator (MCO)?

12. In the OpenShift web console, which view should you use to inspect cluster-wide node health?

13. What JSONPath expression retrieves the image name of the first container in a pod named `web-pod`?

14. What is OVNKubernetes and what role does it play in the cluster?

15. Why is `oc delete project` preferred over `oc delete namespace` in OpenShift?

---

## Answers

1. An OpenShift project is a Kubernetes namespace with additional metadata, default resource quotas, template associations, and lifecycle management hooks. Projects are managed through OpenShift's project API, which provides extra administrative controls beyond the standard namespace resource.

2. `oc get clusterversion` displays the current version, desired version, state (Installing, Updating, or Done), and any available updates.

3. `CrashLoopBackOff` means the container started, exited with an error, and the kubelet is restarting it with exponential backoff. The first command to diagnose is `oc describe pod <pod-name>` to check events and the Last State exit code, followed by `oc logs <pod-name> --previous` to view the crashed container's output.

4. `oc get pods -l app=nginx --field-selector=status.phase=Running`

5. A tag is a mutable label that can point to different image contents over time. A digest is an immutable SHA-256 hash of the exact image content. In production, using digests ensures deployments cannot silently change when a tag is updated in the registry.

6. The Cluster Version Operator (CVO) manages cluster-wide upgrades, component versioning, and configuration drift detection.

7. `oc logs <pod-name> --previous` retrieves logs from the last terminated container instance.

8. `oc get deployment <name> -o yaml > deployment.yaml`

9. It lists all cluster operators and their conditions. A healthy operator shows `AVAILABLE=True`, `PROGRESSING=False`, and `DEGRADED=False`.

10. `oc get events -A --field-selector type=Warning --sort-by=.lastTimestamp`

11. The Machine Config Operator manages node-level configuration including kernel parameters, systemd units, container runtime settings, and SSH keys. It applies configuration changes through MachineConfigObjects and triggers node reboots when required.

12. The **Administrator View** under **Compute Resources → Nodes**.

13. `oc get pod web-pod -o jsonpath='{.spec.containers[0].image}'`

14. OVNKubernetes is the default CNI (Container Network Interface) plugin in OpenShift 4.18. It provides software-defined networking using Open vSwitch, enabling pod-to-pod communication across nodes, network policy enforcement, and multi-network support.

15. `oc delete project` triggers OpenShift-specific cleanup including finalizer removal, project lifecycle hooks, and quota cleanup. `oc delete namespace` bypasses these OpenShift mechanisms and may leave residual configuration.
