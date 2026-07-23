---
kind: chapter
id: C00
title: "OpenShift and Kubernetes Fundamentals: Zero-to-Ready"
status: draft
---

# C00 — OpenShift and Kubernetes Fundamentals: Zero-to-Ready

## Start Here

This is the required first chapter of the handbook. It assumes only that you
can open a terminal and recognize a few basic Podman commands. You do **not**
need previous Kubernetes, OpenShift, networking, storage, security, or YAML
knowledge.

Read this chapter before Chapter 01. Do not try to memorize every command on
the first pass. Your first goal is to understand how the parts relate to each
other. Return to the command and YAML sections while working through later
chapters.

### What you will be able to do

After completing this chapter, you should be able to:

- explain the difference between an image, container, pod, node, cluster,
  namespace, and OpenShift project;
- explain why Kubernetes exists and what OpenShift adds to Kubernetes;
- follow a request from the `oc` command to the API server and finally to a
  running container;
- read and write ordinary YAML without guessing at indentation;
- recognize the required fields of a Kubernetes or OpenShift manifest;
- choose the correct workload controller for a basic application;
- explain how Deployments, ReplicaSets, Pods, Services, Routes, ConfigMaps,
  Secrets, and persistent storage fit together;
- use labels and selectors without accidentally disconnecting resources;
- understand requests, limits, scheduling, health probes, service accounts,
  RBAC, and Security Context Constraints at a foundation level;
- use the safest everyday `oc` workflow to create, inspect, change, verify, and
  troubleshoot resources;
- read status, conditions, events, and logs as evidence instead of treating an
  error message as random text; and
- begin Chapter 01 without an unstated Kubernetes or YAML prerequisite.

### Version boundary

This handbook targets OpenShift Container Platform 4.18 for EX280 study.
Kubernetes and OpenShift continue to evolve, but the durable mental models in
this chapter remain the same. When a field or API is uncertain, ask the cluster
that you are using:

```bash
oc version
oc api-resources
oc api-versions
oc explain deployment
oc explain deployment.spec
```

The cluster API is the final authority for resources and fields supported by
that cluster. Never invent a field because it looks reasonable.

## 1. From Podman to a Cluster

You already know a useful starting point: Podman can pull an image and start a
container on one Linux machine. Kubernetes and OpenShift build on the same
container idea, but solve a larger problem.

### Image, container, and registry

An **image** is a packaged, read-only application filesystem plus metadata that
describes how the application can start. It normally contains the application,
runtime, libraries, and default command.

A **container** is a running instance of an image. The image is the template;
the container is a process created from that template. You can create many
containers from one image.

A **registry** stores and distributes images. An image reference can identify:

```text
registry / repository : tag
```

For example:

```text
registry.example.com/training/web:1.4
```

- `registry.example.com` identifies the registry.
- `training/web` identifies the repository.
- `1.4` is a movable tag selected by a human or automation.

An image **digest**, such as `sha256:...`, identifies exact immutable image
content. A tag can later point to a different digest. A digest does not move.

### What Podman manages

When you run:

```bash
podman run --name web -d IMAGE
```

Podman asks one host to create one container. If the host fails, another host
does not automatically replace that container. If you want three copies, you
must create and manage them yourself or add other tooling.

### What Kubernetes manages

Kubernetes manages desired application state across a group of machines. You
declare something such as:

> Keep three instances of this application running.

Kubernetes continuously compares that desired state with reality. If one
instance disappears, a controller creates a replacement. If a node fails, the
control plane can arrange replacement pods on suitable surviving nodes.

This continuous correction is called **reconciliation**.

### Podman-to-OpenShift translation

| Local container idea | Cluster idea |
| --- | --- |
| `podman pull IMAGE` | Nodes pull the image when a pod is scheduled |
| `podman run IMAGE` | A Pod describes one or more cooperating containers |
| A shell script keeps a process alive | A controller maintains desired pods |
| Host port mapping | Service gives pods a stable internal endpoint |
| Reverse proxy configuration | Route exposes an HTTP/TLS Service externally |
| Local environment variables | Pod spec, ConfigMap, and Secret provide configuration |
| Bind mount or named volume | Volume and usually a PVC provide storage |
| One Linux host | A cluster contains control-plane and worker nodes |
| Root or local user permissions | Service account, RBAC, SCC, and security context apply |

> **Key change:** With Podman, you mainly manage containers. With Kubernetes
> and OpenShift, you mainly manage API objects that describe the state you
> want. Controllers then manage pods and containers for you.

## 2. Kubernetes in One Mental Picture

**Kubernetes** is an open-source container orchestration system. Orchestration
means coordinating containerized applications across machines: placement,
replacement, scaling, networking, configuration, storage attachment, and
controlled updates.

A Kubernetes installation is called a **cluster**. A cluster contains a
control plane and one or more nodes that can run workloads.

```text
You or automation
       |
       | oc / web console / API client
       v
Kubernetes API server
       |
       +---- desired and observed state ----> etcd
       |
       +---- controllers compare desired state with reality
       |
       +---- scheduler chooses a node for each unscheduled Pod
       |
       v
Worker node: kubelet -> CRI-O -> containers
```

### Declarative management

Kubernetes is primarily **declarative**. You describe the end state, rather
than writing every low-level step required to reach it.

Imperative thinking:

```text
Start container A.
Start container B.
If B fails, run B again.
```

Declarative thinking:

```text
The application must have two ready replicas.
```

The second statement leaves the control loop responsible for reaching and
maintaining the result.

### Desired state and observed state

Most API objects have:

- **spec**: the desired state supplied by a user or controller; and
- **status**: the observed state reported by the platform.

Suppose a Deployment has `spec.replicas: 3`, but only two pods are ready.
Three is the desired state; two is the observed state. Controllers work to
remove the difference.

Do not normally write a `status` section into your manifest. The platform owns
status. Your source manifest should contain the desired configuration.

### Reconciliation is not a one-time script

A controller does not finish forever after creating an object. It repeatedly:

1. watches relevant API objects;
2. reads the desired state;
3. observes current state;
4. takes an action if the states differ; and
5. reports status and tries again.

This explains several behaviors that initially look strange:

- deleting a pod owned by a Deployment causes a replacement pod to appear;
- editing a generated ReplicaSet may be undone or superseded;
- changing a Deployment creates or updates lower-level objects;
- a resource can exist successfully while its application is not yet ready;
- deleting a parent controller can delete objects that it owns.

## 3. Kubernetes Cluster Components

You do not need to memorize every internal process before using OpenShift, but
you must understand the direction of control and where to look for evidence.

### Control plane

The **control plane** makes cluster-wide decisions and maintains cluster state.
Its core components include:

#### API server

The API server is the front door to the cluster. The CLI, web console,
controllers, operators, and other clients communicate through the API.

For an incoming request, the API server broadly performs:

1. authentication: who is making the request?
2. authorization: may that identity perform this action?
3. admission: is the requested object acceptable, and must it be changed?
4. schema validation: are the API fields valid?
5. persistence: store accepted state in etcd.

When `oc` returns `Forbidden`, the API server was reached but authorization
denied the action. When it reports an unknown field, the manifest did not match
the API schema. Those are different problems.

#### etcd

**etcd** is the consistent key-value data store holding Kubernetes API state.
It is not a general application database. Administrators interact with cluster
state through supported APIs, not by manually editing etcd data.

Because etcd holds critical cluster state, backup and recovery are
cluster-administration responsibilities. Backing up application files alone is
not the same as backing up cluster state.

#### Scheduler

The scheduler watches for pods that have not been assigned to nodes. It
filters and scores possible nodes using requirements such as:

- resource requests;
- node selectors and affinity;
- taints and tolerations;
- topology rules;
- volume constraints; and
- policy or plugin decisions.

The scheduler chooses a node. It does not start the container itself.

#### Controller manager

The controller manager runs many reconciliation loops. Examples include
controllers for Deployments, ReplicaSets, Jobs, namespaces, nodes, service
accounts, and endpoints.

### Worker-node components

A **node** is a machine in the cluster. A node can be a virtual machine or a
physical server.

#### Kubelet

The **kubelet** is the Kubernetes node agent. It watches the API for pods
assigned to its node and works with the container runtime to make the pod real.
It also reports pod, container, and node status.

#### Container runtime

The runtime pulls images and creates containers. OpenShift uses **CRI-O** as
its Kubernetes container runtime. Podman is valuable on your workstation, but
Podman is not the service that runs OpenShift pods on cluster nodes.

#### Cluster networking

Cluster networking gives pods addresses and connectivity. OpenShift also runs
DNS, routing, ingress, service networking, and network-policy components.

### Add-ons and platform services

Useful cluster functions such as DNS, monitoring, image registry integration,
and ingress are implemented by additional components. In OpenShift, Operators
manage many of these platform capabilities.

## 4. What OpenShift Adds

**OpenShift Container Platform** is a Kubernetes platform. Kubernetes remains
the orchestration foundation, but OpenShift supplies an integrated,
administered product around it.

Important OpenShift additions include:

- the `oc` CLI, which includes Kubernetes operations plus OpenShift commands;
- an integrated web console with Administrator and Developer perspectives;
- OAuth-based user authentication integration;
- **Projects**, which present Kubernetes namespaces with OpenShift behavior and
  metadata;
- **Routes** for exposing Services through the OpenShift ingress router;
- **ImageStreams** for tracking image tags and digests inside the cluster;
- build resources and Source-to-Image workflows;
- Security Context Constraints for pod admission and privilege control;
- Cluster Operators that manage platform capabilities;
- OperatorHub and Operator Lifecycle Manager for optional Operators;
- integrated monitoring, alerting, networking, and update workflows; and
- Red Hat Enterprise Linux CoreOS and the Machine Config Operator for node
  operating-system management.

### Kubernetes resources and OpenShift resources coexist

An OpenShift cluster supports ordinary Kubernetes resources:

```text
Pod, Deployment, StatefulSet, DaemonSet, Job, Service, ConfigMap, Secret...
```

It also supports OpenShift API resources:

```text
Project, Route, ImageStream, BuildConfig, SecurityContextConstraints...
```

The API group in `apiVersion` shows where a named API comes from. Examples:

```yaml
apiVersion: v1
kind: Service
```

```yaml
apiVersion: apps/v1
kind: Deployment
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
```

### `oc` and `kubectl`

`kubectl` is the Kubernetes CLI. `oc` includes compatible Kubernetes
capabilities and understands OpenShift-specific resources and workflows.
Within this handbook, use `oc` unless a task explicitly requires something
else.

Examples of OpenShift-oriented commands include:

```bash
oc login
oc new-project
oc project
oc status
oc expose service
oc get route
oc get clusteroperators
oc adm
```

### OpenShift Local

OpenShift Local, also known through the `crc` command, provides a compact
OpenShift environment on a workstation for learning and development. It is
useful for practicing API objects, CLI work, the console, projects, workloads,
Services, Routes, RBAC, and many operator workflows.

It is not a replacement for a highly available production cluster. A compact
local environment has fewer machines, limited resources, and different
failure scenarios. Learn concepts locally, but do not infer production
capacity or high availability from one workstation.

Typical Linux preparation after OpenShift Local is running:

```bash
crc status
crc console --credentials
eval "$(crc oc-env)"
oc version
```

The exact credentials and API address come from your own environment. Never
copy credentials printed for another cluster.

## 5. The Kubernetes API Object Model

Almost everything you manage is represented as an **API resource**. A saved
instance of a resource is an **object**.

Examples:

- `Deployment` is a resource kind.
- A Deployment named `web` in project `training` is an object.
- `deployments.apps` is an API resource name shown by discovery.

### The four fields you must recognize

A normal manifest begins with:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
```

| Field | Meaning |
| --- | --- |
| `apiVersion` | API group and version that defines the object |
| `kind` | Type of object to create |
| `metadata` | Identity and organization data, such as name, namespace, labels, and annotations |
| `spec` | Desired state for this particular kind |

Not every object has exactly the same `spec`. A Service spec has networking
fields; a Deployment spec has replica and pod-template fields; a Role spec has
permission rules.

### Metadata you will see

Common metadata includes:

```yaml
metadata:
  name: web
  namespace: training
  labels:
    app.kubernetes.io/name: web
    app.kubernetes.io/part-of: storefront
  annotations:
    example.com/owner: platform-team
```

- **name** identifies an object within its scope.
- **namespace** selects the namespace or OpenShift project for a namespaced
  object.
- **labels** are short indexed key-value data used for grouping and selection.
- **annotations** store non-identifying information not normally used for
  selection.

The server adds fields such as `uid`, `resourceVersion`, `creationTimestamp`,
and `managedFields`. Do not normally copy generated metadata into a new source
manifest.

### Namespaced and cluster-scoped resources

Most application objects are namespaced:

```text
Pods, Deployments, Services, ConfigMaps, Secrets, Roles, PVCs
```

Some objects are cluster-scoped:

```text
Nodes, PersistentVolumes, StorageClasses, ClusterRoles, ClusterOperators
```

Discover scope instead of guessing:

```bash
oc api-resources --namespaced=true
oc api-resources --namespaced=false
```

The same namespaced object name can exist in two projects. For example,
`training/web` and `production/web` are distinct Deployments.

### Objects can own other objects

Controllers create lower-level objects and record ownership. A common chain is:

```text
Deployment -> ReplicaSet -> Pod
```

Deleting a parent with normal cascading deletion can remove its owned
children. This is why you should identify ownership before deleting something:

```bash
oc get pod POD_NAME -o jsonpath='{.metadata.ownerReferences}'
```

### Finalizers

A **finalizer** tells the API that cleanup must finish before an object
disappears completely. An object with a deletion timestamp can remain in
`Terminating` while a responsible controller performs cleanup.

Do not remove finalizers as a first troubleshooting step. First identify which
controller owns the cleanup and why it cannot finish. Forced finalizer removal
can orphan infrastructure or data.

### API discovery is a working skill

Use:

```bash
oc api-resources
oc api-resources | less
oc explain pod
oc explain pod.spec
oc explain pod.spec.containers
oc explain deployment.spec.template.spec --recursive
```

`oc explain` reads the API schema exposed by the cluster. It is one of the
safest ways to find the correct field path during study and exam work.

## 6. YAML From the Beginning

YAML is a human-readable data format. Kubernetes accepts JSON at its API, and
CLI tools commonly convert YAML manifests into the required representation.
YAML is popular because nested data is easier for humans to read than large
JSON documents.

YAML is not a shell script and it is not a programming language. Indentation
expresses structure.

### The three building blocks

#### Mapping: named keys and values

A mapping is like a dictionary:

```yaml
name: web
replicas: 2
enabled: true
```

The space after `:` matters for ordinary key-value syntax.

#### Sequence: an ordered list

A sequence uses a dash:

```yaml
ports:
  - 8080
  - 8443
```

#### Scalar: one value

Scalars include strings, numbers, booleans, and null:

```yaml
name: web
replicas: 2
enabled: true
optionalValue: null
```

### Indentation rules

Use spaces, never tabs. Two spaces per nesting level is the convention used
throughout this handbook.

Correct:

```yaml
metadata:
  name: web
  labels:
    app: web
```

Incorrect:

```yaml
metadata:
name: web
    labels:
  app: web
```

Indentation answers this question:

> Which parent does this field belong to?

Here, `name` and `labels` belong to `metadata`, while `app` belongs to
`labels`.

### A list of mappings

Container definitions are a list. Each list item is itself a mapping:

```yaml
containers:
  - name: web
    image: registry.example.com/apps/web:1.0
    ports:
      - name: http
        containerPort: 8080
  - name: log-helper
    image: registry.example.com/tools/log-helper:2.0
```

Follow the first item:

1. `containers` begins a list.
2. `- name: web` begins the first list item.
3. `image` belongs to that same item because it aligns with `name`.
4. `ports` also belongs to the first container.
5. `- name: http` begins an item inside the ports list.
6. The second `- name: log-helper` aligns with the first container dash, so it
   begins the second container.

### Strings and quoting

Simple strings can be unquoted:

```yaml
name: web
```

Quote a value when it might be misunderstood, contains special punctuation, or
must remain a string:

```yaml
version: "1.0"
enabledText: "true"
message: "value: with a colon"
commentText: "value # this is data, not a comment"
```

Environment variable values must be strings:

```yaml
env:
  - name: WORKER_COUNT
    value: "3"
  - name: FEATURE_ENABLED
    value: "true"
```

Use lowercase YAML booleans when a field actually expects a boolean:

```yaml
readOnlyRootFilesystem: true
```

Be especially careful with memory. `512Mi` means mebibytes. `512m` does not
mean megabytes; lowercase `m` is a milli-unit.

### Comments

A comment begins with `#` outside a quoted string:

```yaml
replicas: 2  # Keep two interchangeable application pods.
```

Comments help humans but are not stored as part of the API object.

### Multiline text

The literal block marker `|` preserves line breaks:

```yaml
data:
  message.txt: |
    First line
    Second line
```

The folded marker `>` joins ordinary line breaks into spaces:

```yaml
data:
  description: >
    This long description is written on several source lines
    but is treated as one folded line of text.
```

### Empty values

These structures are different:

```yaml
emptyMap: {}
emptyList: []
emptyValue: null
emptyString: ""
```

Use the type required by the API schema.

### Multiple objects in one file

Separate complete YAML documents with `---`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: settings
data:
  color: blue
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 8080
      targetPort: 8080
```

One failed document can cause the command to return an error even if another
document was accepted. Always read the complete command output and verify each
resource.

### YAML is data, not shell expansion

This text:

```yaml
value: "$HOME"
```

normally sends the literal characters `$HOME`. The shell does not expand
variables inside a file just because `oc apply -f` reads it.

Be cautious with unquoted shell here-documents because the shell can expand
values before YAML reaches `oc`. A quoted delimiter prevents expansion:

```bash
cat <<'EOF'
value: "$HOME"
EOF
```

### Common YAML failures

| Symptom | Typical cause | Correction |
| --- | --- | --- |
| Parser says a mapping value is not allowed | Missing quote, misplaced colon, or bad indentation | Inspect the named line and its parent indentation |
| A list field is rejected as an object | Missing `-` before a list item | Check `oc explain` for the field type |
| Unknown field | Correct YAML syntax but wrong Kubernetes schema path | Use `oc explain KIND.FIELD` |
| Boolean or number rejected where string required | Value was not quoted | Quote environment and annotation values |
| Selector does not find pods | Label keys or values differ | Compare Service/Deployment selector with pod labels |
| Tab-related error | Editor inserted tab characters | Convert indentation to spaces |
| Unexpected value survives | Duplicate key in the YAML | Remove duplicates; every mapping key should appear once |
| Object created in wrong project | `metadata.namespace` or CLI context selected another project | Check `oc project` and `metadata.namespace` |

### Read this complete manifest from the outside inward

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: registry.access.redhat.com/ubi9/httpd-24:latest
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

The structure means:

1. Create an `apps/v1` `Deployment`.
2. Name it `web`.
3. Ask for two replicas.
4. The Deployment selector owns pods carrying label `app: web`.
5. The pod template gives every created pod that exact label.
6. Each pod has a container named `web`.
7. The container uses the stated image.
8. The named container port is `http`, number 8080.
9. The scheduler considers the requests.
10. Runtime use is bounded by the limits.

The two `app: web` label locations are intentionally the same:

```text
Deployment selector ----must match----> Pod-template labels
```

For a Deployment, that selector is effectively part of the object's identity
and cannot normally be changed after creation. Choose it carefully.

### Generate YAML instead of starting from a blank page

Use a CLI generator:

```bash
oc create deployment web \
  --image=registry.access.redhat.com/ubi9/httpd-24:latest \
  --replicas=2 \
  --dry-run=client \
  -o yaml > web-deployment.yaml
```

The command does not create the Deployment because of
`--dry-run=client`. It gives you a valid starting structure. Read and edit the
file before applying it.

Other useful generators:

```bash
oc create service clusterip web \
  --tcp=8080:8080 \
  --dry-run=client \
  -o yaml

oc create configmap app-settings \
  --from-literal=MODE=training \
  --dry-run=client \
  -o yaml

oc create secret generic app-credentials \
  --from-literal=username=student \
  --from-literal=password=example-only \
  --dry-run=client \
  -o yaml
```

### Validate before changing the cluster

Use this sequence:

```bash
oc apply --dry-run=client -f web-deployment.yaml
oc apply --dry-run=server -f web-deployment.yaml
oc diff -f web-deployment.yaml
oc apply -f web-deployment.yaml
```

- Client dry-run catches local parsing and some schema problems.
- Server dry-run asks the real API server to authenticate, authorize, admit,
  default, and validate without persisting the object.
- `oc diff` shows the intended change for an existing object. It can return a
  nonzero status when differences exist, so do not treat every nonzero result
  as a tool failure without reading the output.
- `oc apply` persists the declared configuration.

## 7. Labels, Selectors, and Annotations

Labels look small, but they connect many Kubernetes resources.

### Labels

A label is a key-value pair:

```yaml
metadata:
  labels:
    app: web
    tier: frontend
    environment: training
```

Use labels for properties that tools or selectors need to query.

List by label:

```bash
oc get pods -l app=web
oc get pods -l 'environment in (training,test)'
oc get all -l app=web
```

`oc get all` does not literally return every resource kind. It returns a useful
subset of common workload resources. Query important kinds explicitly.

### Selectors

A selector chooses objects by labels.

A Service might contain:

```yaml
spec:
  selector:
    app: web
```

It sends traffic only to ready pod endpoints with the matching label. If the
pods have `app: website`, the Service can exist and still have no application
endpoints.

Check both sides:

```bash
oc get service web -o jsonpath='{.spec.selector}{"\n"}'
oc get pods --show-labels
oc get endpointslices -l kubernetes.io/service-name=web
```

### Annotations

Annotations carry extra information that does not identify a selection group:

```yaml
metadata:
  annotations:
    example.com/change-ticket: "CHG-1042"
    example.com/description: "Training frontend"
```

Controllers can also use defined annotation keys as configuration. An
annotation is not automatically meaningful; the responsible component must
understand it.

### Recommended application labels

A useful, readable set includes:

```yaml
app.kubernetes.io/name: catalog
app.kubernetes.io/instance: catalog-training
app.kubernetes.io/component: frontend
app.kubernetes.io/part-of: storefront
app.kubernetes.io/managed-by: oc
```

These labels are conventions, not a replacement for understanding the exact
selectors used by your objects.

## 8. Workload Resources

A **workload** is an application or task running on the cluster. Pods are the
execution unit, but you usually create a controller rather than a standalone
pod.

### Pod

A **Pod** is the smallest Kubernetes deployable compute unit. It contains one
or more containers that share:

- one network identity and pod IP;
- the same network namespace;
- localhost communication;
- declared pod volumes; and
- one scheduling placement on a node.

Containers in one pod are tightly coupled. Do not put unrelated applications
in one pod merely to reduce the pod count.

Pods are replaceable. A replacement normally receives a new UID and IP.
Application design should not treat one pod as a permanent server.

### Multi-container pod patterns

Common reasons for more than one container include:

- a helper that prepares data before the application starts;
- a proxy or adapter that serves the main application;
- a tightly coupled helper that processes shared files.

An **init container** completes before ordinary application containers start.
Use it for ordered preparation, not as a permanent service.

### ReplicaSet

A **ReplicaSet** keeps a specified number of matching pods present. You rarely
author ReplicaSets directly because a Deployment manages them and adds rollout
history.

### Deployment

A **Deployment** is the normal controller for stateless, interchangeable
application replicas. It manages ReplicaSets and supports rolling updates and
rollback.

Relationship:

```text
Deployment web
  └── ReplicaSet web-7d9...
        ├── Pod web-7d9...-a
        └── Pod web-7d9...-b
```

Useful commands:

```bash
oc get deployment
oc describe deployment web
oc rollout status deployment/web
oc rollout history deployment/web
oc scale deployment/web --replicas=3
oc set image deployment/web web=NEW_IMAGE
oc rollout undo deployment/web
```

Verify after every change. A successful `oc set image` means the API accepted
the change; it does not prove the new pods became ready.

### StatefulSet

A **StatefulSet** manages pods that require stable identities, ordered behavior,
or a stable relationship with persistent storage. Its pods are not treated as
fully interchangeable.

Typical uses include clustered databases and applications requiring numbered
members. A StatefulSet is not a magic database high-availability solution; the
application must still understand replication, consistency, backup, and
recovery.

### DaemonSet

A **DaemonSet** aims to run a pod on every eligible node, or every node in a
selected group. Typical examples include node-level agents, drivers, and log
collectors.

Adding an eligible node causes the DaemonSet controller to create a pod for it.

### Job

A **Job** runs one or more pods to successful completion. Use it for a finite
task such as a migration or report generation, not a continuously available
web server.

### CronJob

A **CronJob** creates Jobs on a schedule. A schedule does not guarantee that
the application is safe to run twice. Design the task to handle retries and
overlap according to its requirements.

### Choose the controller

| Need | Normal starting resource |
| --- | --- |
| Stateless long-running replicas | Deployment |
| Stable identity or per-member storage | StatefulSet |
| One pod on each eligible node | DaemonSet |
| One finite task | Job |
| Repeated scheduled task | CronJob |
| Package lifecycle knowledge as software | Operator and custom resources |
| Quick temporary troubleshooting process | Pod, often generated with `oc run` |

## 9. Pod Lifecycle and Health

Creating a pod object is not the same as having a ready application.

### Pod phase

Common pod phases are:

- `Pending`: accepted, but one or more containers are not ready to run;
- `Running`: assigned to a node and at least one container is running or
  starting/restarting;
- `Succeeded`: all containers terminated successfully and will not restart;
- `Failed`: all containers terminated and at least one failed, or the pod could
  not complete successfully; and
- `Unknown`: the control plane cannot reliably obtain the pod state.

Text such as `CrashLoopBackOff` and `ImagePullBackOff` is commonly shown by CLI
status summaries, but it is not a pod phase. It describes a container waiting
reason.

### Container states

A container can be:

- waiting;
- running; or
- terminated.

Inspect exact state and reason:

```bash
oc describe pod POD_NAME
oc get pod POD_NAME -o yaml
oc get pod POD_NAME \
  -o jsonpath='{.status.containerStatuses[*].state}{"\n"}'
```

### Restart policy

The pod-level `restartPolicy` is:

- `Always`;
- `OnFailure`; or
- `Never`.

For ordinary Deployment pods, `Always` is used. A container restart inside the
same pod is different from a controller creating a replacement pod.

### Readiness probe

A **readiness probe** asks whether a container is ready to receive Service
traffic. A failed readiness probe removes the pod from ready endpoints but does
not by itself restart the container.

### Liveness probe

A **liveness probe** asks whether a running container has become unhealthy in a
way that requires restart. A bad liveness probe can create an unnecessary
restart loop.

### Startup probe

A **startup probe** protects slow-starting applications. Until it succeeds,
liveness and readiness checks do not take over in the ordinary way.

### Probe example

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: http
  initialDelaySeconds: 10
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /health/ready
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5
```

The named port `http` must exist in the same container. The endpoint and timing
must reflect the real application, not a copied example.

### Graceful termination

When a pod is terminated, Kubernetes normally gives containers a grace period.
Applications should handle the termination signal, stop accepting new work,
finish or transfer in-flight work, and exit. Force deletion skips part of the
normal safety process and should not be your default response.

## 10. Configuration: ConfigMaps and Secrets

Do not rebuild an image for every environment-specific setting.

### ConfigMap

A **ConfigMap** stores non-secret configuration as key-value data or files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-settings
data:
  MODE: training
  greeting.txt: |
    Welcome to the fundamentals project.
```

Create from literals or files:

```bash
oc create configmap web-settings \
  --from-literal=MODE=training \
  --from-file=greeting.txt \
  --dry-run=client \
  -o yaml
```

### Secret

A **Secret** stores sensitive values intended for controlled use by workloads
and platform components.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-credentials
type: Opaque
stringData:
  username: student
  password: example-only-change-me
```

`stringData` lets you supply clear text to the API request; the API stores it
in the Secret's `data` representation. Base64 encoding is not encryption.
Access control, encryption configuration, secure delivery, rotation, and not
committing real secrets to Git still matter.

Avoid putting real passwords directly on a shared command line because shell
history and process inspection can expose them.

### Environment variable consumption

```yaml
env:
  - name: MODE
    valueFrom:
      configMapKeyRef:
        name: web-settings
        key: MODE
  - name: APP_USERNAME
    valueFrom:
      secretKeyRef:
        name: web-credentials
        key: username
```

Environment variables are read when the container starts. Changing the source
object does not rewrite the environment of an already running process. A
rollout or application-specific reload may be required.

### Volume consumption

ConfigMaps and Secrets can also be projected as files. Projected files can be
updated by the platform, but the application must notice and reload them.
Sub-path mounts and application caching can change update behavior. Always
verify with the application you are running.

### Safe inspection

```bash
oc get configmap web-settings -o yaml
oc describe secret web-credentials
```

`oc describe secret` avoids printing decoded secret values, but metadata can
still be sensitive. Do not casually use `oc get secret -o yaml` in logs,
screenshots, or shared terminal output.

## 11. Networking From Pod to User

Networking becomes easier when you separate four identities:

1. container port: where the process listens inside the pod;
2. pod IP: temporary address of one pod;
3. Service IP and DNS: stable access to a selected pod set; and
4. Route host name: external HTTP, HTTPS, or TLS access through OpenShift
   ingress.

### Pod networking

Each pod receives a cluster network address. Containers in the same pod share
that network identity and can communicate through `localhost`.

Pod IPs are not stable identities. Replacement pods normally have different
IPs. Clients should use a Service or another appropriate discovery mechanism.

### Service

A **Service** provides a stable virtual endpoint for a changing set of pods.
Its selector normally chooses pods by labels.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - name: http
      port: 8080
      targetPort: http
```

- `port` is the port offered by the Service.
- `targetPort` is the port on selected pods. It can be a number or a named
  container port.
- The Service controller maintains EndpointSlices describing ready backends.

Trace it:

```text
Client -> Service web:8080 -> ready Pod endpoint -> container port http
```

### Service DNS

Within the same project, a pod can normally resolve:

```text
web
```

Across projects, use:

```text
web.PROJECT.svc
```

The fuller cluster DNS suffix depends on cluster configuration. Prefer Service
DNS over hard-coded Service or pod IP addresses.

### Service types

| Type | Foundation meaning |
| --- | --- |
| `ClusterIP` | Internal virtual IP; default Service type |
| `NodePort` | Exposes a port on nodes in addition to a ClusterIP |
| `LoadBalancer` | Requests an external load balancer from supported infrastructure |
| `ExternalName` | Returns a DNS alias for an external name |

A **headless Service** uses `clusterIP: None` to provide direct discovery
instead of one virtual ClusterIP. It is common with StatefulSets.

### OpenShift Route

A **Route** maps an external host name to a Service through the OpenShift
ingress router.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: web
spec:
  to:
    kind: Service
    name: web
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

Common TLS termination modes are:

- **edge**: the router terminates TLS and forwards to the Service;
- **reencrypt**: the router terminates client TLS and establishes TLS to the
  backend; and
- **passthrough**: encrypted traffic passes to the backend, which terminates
  TLS.

Routes are designed for HTTP, HTTPS, and TLS routing. Non-HTTP protocols may
require a `LoadBalancer` Service, NodePort, MetalLB, or infrastructure-specific
configuration covered later in the handbook.

### NetworkPolicy

A **NetworkPolicy** permits selected network traffic at the pod level. Policies
are additive:

- if no policy selects a pod for a direction, that direction is ordinarily not
  isolated by NetworkPolicy;
- once a policy selects the pod for ingress or egress, allowed traffic is the
  union of applicable policy rules for that direction; and
- both source egress and destination ingress policy can matter.

Labels therefore influence both application grouping and security.

Do not create a default-deny policy before identifying required DNS, ingress,
monitoring, database, and platform flows.

### Network troubleshooting chain

Check in this order:

```bash
oc get pod -l app=web -o wide
oc get service web -o yaml
oc get endpointslices -l kubernetes.io/service-name=web
oc get route web -o yaml
oc get networkpolicy
```

If the Service has no endpoints, test selector and readiness before blaming
the router or external DNS.

## 12. Storage Without the Mystery

Container writable layers and pod-local ephemeral data are not durable
application storage.

### Ephemeral storage

Data written into a container's writable layer can disappear when the
container or pod is replaced. An `emptyDir` volume survives container restarts
within the same pod but is removed when that pod ends.

Use ephemeral storage for caches and temporary work, not for irreplaceable
business data.

### PersistentVolume

A **PersistentVolume** (PV) is cluster-scoped storage capacity. It can be
created by an administrator or dynamically provisioned through a storage
system.

### PersistentVolumeClaim

A **PersistentVolumeClaim** (PVC) is a namespaced request for storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

The pod refers to the claim, not usually to a specific PV:

```yaml
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: web-data
  containers:
    - name: web
      image: IMAGE
      volumeMounts:
        - name: data
          mountPath: /var/lib/web
```

### StorageClass

A **StorageClass** describes a storage service and provisioning behavior. A
default StorageClass can dynamically create a suitable PV when a PVC is
submitted.

Inspect:

```bash
oc get storageclass
oc get pvc
oc describe pvc web-data
oc get pv
```

A PVC in `Pending` often means no matching PV or dynamic provisioner can
satisfy its class, size, access mode, topology, or other requirements.

### Access modes

Common requested modes are:

- `ReadWriteOnce` (RWO): read-write mounting by one node;
- `ReadOnlyMany` (ROX): read-only mounting by many nodes;
- `ReadWriteMany` (RWX): read-write mounting by many nodes; and
- `ReadWriteOncePod` (RWOP): read-write mounting by one pod when supported.

An access mode describes volume capability and mounting rules. It does not
replace filesystem ownership and permissions inside the mounted filesystem.

### Reclaim behavior

The PV reclaim policy influences what happens to storage after its claim is
released. `Delete` and `Retain` have very different data-lifecycle outcomes.
Inspect policy before deleting a PVC.

A PVC is not a backup. Storage snapshots, application-consistent backup,
off-cluster copies, and tested recovery are separate concerns.

## 13. CPU, Memory, and Scheduling

The scheduler needs resource information to place pods responsibly.

### Requests

A **request** is the amount used for scheduling and resource accounting.

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

`100m` CPU means one tenth of one CPU core.

### Limits

A **limit** is an upper bound enforced by runtime mechanisms:

```yaml
resources:
  limits:
    cpu: 500m
    memory: 256Mi
```

Excess CPU use is normally throttled. Exceeding a hard memory limit can cause
the container to be terminated for out-of-memory use.

### Resource units

| Value | Meaning |
| --- | --- |
| `1` CPU | One CPU core or equivalent share |
| `500m` CPU | Half a CPU |
| `128Mi` memory | 128 mebibytes |
| `1Gi` memory | One gibibyte |

Never use `m` when you mean `Mi` for memory.

### Quality of Service

Pod resource declarations influence Kubernetes QoS classification:

- **Guaranteed**: qualifying containers have equal CPU and memory requests and
  limits;
- **Burstable**: at least one request or limit exists, but Guaranteed
  requirements are not met; and
- **BestEffort**: no CPU or memory requests or limits are set.

QoS can influence eviction behavior during node pressure. It does not make an
application highly available.

### Node selection

Scheduling controls include:

- `nodeSelector`: simple required label matching;
- node affinity: expressive preferred or required node matching;
- pod affinity and anti-affinity: placement relative to other pods;
- topology spread constraints: distribute pods across failure domains;
- taints: repel pods from nodes; and
- tolerations: allow, but do not necessarily force, placement on tainted
  nodes.

A toleration is not a permission to use every protected host resource. RBAC,
SCC, resource availability, and other admission or scheduling rules still
apply.

### Why a pod remains Pending

Use events:

```bash
oc describe pod POD_NAME
oc get events --sort-by=.lastTimestamp
```

Possible causes include insufficient CPU or memory, unmatched selectors,
untolerated taints, unbound PVCs, topology conflicts, quota, admission denial,
or image and startup issues.

## 14. Identity, RBAC, Service Accounts, and SCC

Security questions become clearer when you separate identity, API permission,
and pod runtime permission.

### Authentication

Authentication proves identity. A person can authenticate through configured
OpenShift OAuth identity providers. The CLI stores connection and credential
context in kubeconfig.

Inspect your current identity:

```bash
oc whoami
oc whoami --show-server
oc whoami --show-token
```

Treat tokens like passwords. Do not paste `--show-token` output into notes,
screenshots, chat, or source control.

### Authorization and RBAC

Authorization decides whether the authenticated identity may perform a verb on
a resource.

RBAC building blocks:

| Object | Scope | Purpose |
| --- | --- | --- |
| Role | Namespace/project | Permission rules within one namespace |
| RoleBinding | Namespace/project | Grants a Role or ClusterRole to subjects in that namespace |
| ClusterRole | Cluster | Reusable or cluster-wide permission rules |
| ClusterRoleBinding | Cluster | Grants a ClusterRole across the cluster |

Permission rules combine:

- API groups;
- resources and sometimes resource names; and
- verbs such as `get`, `list`, `watch`, `create`, `update`, `patch`, and
  `delete`.

Test permission:

```bash
oc auth can-i create deployments
oc auth can-i delete pods -n training
oc auth can-i --list
oc adm policy who-can get secrets -n training
```

`Forbidden` means the request reached the API but the current identity lacks an
applicable permission. Do not work around that by copying an administrator
token.

### Service accounts

A **service account** is a namespaced identity for workloads and automation.
Every pod uses a service account; if none is named, it normally uses the
project's `default` service account.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: report-reader
```

Reference it in a pod template:

```yaml
spec:
  serviceAccountName: report-reader
```

Grant only permissions required by that workload. Do not use a human's token
inside an application.

### Security Context Constraints

OpenShift **Security Context Constraints** (SCCs) control whether a pod may be
admitted with requested runtime privileges and security settings. SCCs can
control items such as:

- privileged containers;
- host networking and namespaces;
- host directory volumes;
- Linux capabilities;
- user and group IDs;
- SELinux context;
- privilege escalation;
- seccomp profiles; and
- allowed volume types.

RBAC and SCC answer different questions:

```text
RBAC: May this identity ask the API to create this resource?
SCC: Is this pod's requested security context permitted?
```

Do not edit default SCCs. Use least privilege and create a carefully scoped
custom solution only when required by a real workload.

### Arbitrary user IDs

OpenShift commonly runs application containers with an assigned non-root UID.
A well-designed image should:

- not require UID 0;
- avoid a hard-coded user-ID ownership assumption;
- allow the assigned user to write only required paths;
- listen on a non-privileged port where possible; and
- avoid unnecessary Linux capabilities.

When an image works with Podman as root but fails in OpenShift, inspect file
permissions and runtime assumptions before granting a more permissive SCC.

### Three security layers to remember

```text
API access       -> authentication + RBAC
Pod admission    -> SCC and admission controls
Network access   -> NetworkPolicy, Service, Route, and infrastructure rules
```

Passing one layer does not imply passing the others.

## 15. Operators and Custom Resources

An **Operator** is software that watches Kubernetes APIs and automates
application or platform lifecycle knowledge.

### CustomResourceDefinition and custom resource

A **CustomResourceDefinition** (CRD) extends the Kubernetes API with a new
resource type. A **custom resource** (CR) is an object of that type.

Example mental model:

```text
CRD defines Database
Database object says replicas: 3 and backupSchedule: ...
Database Operator observes it and manages lower-level resources
```

Creating a custom resource without its responsible controller does not
magically perform the application lifecycle.

### Cluster Operators

OpenShift platform capabilities are divided among Cluster Operators. The
Cluster Version Operator coordinates platform payload and operator updates.
Other operators manage areas such as networking, authentication, storage
integration, image registry, monitoring, and machine configuration.

Foundation health check:

```bash
oc get clusteroperators
```

Important conditions include:

- `Available=True`;
- `Progressing=False` after normal work settles; and
- `Degraded=False`.

Read the actual condition messages instead of diagnosing only from the three
columns.

### OLM-managed Operators

Operator Lifecycle Manager (OLM) helps install and manage optional Operators.
Common objects include:

- `CatalogSource`;
- `Subscription`;
- `InstallPlan`;
- `ClusterServiceVersion`; and
- `OperatorGroup`.

Cluster Operators and OLM-managed add-on Operators are related concepts but
are not managed by the same lifecycle mechanism.

## 16. Images, ImageStreams, and Builds

### ImageStream

An OpenShift **ImageStream** is an API abstraction that tracks image references
by tags and digests. It does not contain the image layers themselves.

This separation is important:

```text
Registry repository -> stores image content
ImageStream          -> tracks image references inside OpenShift
```

An ImageStreamTag such as `web:latest` can point to a particular image digest.
Build and deployment workflows can react when an ImageStreamTag changes.

Inspect:

```bash
oc get imagestream
oc describe imagestream web
oc get imagestreamtag web:latest
```

### Build

A build transforms source and inputs into an output, commonly a runnable
container image. A **BuildConfig** describes an OpenShift build process and its
triggers.

Foundation flow:

```text
Source code + build strategy + builder image
                    |
                    v
                 Build pod
                    |
                    v
          output image in a registry
                    |
                    v
            ImageStreamTag update
                    |
                    v
              application rollout
```

Do not confuse these files:

- **Containerfile/Dockerfile**: instructions for building image content;
- **Kubernetes/OpenShift YAML**: API objects describing how the platform should
  run or manage something; and
- **application configuration**: settings read by the application.

## 17. Everyday `oc` Navigation

The general command shape is:

```text
oc VERB RESOURCE [NAME] [OPTIONS]
```

Examples:

```bash
oc get pods
oc describe deployment web
oc delete configmap old-settings
```

### Understand placeholders

In documentation:

```bash
oc describe pod <pod-name>
```

`<pod-name>` is a placeholder. Replace the whole placeholder, including angle
brackets:

```bash
oc describe pod web-7d9f6b8d8c-k4m2p
```

Do not type the angle brackets; the shell interprets them as redirection.

### Connect and orient

```bash
oc login https://api.CLUSTER_NAME:6443
oc whoami
oc whoami --show-server
oc project
oc get clusterversion
oc get clusteroperators
oc get nodes
```

Avoid `--insecure-skip-tls-verify=true` unless the training environment
explicitly requires it and you understand why trust validation is unavailable.

### Project scope

```bash
oc get projects
oc project training
oc get pods
oc get pods -n another-project
oc get pods -A
```

- current project is the default scope;
- `-n NAME` selects one project for that command;
- `-A` or `--all-namespaces` requests all namespaces when supported and
  authorized.

Before a destructive command:

```bash
oc project
oc whoami
```

### Discover resource names

```bash
oc api-resources
oc get pods
oc get pod
oc get po
```

Plural, singular, and short names can refer to the same resource, but use
readable names in study notes and scripts when ambiguity is possible.

### Get, describe, and explain are different

```bash
oc get deployment web
oc describe deployment web
oc explain deployment.spec.strategy
```

- `get` reads objects and supports structured output.
- `describe` presents human-oriented details and related events.
- `explain` describes API schema fields, not one live object's current value.

### Output formats

```bash
oc get pods -o wide
oc get deployment web -o yaml
oc get deployment web -o json
oc get pods -o name
oc get deployment web \
  -o jsonpath='{.spec.replicas}{"\n"}'
```

YAML and JSON output contain generated fields. Do not copy the entire live
object as a new manifest without removing status and server-managed metadata.

### Logs and interactive access

```bash
oc logs POD_NAME
oc logs POD_NAME -c CONTAINER_NAME
oc logs POD_NAME --previous
oc logs -f POD_NAME
oc exec POD_NAME -- COMMAND
oc rsh POD_NAME
```

Use `--` to separate `oc` options from the command executed inside the
container.

### Create, apply, edit, patch, and replace

- `oc create -f FILE` creates objects and normally fails if they already exist.
- `oc apply -f FILE` declaratively reconciles file-managed configuration.
- `oc edit KIND NAME` opens the live object in an editor.
- `oc patch KIND NAME ...` changes selected fields.
- `oc replace -f FILE` replaces an existing object's supplied representation
  and has stricter existence/resource-version behavior.

For durable work, keep a clean source manifest and use a consistent management
method. Mixing several tools that own the same field can create field-management
conflicts or overwrite changes.

### Wait and verify

```bash
oc rollout status deployment/web
oc wait --for=condition=Available deployment/web --timeout=120s
oc get pods -l app=web
oc get events --sort-by=.lastTimestamp
```

Never treat “created,” “configured,” or command exit status alone as proof that
the application is working.

### Deletion

Preview identity and scope:

```bash
oc get deployment web
oc project
```

Then delete the exact resource:

```bash
oc delete deployment web
```

Deleting a project removes namespaced content in that project. It is a
high-impact operation:

```bash
oc delete project training
```

Do not use broad selectors or `--all` until you have displayed exactly what
they match.

## 18. Resource Relationship Map

For a typical web application:

```text
Deployment
    |
    | owns
    v
ReplicaSet
    |
    | owns
    v
Pods <---------------- ConfigMap / Secret
  |  \---------------- PVC -> PV -> storage backend
  |
  | selected by labels
  v
Service
  |
  | targeted by
  v
Route -> OpenShift ingress router -> external client
```

Security and operations cross the picture:

```text
User ---------- RBAC ----------> API objects
Pod -------- ServiceAccount ---> API permissions
Pod request ---- SCC -----------> admitted or denied
Pod traffic -- NetworkPolicy ---> allowed or denied
Controllers --------------------> reconcile desired state
```

When troubleshooting, locate the broken connection in this map.

## 19. A Safe Manifest Workflow

Use this repeatable process for later chapters and exams.

### Step 1: orient

```bash
oc whoami
oc whoami --show-server
oc project
```

### Step 2: discover the API

```bash
oc api-resources | grep -i deployment
oc explain deployment
oc explain deployment.spec.template.spec.containers
```

### Step 3: generate or copy a minimal structure

```bash
oc create deployment web \
  --image=registry.access.redhat.com/ubi9/httpd-24:latest \
  --dry-run=client \
  -o yaml > web.yaml
```

### Step 4: edit deliberately

Change only fields you understand. Keep indentation consistent. Compare every
selector with the labels it is intended to match.

### Step 5: perform dry runs

```bash
oc apply --dry-run=client -f web.yaml
oc apply --dry-run=server -f web.yaml
```

### Step 6: review differences

```bash
oc diff -f web.yaml
```

### Step 7: apply

```bash
oc apply -f web.yaml
```

### Step 8: wait for the controller

```bash
oc rollout status deployment/web
```

### Step 9: verify from several layers

```bash
oc get deployment web
oc get replicasets
oc get pods -l app=web
oc get events --sort-by=.lastTimestamp
```

If networking is involved:

```bash
oc get service web
oc get endpointslices -l kubernetes.io/service-name=web
oc get route web
```

### Step 10: preserve the known-good manifest

Store the reviewed YAML without tokens or passwords. The file becomes a
repeatable description and troubleshooting reference.

## 20. Guided Foundation Lab

Run this lab only on a personal or authorized training cluster. Do not use a
production or shared namespace.

### Lab goal

You will create:

```text
Project fundamentals
  ├── Deployment web (two pods)
  ├── Service web
  └── Route web
```

### Step 1: create and select a project

```bash
oc new-project fundamentals
oc project
```

Expected current project:

```text
fundamentals
```

If the project already exists and belongs to you:

```bash
oc project fundamentals
```

### Step 2: save the complete manifest

Create `fundamentals-web.yaml` with:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
    app.kubernetes.io/name: web
    app.kubernetes.io/part-of: fundamentals
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        app.kubernetes.io/name: web
        app.kubernetes.io/part-of: fundamentals
    spec:
      containers:
        - name: web
          image: registry.access.redhat.com/ubi9/httpd-24:latest
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: web
  labels:
    app: web
spec:
  selector:
    app: web
  ports:
    - name: http
      port: 8080
      targetPort: http
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: web
  labels:
    app: web
spec:
  to:
    kind: Service
    name: web
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

### Step 3: ask the API to validate without saving

```bash
oc apply --dry-run=server -f fundamentals-web.yaml
```

Every object must be accepted. If the Route API or image is unavailable in
your particular training environment, stop and inspect the exact message
instead of deleting unrelated fields.

### Step 4: apply

```bash
oc apply -f fundamentals-web.yaml
```

Expected object types:

```text
deployment.apps/web
service/web
route.route.openshift.io/web
```

### Step 5: trace ownership

```bash
oc get deployment web
oc get replicasets
oc get pods -l app=web
oc get pods -l app=web \
  -o custom-columns=NAME:.metadata.name,OWNER:.metadata.ownerReferences[0].kind,NODE:.spec.nodeName,PHASE:.status.phase
```

You should see the Deployment, one current ReplicaSet, and two pods when the
rollout is complete.

### Step 6: wait and inspect

```bash
oc rollout status deployment/web --timeout=180s
oc get pods -l app=web -o wide
oc describe deployment web
```

If the rollout fails, continue to the troubleshooting section instead of
pretending the lab passed.

### Step 7: trace network selection

```bash
oc get service web -o yaml
oc get pods -l app=web --show-labels
oc get endpointslices -l kubernetes.io/service-name=web
oc get route web
```

The EndpointSlice should contain ready pod backends. The Service selector and
pod label must both be `app: web`.

### Step 8: get the external host

```bash
oc get route web -o jsonpath='{.spec.host}{"\n"}'
```

If your workstation can reach the training cluster's applications domain, test
the printed host with a browser. A Route object can exist even when external
DNS or workstation connectivity is unavailable, so separate object validation
from external-path validation.

### Step 9: scale declaratively

Edit the manifest:

```yaml
spec:
  replicas: 3
```

Apply and verify:

```bash
oc apply -f fundamentals-web.yaml
oc rollout status deployment/web
oc get pods -l app=web
```

Explain what happened: the Deployment desired state changed, its controller
adjusted the ReplicaSet, and the ReplicaSet created one more pod.

### Step 10: observe before cleanup

```bash
oc get all
oc get route
oc get events --sort-by=.lastTimestamp
```

Remember that `oc get all` does not include every resource type, which is why
the Route is queried separately.

### Step 11: cleanup

Confirm that this is your disposable project:

```bash
oc project
oc get project fundamentals
```

Then:

```bash
oc delete project fundamentals
```

Do not run that command in a project containing work you need.

## 21. Troubleshooting Without Guessing

Use the sequence:

```text
Symptom -> scope -> evidence -> cause -> smallest fix -> verification
```

### Start with scope

```bash
oc whoami
oc whoami --show-server
oc project
oc get RESOURCE NAME
```

Many “missing resource” problems are wrong-cluster or wrong-project problems.

### Read conditions and events

```bash
oc describe pod POD_NAME
oc get events --sort-by=.lastTimestamp
oc get deployment web -o yaml
```

Events are time-limited evidence, not permanent logs. Capture relevant evidence
while the problem is active.

### Read current and previous logs

```bash
oc logs POD_NAME
oc logs POD_NAME -c CONTAINER_NAME
oc logs POD_NAME --previous
oc logs POD_NAME --tail=100 --timestamps
```

`--previous` is valuable after a container restarts.

### Common symptoms

| Symptom | Evidence to inspect first | Common cause categories |
| --- | --- | --- |
| `Pending` | Pod events, PVC, requests, selectors, taints | Scheduling, storage, quota, admission |
| `ImagePullBackOff` | Pod events and image reference | Bad name/tag, registry access, pull secret, mirror |
| `CrashLoopBackOff` | Current/previous logs, command, probes, mounts | Process exits, bad configuration, permission, liveness |
| Running but not Ready | Readiness probe and application logs | Wrong path/port, dependency unavailable, slow start |
| Service has no endpoints | Service selector, pod labels, readiness | Selector mismatch or no ready matching pods |
| Route returns no application response | Route, Service, endpoints, target port, router | Broken chain between route and pods |
| `Forbidden` | `oc auth can-i`, identity, binding | Missing RBAC permission |
| Pod rejected by SCC | API error, service account, security context | Image/runtime privilege assumption |
| PVC Pending | PVC events, StorageClass, access mode, size | No matching/provisionable storage |
| `OOMKilled` | Container last state, limit, application metrics | Memory limit too low or application growth |

### Do not destroy evidence

Deleting a failing pod may make a replacement appear, but it can also hide
events, current filesystem state, and the original timing. Collect:

```bash
oc describe pod POD_NAME
oc logs POD_NAME --all-containers
oc logs POD_NAME --all-containers --previous
oc get pod POD_NAME -o yaml
```

before deletion when time and safety allow.

### Fix one layer and verify that layer

Example: a Service has no endpoints.

1. Compare selector and pod labels.
2. Correct only the mismatch.
3. Apply the corrected resource.
4. Verify EndpointSlices populate.
5. Test Service access.
6. Test the Route only after Service access is correct.

This prevents random changes across several layers.

## 22. How to Read Later Chapters

Whenever a later chapter presents YAML:

1. identify `apiVersion` and `kind`;
2. identify whether the object is namespaced or cluster-scoped;
3. read `metadata.name`, namespace, labels, and annotations;
4. find selectors and compare them with target labels;
5. read `spec` as desired state;
6. use `oc explain` for unknown field paths;
7. identify which controller will reconcile it;
8. predict which lower-level objects should appear;
9. identify status and events that prove success; and
10. identify a safe rollback or cleanup action.

Whenever a later chapter presents a command:

1. replace placeholders;
2. confirm cluster, user, and project;
3. identify whether it reads or changes state;
4. predict the resource and field affected;
5. run the narrowest command;
6. read all output; and
7. verify with a separate observation command.

## 23. Foundation Command Reference

### Identity and context

```bash
oc login API_URL
oc whoami
oc whoami --show-server
oc project
oc config current-context
oc config get-contexts
```

### Discovery

```bash
oc api-resources
oc api-versions
oc explain RESOURCE
oc explain RESOURCE.FIELD
oc options
oc COMMAND --help
```

### Observation

```bash
oc get RESOURCE
oc get RESOURCE NAME -o yaml
oc get pods -o wide
oc describe RESOURCE NAME
oc get events --sort-by=.lastTimestamp
oc logs POD
oc logs POD --previous
```

### Selection

```bash
oc get pods -n PROJECT
oc get pods -A
oc get pods -l KEY=VALUE
oc get pods --field-selector=status.phase=Running
```

Labels and fields are different selection mechanisms. Supported field
selectors vary by resource.

### Declarative changes

```bash
oc apply --dry-run=client -f FILE
oc apply --dry-run=server -f FILE
oc diff -f FILE
oc apply -f FILE
oc delete -f FILE
```

### Workloads

```bash
oc create deployment NAME --image=IMAGE
oc scale deployment/NAME --replicas=NUMBER
oc set image deployment/NAME CONTAINER=IMAGE
oc rollout status deployment/NAME
oc rollout history deployment/NAME
oc rollout undo deployment/NAME
```

### Networking

```bash
oc expose deployment/NAME --port=8080
oc expose service/NAME
oc get service
oc get endpointslices
oc get route
```

`oc expose deployment` normally creates a Service. `oc expose service`
normally creates a Route in OpenShift. Always inspect what was created.

### Authorization

```bash
oc auth can-i VERB RESOURCE
oc auth can-i VERB RESOURCE -n PROJECT
oc auth can-i --list
oc get role,rolebinding
oc get clusterrole,clusterrolebinding
```

## 24. Essential Vocabulary

| Term | Easy-English meaning |
| --- | --- |
| Admission | API processing that validates or modifies a request after authorization |
| Annotation | Non-identifying metadata used by people or controllers |
| API | Structured interface used to read and change cluster resources |
| API server | Front door for Kubernetes and OpenShift API requests |
| Build | A finite process that produces an output, commonly an image |
| BuildConfig | OpenShift object describing build inputs, strategy, output, and triggers |
| Cluster | Control plane plus nodes and cluster services |
| ClusterIP | Default internal Service virtual IP type |
| Cluster Operator | Operator managing an installed OpenShift platform capability |
| ConfigMap | Namespaced non-secret configuration data |
| Container | Running process instance created from an image |
| Controller | Reconciliation loop that works toward desired state |
| CR | Object created from a custom resource definition |
| CRD | API extension defining a custom resource kind |
| CRI-O | Container runtime used by OpenShift nodes for Kubernetes workloads |
| CronJob | Controller that creates Jobs on a schedule |
| DaemonSet | Controller that runs a pod on each eligible node |
| Deployment | Controller for rolling, scalable, normally stateless replicas |
| Desired state | Configuration requested in an object's spec |
| Digest | Immutable content identifier for an image |
| EndpointSlice | API object recording network backends for a Service |
| etcd | Consistent store for Kubernetes API state |
| Event | Time-limited record describing a cluster occurrence |
| Finalizer | Cleanup gate that delays final object removal |
| Image | Read-only application package used to create containers |
| ImageStream | OpenShift object tracking related image references and tags |
| Ingress | Kubernetes HTTP/HTTPS external-routing API |
| Job | Controller for a finite task that must complete |
| Kubeconfig | Client configuration containing clusters, users, and contexts |
| Kubelet | Node agent that makes assigned pods run and reports their status |
| Kubernetes | Container orchestration foundation used by OpenShift |
| Label | Indexed key-value metadata used for grouping and selection |
| Limit | Maximum CPU or memory a container is allowed to use |
| Manifest | YAML or JSON document describing one or more API objects |
| Namespace | Scope that separates many Kubernetes resources |
| NetworkPolicy | Rules controlling selected pod ingress and egress traffic |
| Node | Machine participating in the cluster |
| Observed state | Current state reported in status |
| Operator | Controller that automates domain-specific lifecycle knowledge |
| OLM | Operator Lifecycle Manager for optional add-on Operators |
| OpenShift project | Kubernetes namespace presented with OpenShift behavior |
| PersistentVolume | Cluster-scoped persistent storage capacity |
| PersistentVolumeClaim | Namespaced request for persistent storage |
| Pod | Smallest scheduled compute unit; one or more shared-context containers |
| Probe | Health check used for startup, readiness, or liveness |
| Project | OpenShift user-facing namespace |
| RBAC | Role-based authorization for API actions |
| Reconciliation | Repeated work to bring observed state toward desired state |
| Registry | Service storing and distributing container images |
| ReplicaSet | Controller maintaining a number of matching pods |
| Request | CPU or memory amount used for scheduling and accounting |
| Route | OpenShift external host-to-Service mapping for HTTP/HTTPS/TLS |
| SCC | OpenShift Security Context Constraints for pod admission |
| Scheduler | Control-plane component selecting nodes for unscheduled pods |
| Secret | Namespaced object for controlled sensitive data |
| Selector | Expression selecting objects by labels or supported fields |
| Service | Stable network endpoint for a changing group of pod backends |
| Service account | Namespaced API identity used by pods and automation |
| Spec | Desired configuration of an API object |
| StatefulSet | Controller for pods with stable identities or storage relationships |
| Status | Platform-reported observed condition of an object |
| StorageClass | Cluster-scoped description of storage provisioning behavior |
| Tag | Human-friendly image reference that can move to another digest |
| Taint | Node property that repels pods without a matching toleration |
| Toleration | Pod rule allowing scheduling consideration on a matching tainted node |
| UID | Server-generated unique identity for one object instance |
| YAML | Indentation-based data format commonly used for manifests |

## 25. Readiness Check

You are ready for Chapter 01 when you can answer these without guessing.

### Questions

1. What is the difference between an image and a container?
2. Why does Kubernetes use pods rather than directly treating one container as
   every management object?
3. What is reconciliation?
4. What is the difference between `spec` and `status`?
5. Which component accepts API requests?
6. Which component chooses a node for a new pod?
7. Which node agent works with CRI-O?
8. What does OpenShift add beyond ordinary Kubernetes?
9. What four top-level manifest fields must you immediately recognize?
10. Why are spaces and indentation important in YAML?
11. What does a dash mean in YAML?
12. Why should an environment value such as `3` normally be quoted?
13. Why must a Deployment selector match its pod-template labels?
14. Why do you normally create a Deployment instead of a standalone Pod?
15. When would a StatefulSet be more suitable than a Deployment?
16. What does a readiness probe change?
17. Why should clients not connect directly to pod IPs?
18. How does a Service find its backends?
19. What does an OpenShift Route target?
20. What is the difference between a PV and a PVC?
21. What is the difference between a resource request and a limit?
22. What is a service account?
23. What different questions do RBAC and SCC answer?
24. Why is base64 data in a Secret not the same as encrypted data?
25. What is an Operator?
26. Why can `oc apply` succeed while the application still fails?
27. What three commands would you start with for a crashing container?
28. Why can a running pod be absent from a Service's ready endpoints?
29. Why should you not remove finalizers or delete pods as the first
   troubleshooting action?
30. What should you check before any destructive command?

### Answers

1. An image is packaged content and metadata; a container is a running instance
   created from that image.
2. A pod adds a schedulable unit, metadata, shared networking, volumes, and a
   place for tightly coupled containers.
3. Reconciliation is repeated comparison and correction of observed state
   toward desired state.
4. `spec` is desired configuration; `status` is observed state reported by the
   platform.
5. The API server.
6. The scheduler.
7. The kubelet.
8. Integrated authentication, console, Operators, Projects, Routes,
   ImageStreams, SCCs, builds, platform lifecycle, and other administered
   capabilities.
9. `apiVersion`, `kind`, `metadata`, and normally `spec`.
10. Indentation defines parent-child data structure.
11. A dash starts an item in a sequence.
12. Environment values are strings, while unquoted `3` is parsed as a number.
13. The controller must know which pods it owns and manages.
14. A Deployment replaces failed pods, scales replicas, and manages rollouts.
15. When pods require stable identity, ordered behavior, or a stable storage
   relationship.
16. It controls whether the pod is considered ready for Service traffic; it
   does not simply restart the container.
17. Pod IPs change when pods are replaced; a Service provides stable discovery.
18. Normally through a label selector, with ready backends represented in
   EndpointSlices.
19. A Service.
20. A PV is cluster-scoped capacity; a PVC is a namespaced request for
   capacity.
21. A request guides scheduling/accounting; a limit bounds runtime use.
22. A namespaced API identity for a pod or automation.
23. RBAC controls allowed API actions; SCC controls acceptable pod runtime
   security.
24. Base64 is reversible encoding, not encryption.
25. A controller that automates domain-specific operational lifecycle
   knowledge through APIs.
26. API acceptance proves that desired state was stored, not that controllers,
   scheduling, images, configuration, probes, and the application all
   succeeded.
27. `oc describe pod`, `oc logs`, and `oc logs --previous`, using the exact pod
   and container as needed.
28. Its labels may not match, or its readiness condition may be false.
29. Those actions can hide evidence, orphan resources, or cause a controller to
   recreate the same failure.
30. Current cluster, identity, project, exact target, ownership, and the impact
   of deletion.

## 27. Source and Scope Note

This independently written foundation is aligned with the OpenShift Container
Platform 4.18 Architecture, CLI Tools, Networking, Storage, Images, Builds, and
Authentication and Authorization guides, together with the Kubernetes concept
model for components, objects, workloads, Services, and persistent volumes.

The chapter intentionally teaches concepts in a beginner-first order rather
than copying the organization or prose of vendor documentation. Later handbook
chapters provide deeper procedures for the EX280 objective groups. When a live
cluster differs, use API discovery, `oc explain`, exact status, and official
version-specific behavior as the authority.

You now have the prerequisite foundation for Chapter 01: **Manage OpenShift
Container Platform**.
