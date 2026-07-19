# Chapter 1: Manage OpenShift Container Platform

> **Exam baseline:** EX280 on OpenShift Container Platform 4.18.  
> **Reviewed:** 2026-07-19.  
> The installation labs in this chapter build a home practice environment. Installing a cluster is not a direct EX280 objective, but a working lab is essential for practice.

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

## Foundations: Linux, Containers, Kubernetes, and OpenShift

### Start with the full picture

OpenShift is not a replacement for Linux, containers, or Kubernetes. It combines them into an administered application platform:

```text
physical computer or virtual machine
                |
              Linux
                |
        CRI-O container runtime
                |
             containers
                |
        pods and controllers
                |
          Kubernetes APIs
                |
OpenShift APIs, Operators, console, Routes, ImageStreams, SCCs, monitoring
```

A **Linux process** is a running program. A **container** is still one or more Linux processes, but kernel features isolate its file system, process IDs, network, users, and resource consumption. Containers on one node share that node's kernel.

A **virtual machine (VM)** emulates a complete computer and normally runs its own kernel. A VM is heavier, but it can run a different operating system kernel. OpenShift nodes are often VMs; containers then run inside those node VMs.

### Image, container, registry, and repository

- An **OCI image** is an immutable package of file-system layers plus metadata such as the default command, environment, and architecture.
- A **container** is a runtime instance created from an image. Its writable layer is normally disposable.
- A **registry** is a server that stores and distributes images, such as `registry.redhat.io` or Quay.
- A **repository** groups related images in a registry.
- A **tag** such as `1.4` is a movable name. A **digest** such as `sha256:...` identifies exact content.

This pull specification has all four useful parts:

```text
registry.example.com/team/payment-api:1.4
| registry          | repository       | tag
```

The immutable form uses a digest:

```text
registry.example.com/team/payment-api@sha256:<64-hex-characters>
```

### Podman, CRI-O, and `oc`

**Podman** is a daemonless tool for building, pulling, inspecting, and running OCI containers on one host. It is excellent for learning container behavior and testing an image before deploying it.

**CRI-O** is the Kubernetes container runtime on OpenShift nodes. The kubelet asks CRI-O to create containers for pods. Administrators do not use Podman to manage Kubernetes pods on a node.

**`oc`** talks to the OpenShift/Kubernetes API. The control plane records desired state and controllers make it real across the cluster.

| Goal | Local container host | OpenShift cluster |
|---|---|---|
| Pull image | `podman pull IMAGE` | Kubelet/CRI-O pulls when a pod is scheduled |
| Inspect image | `podman image inspect IMAGE` | `oc get pod ... -o yaml`; `oc describe imagestreamtag ...` |
| Run workload | `podman run ...` | Create a Pod, Deployment, Job, or other controller |
| See running units | `podman ps` | `oc get pods` |
| Read logs | `podman logs CONTAINER` | `oc logs POD` |
| Stop/delete | `podman rm -f CONTAINER` | Change/delete the controller; do not manage a controller's pod manually |

Try this on a container host:

```bash
podman pull registry.access.redhat.com/ubi9/ubi-minimal:latest
podman image inspect registry.access.redhat.com/ubi9/ubi-minimal:latest
podman run --rm registry.access.redhat.com/ubi9/ubi-minimal:latest cat /etc/os-release
```

### Kubernetes objects in plain English

- A **Pod** is the smallest Kubernetes scheduling unit. Its containers share one network identity and can share volumes. Pods are replaceable.
- A **Deployment** states how many replicas of a stateless pod template should run and manages rolling updates through ReplicaSets.
- A **ReplicaSet** keeps a requested number of matching pods running. A Deployment normally owns it.
- A **StatefulSet** gives replicas stable identities and supports stable storage patterns.
- A **DaemonSet** aims to run a pod on each selected node.
- A **Job** runs work to successful completion. A **CronJob** creates Jobs on a schedule.
- A **Service** gives a changing set of selected pods a stable virtual IP and DNS name.
- An OpenShift **Route** exposes a Service through an Ingress Controller for HTTP, HTTPS, or TLS with SNI.
- A **ConfigMap** stores non-secret configuration. A **Secret** stores sensitive bytes, but base64 encoding is not encryption.
- A **PersistentVolumeClaim (PVC)** requests persistent storage from the cluster.
- A **Namespace/Project** separates names and provides an authorization, quota, and policy boundary.

The common request path is:

```text
browser -> Route -> Ingress Controller -> Service -> ready Pod -> container
```

The common reconciliation path is:

```text
oc apply -> API server validates and stores desired state in etcd
         -> controller creates/updates child objects
         -> scheduler assigns unscheduled pod to a node
         -> kubelet asks CRI-O to start containers
         -> status/events flow back through the API
```

### Declarative administration

Kubernetes is designed around desired state. Prefer a version-controlled manifest and `oc apply` for durable configuration. Imperative commands are useful for discovery, fast exam work, and generating a starting manifest:

```bash
oc create deployment hello \
  --image=registry.access.redhat.com/ubi9/httpd-24:latest \
  --dry-run=client -o yaml > hello-deployment.yaml

oc apply -f hello-deployment.yaml
```

Always verify the resulting object and workload. A successful `apply` only proves that the API accepted the request.

---

## Home Lab Option A: OpenShift Local (CRC)

OpenShift Local is the simplest official way to run a minimal, preconfigured single-node OpenShift cluster on a laptop or desktop. The command is named `crc`. On Linux it uses KVM/libvirt, so this option already teaches OpenShift on local virtualization without requiring you to assemble the cluster yourself.

### When to choose it

Choose OpenShift Local when your goal is EX280 practice: projects, deployments, Routes, RBAC, quotas, NetworkPolicy, SCCs, and Operators. Choose the KVM/QEMU single-node lab in the next section when your goal is also to observe an installer workflow, DNS, RHCOS, and bootstrap.

CRC does not model a highly available production cluster. It does not support nested virtualization, so do not expect reliable results when the desktop itself is a VM.

### Current minimum and practical allocation

The CRC project currently lists 4 physical CPU cores, 10.5 GB free memory, and 35 GB storage for the OpenShift preset. That is a minimum. For serious labs, a host with at least 8 cores, 24–32 GB RAM, and 100 GB free SSD space is much more comfortable. Leave memory for the host operating system.

### Linux preparation

On a supported RHEL/Fedora-family host:

```bash
sudo dnf install -y libvirt NetworkManager
sudo systemctl enable --now libvirtd
```

Confirm hardware virtualization:

```bash
test -e /dev/kvm && echo "KVM device present"
lsmod | grep -E '^kvm'
```

Download the current OpenShift Local archive from the OpenShift Cluster Manager local installation page, extract it, and put `crc` in a directory on `PATH`. Also download your pull secret. Do not publish or commit the pull secret.

### Configure and start

```bash
crc version
crc setup
crc config set cpus 6
crc config set memory 16384
crc config set disk-size 80
crc config view
crc start -p "$HOME/Downloads/pull-secret.txt"
```

Memory is specified in MiB. Adjust to your host. Configuration changes take effect when the instance starts; stop and start it if you change settings later.

### Log in and verify

At the end of `crc start`, CRC displays developer and administrator credentials. You can print access instructions again:

```bash
crc console --credentials
eval "$(crc oc-env)"
oc login -u kubeadmin https://api.crc.testing:6443
oc whoami
oc get nodes
oc get clusteroperators
oc get clusterversion
```

Use `crc console` to open the web console. A healthy learning cluster should have a Ready node and Cluster Operators that are Available without persistent Degraded conditions.

### Day-to-day lifecycle

```bash
crc status
crc stop
crc start
```

`crc delete` destroys the instance and its cluster data. Use it only when you intend to rebuild:

```bash
crc delete
crc cleanup
```

---

## Home Lab Option B: Single-Node OpenShift in a KVM/QEMU VM

This lab uses the Assisted Installer discovery ISO and a libvirt VM. It gives a home user a real single-node OpenShift (SNO) installation flow. It is more demanding than CRC.

### Support boundary

Red Hat documents SNO for bare metal and certified third-party hypervisors, and the Assisted Installer can install on other platforms without infrastructure integration. A generic desktop KVM/QEMU lab can be technically useful but is not automatically a supported production platform. Do not present a home VM as production architecture.

### SNO limitations and resources

SNO combines control-plane and compute work on one node. It has no control-plane high availability: if the VM is down, the cluster is down.

The documented minimum is 8 vCPUs, 16 GB RAM, and 120 GB storage. For a useful home lab, prefer roughly 12 vCPUs, 24–32 GB RAM, and 150–200 GB fast storage. Operators add requirements.

### Required names and networking

Choose values and keep them consistent. This example uses:

```text
cluster name: sno
base domain: ocp.test
node address: 192.168.122.50
API:          api.sno.ocp.test        -> 192.168.122.50
internal API: api-int.sno.ocp.test    -> 192.168.122.50
applications: *.apps.sno.ocp.test     -> 192.168.122.50
```

All three names must resolve where needed. An `/etc/hosts` entry cannot express a wildcard and changes only one machine, so use a real local DNS service. Configure a DHCP reservation for the VM's MAC address or a stable static address. OpenShift and etcd require the address to remain stable.

### Configure home-lab DNS before installation

Use a DNS resolver that both the desktop and the discovery VM can reach. A home router with custom DNS, Pi-hole, or a small dedicated `dnsmasq` host can provide these records. For a `dnsmasq`-compatible resolver, the essential configuration is:

```ini
host-record=api.sno.ocp.test,192.168.122.50
host-record=api-int.sno.ocp.test,192.168.122.50
address=/apps.sno.ocp.test/192.168.122.50
```

The `address=/apps.../` line supplies the wildcard behavior: every name below `apps.sno.ocp.test` resolves to the ingress address. Test the DNS server directly before installation:

```bash
dig @<dns-server-ip> api.sno.ocp.test +short
dig @<dns-server-ip> api-int.sno.ocp.test +short
dig @<dns-server-ip> console-openshift-console.apps.sno.ocp.test +short
```

All three commands should return `192.168.122.50`. Configure the desktop's connection and the VM's DHCP-provided resolver to use that DNS server. Do not start a second DNS daemon on an address/port already owned by libvirt's DNS service.

For a temporary workstation-only workaround, `/etc/hosts` can contain exact entries for `api`, `api-int`, the console, OAuth, and each Route you test. It cannot provide `*.apps` wildcard DNS and does not configure the VM, so it is not a complete installation DNS design.

### Prepare the Linux host

Package names vary slightly by distribution. On RHEL/Fedora-family systems:

```bash
sudo dnf install -y qemu-kvm libvirt virt-install virt-viewer dnsmasq
sudo systemctl enable --now libvirtd
sudo virsh net-start default 2>/dev/null || true
sudo virsh net-autostart default
sudo virsh net-info default
```

Confirm that the host has enough free resources and that `/dev/kvm` exists:

```bash
test -e /dev/kvm
free -h
df -h
```

Reserve the example address for the example MAC on libvirt's `default` network **before** booting the VM:

```bash
sudo virsh net-update default add-last ip-dhcp-host \
  "<host mac='52:54:00:28:00:01' name='ocp-sno' ip='192.168.122.50'/>" \
  --live --config
sudo virsh net-dumpxml default | grep -A2 52:54:00:28:00:01
```

If that host entry already exists, inspect `sudo virsh net-dumpxml default` instead of adding a duplicate. The reservation, VM definition, DNS records, and installer form must all use the same MAC address and IP.

### Create the Assisted Installer ISO

1. Sign in to Red Hat OpenShift Cluster Manager and choose **Create cluster**.
2. Choose the Assisted Installer and select **Install single node OpenShift**.
3. Enter the cluster name and base domain exactly as planned.
4. Add your SSH public key and networking information.
5. Generate and download the discovery ISO.
6. Use the ISO promptly. Regenerate expired installation media rather than debugging old credentials embedded in it.

The exact web form changes over time, so trust the current installer validation messages for mandatory fields.

### Create the VM

Pick a MAC address first so that local DHCP can reserve the chosen IP. Then create the VM:

```bash
sudo virt-install \
  --name ocp-sno \
  --memory 24576 \
  --vcpus 12 \
  --cpu host-passthrough \
  --disk path=/var/lib/libvirt/images/ocp-sno.qcow2,size=160,bus=virtio,format=qcow2 \
  --network network=default,model=virtio,mac=52:54:00:28:00:01 \
  --cdrom "$HOME/Downloads/discovery_image_sno.iso" \
  --os-variant rhel9.0 \
  --graphics spice \
  --noautoconsole
```

If the host has no graphical session, replace the graphics choice with a console or VNC setup you can access. Do not blindly copy the example disk path, IP, or MAC when they conflict with your machine.

Open the VM console, boot the ISO, and return to the Assisted Installer. Wait for the host to appear. Resolve every host validation error before starting installation. Monitor until the installer reports completion.

### Boot from the installed disk

After installation, eject the ISO so the VM boots from its disk. First identify the optical device:

```bash
sudo virsh domblklist ocp-sno
```

Then use the reported CD-ROM target in place of `<cdrom-target>`:

```bash
sudo virsh change-media ocp-sno <cdrom-target> --eject --config
sudo virsh reboot ocp-sno
```

### Obtain kubeconfig and verify

Download the cluster kubeconfig from the Assisted Installer, protect it, and use it:

```bash
install -m 600 "$HOME/Downloads/kubeconfig" "$HOME/.kube/config-sno"
export KUBECONFIG="$HOME/.kube/config-sno"
oc whoami
oc get nodes -o wide
oc get clusteroperators
oc get clusterversion
oc get ingresscontroller/default -n openshift-ingress-operator
```

Check DNS from the workstation:

```bash
getent hosts api.sno.ocp.test
getent hosts console-openshift-console.apps.sno.ocp.test
```

If the API works but application Routes do not, wildcard application DNS is the first item to check.

### What this lab teaches

- the difference between OpenShift Local and a normal installer-created cluster;
- RHCOS as the node operating system;
- why stable IP addressing and DNS are part of the cluster design;
- why SNO is suitable for learning but not highly available; and
- how the installer, Cluster Version Operator, and Cluster Operators establish the final platform.

---

## OpenShift Concepts

OpenShift Container Platform (OCP) is Red Hat's enterprise Kubernetes distribution. It extends upstream Kubernetes with supported platform Operators, developer workflows, security controls, Routes, ImageStreams, and a unified management console. Capabilities such as OpenShift Pipelines are installed separately when required; do not assume every optional Operator is present.

### OpenShift vs. Kubernetes

While Kubernetes provides the core container orchestration engine, OpenShift adds:

- **Integrated registry capability**: The Image Registry Operator can provide a cluster registry, but its storage and management state depend on installation/platform configuration
- **Web console**: A unified graphical interface for administrators, developers, and cluster administrators
- **Security Context Constraints (SCCs)**: OpenShift admission policy for pod security; RBAC grants a user or service account permission to use an SCC, so SCCs do not replace RBAC
- **Routes**: OpenShift's Layer 7 load balancer, built on HAProxy, for exposing applications externally
- **Image Streams**: An OpenShift-specific resource that tracks container images by tag and digest
- **Templates**: Server-side parameterized application blueprints
- **Operators**: Packaged, deployable, and manageable operators for automated cluster operations

### The OpenShift Cluster Model

An OpenShift cluster consists of three node categories:

1. **Control Plane Nodes**: Run the Kubernetes API server, etcd, scheduler, and controller manager. The installer/platform creates machines; the Machine Config Operator manages supported operating-system configuration after they join.
2. **Compute (Worker) Nodes**: Run user workloads — pods, containers, and applications.
3. **Infrastructure Nodes**: An optional operational pattern: worker nodes are labeled/tainted so selected infrastructure workloads such as ingress, monitoring, or the registry run there. They are not a separate Kubernetes node type.

OpenShift 4 uses a self-hosted etcd model where the etcd cluster runs as static pods on control plane nodes, managed by the etcd operator.

### The Operator Framework

OpenShift is built on the Operator Framework. Nearly every cluster capability is managed by an operator:

- **Cluster Version Operator (CVO)**: Manages cluster-wide upgrades and component versioning
- **Machine Config Operator (MCO)**: Manages node configuration, kernel parameters, and systemd units
- **Cluster Network Operator**: Manages the CNI plugin (OVNKubernetes by default)
- **Ingress Operator**: Manages the HAProxy-based ingress controller
- **Cluster Monitoring Operator**: Manages the supported monitoring stack, including Prometheus, Alertmanager, Thanos Querier, and console-integrated dashboards

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

Each node runs Open vSwitch components that implement logical networking. The default overlay uses **Geneve**, not VXLAN. Do not assume that every current pod subnet maps one-to-one to a node; inspect the installed network configuration.

### Image Registry

When managed and configured, the cluster-internal registry service is available as `image-registry.openshift-image-registry.svc:5000`. It stores images pushed by builds and users. Backing storage is platform-dependent and can be object storage, a PVC, or another supported configuration; it is not always a PVC.

### Authentication Flow

When a user authenticates to OpenShift, the following components participate:

1. A user reaches the integrated OpenShift OAuth server, often through the console or `oc login`.
2. The selected configured **identity provider** validates the user.
3. OpenShift maps the external identity to an OpenShift `User` according to the provider's mapping method.
4. The OAuth server issues an OAuth access token. `oc` sends that token directly as a bearer token to the API; it is not exchanged for a service-account token.

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
oc login https://api.cluster.example.com:6443 -u administrator
```

Let the command prompt for the password so it is not written into shell history. Use the cluster's trusted CA. `--insecure-skip-tls-verify=true` is a temporary lab diagnostic, not a normal configuration or a certificate fix.

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

# Generate a clean starting manifest when a generator supports dry-run
oc create deployment example --image=<image> --dry-run=client -o yaml > deployment.yaml
```

The removed `--export` flag must not be used on OpenShift 4.18. Output from `oc get -o yaml` contains server-generated fields such as `status`, `uid`, `resourceVersion`, and `managedFields`; remove those before treating it as a portable desired-state manifest. Also, `oc get all` does not literally return every resource kind and is not a backup method.

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

The ClusterVersion object shows current and desired release state. Use `oc get clusterversion -o yaml` or `oc adm upgrade` when you specifically need the available/recommended update information.

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
- `resources.requests`: Values used by the scheduler for placement and by the runtime for resource-sharing behavior; they are not a promise that an application will continuously consume that amount
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
oc get oauth cluster -o yaml
oc get users
oc get identities
oc get groups
```

**Resolution**:
- Verify the identity provider entries under `oauth.config.openshift.io/cluster.spec.identityProviders`
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
- Use the integrated registry and ImageStreams when they solve a defined build or image-promotion requirement; do not treat them as an automatic mirror of every external image
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
- **Product Documentation**: `https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/`
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

1. **"Find the pod using image X"** → print names and images together: `oc get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'`, then locate the requested image
2. **"Identify the digest for a tagged image"** → `oc get istag <name>:<tag> -o jsonpath='{.image.dockerImageReference}'`
3. **"Find all Warning events"** → `oc get events -A --field-selector type=Warning`
4. **"Export a deployment"** → `oc get deployment <name> -o yaml > file.yaml`
5. **"Assess cluster health"** → Run `oc get clusteroperators`, `oc get nodes`, and `oc get clusterversion`

---

## Common Mistakes

### Mistake: Confusing `oc delete namespace` with `oc delete project`

Use `oc delete project <name>` when the task speaks in OpenShift project terms. A Project represents the same isolation boundary as its Kubernetes Namespace; both deletion paths enter namespace termination and honor finalizers. `oc delete namespace` does not bypass finalizers. If deletion is stuck, inspect the Project/Namespace and remaining namespaced resources instead of switching commands repeatedly.

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
| Login to cluster | `oc login -u <user> https://api.cluster:6443` (enter password at prompt) |
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
| Optional exposed registry route | Query with `oc get route/default-route -n openshift-image-registry`; it does not exist unless registry routing is configured |

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

15. What is the practical relationship between `oc delete project` and `oc delete namespace`?

---

## Answers

1. An OpenShift Project represents a Kubernetes Namespace and adds OpenShift project-request behavior and user-facing metadata. Quotas, limits, and policies are separate objects; a custom project-request template can create them, but they are not guaranteed in every project.

2. `oc get clusterversion` shows current/desired release and progress. Inspect `oc get clusterversion -o yaml` or run `oc adm upgrade` for available update details.

3. `CrashLoopBackOff` means the container started, exited with an error, and the kubelet is restarting it with exponential backoff. The first command to diagnose is `oc describe pod <pod-name>` to check events and the Last State exit code, followed by `oc logs <pod-name> --previous` to view the crashed container's output.

4. `oc get pods -l app=nginx --field-selector=status.phase=Running`

5. A tag is a mutable label that can point to different image contents over time. A digest is an immutable SHA-256 hash of the exact image content. In production, using digests ensures deployments cannot silently change when a tag is updated in the registry.

6. The Cluster Version Operator (CVO) applies and monitors the OpenShift release payload and coordinates cluster updates. Individual Operators reconcile the resources they own.

7. `oc logs <pod-name> --previous` retrieves logs from the last terminated container instance.

8. `oc get deployment <name> -o yaml > deployment.yaml`

9. It lists all cluster operators and their conditions. A healthy operator shows `AVAILABLE=True`, `PROGRESSING=False`, and `DEGRADED=False`.

10. `oc get events -A --field-selector type=Warning --sort-by=.lastTimestamp`

11. The Machine Config Operator manages node-level configuration including kernel parameters, systemd units, container runtime settings, and SSH keys. It applies configuration changes through MachineConfigObjects and triggers node reboots when required.

12. The **Administrator View** under **Compute Resources → Nodes**.

13. `oc get pod web-pod -o jsonpath='{.spec.containers[0].image}'`

14. OVNKubernetes is the default CNI (Container Network Interface) plugin in OpenShift 4.18. It provides software-defined networking using Open vSwitch, enabling pod-to-pod communication across nodes, network policy enforcement, and multi-network support.

15. Use `oc delete project` to match OpenShift terminology. Both Project and Namespace deletion honor namespace finalizers; neither command is a way to bypass cleanup.

---

## Verified References

- EX280 objectives and 4.18 exam baseline: <https://www.redhat.com/en/services/training/red-hat-certified-openshift-administrator-exam>
- OpenShift 4.18 installation overview and OpenShift Local overview: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installation_overview/index>
- OpenShift 4.18 single-node requirements and installation: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_on_a_single_node/index>
- CRC/OpenShift Local installation and minimum requirements: <https://crc.dev/docs/installing/>
- Podman documentation: <https://docs.podman.io/en/latest/>
- OpenShift 4.18 architecture: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/architecture/index>
