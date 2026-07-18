# Chapter 3: Deploy Applications

## Learning Objectives

By the end of this chapter, you will be able to:

- Deploy applications from OpenShift Templates
- Deploy applications using Helm charts
- Manage application deployments including scaling, updates, and rollbacks
- Work with ReplicaSets and understand their role in deployment management
- Work with labels and selectors to organize and target resources
- Configure Services for internal application communication
- Expose applications to external access using Routes
- Understand service discovery and DNS within the cluster
- Diagnose and resolve deployment failures

---

## OpenShift Concepts

### Application Deployment Models

OpenShift supports multiple application deployment models, each suited to different use cases:

- **YAML Manifests**: Direct resource definitions applied with `oc apply`. Maximum control, no abstraction layer.
- **Templates**: Server-side parameterized blueprints processed by the OpenShift Template Service. Ideal for standardized application patterns with configurable parameters.
- **Helm Charts**: Package-based deployments using the Helm client. Industry-standard for complex applications with many interdependent resources.
- **Operators**: Automated application lifecycle management using custom controllers. Best for stateful applications requiring specialized operations.

### OpenShift Templates

Templates are OpenShift-specific resources that define parameterized application blueprints. A Template contains:

- **Objects**: One or more Kubernetes resources (Deployments, Services, Routes, ConfigMaps, etc.)
- **Parameters**: Configurable variables with default values, validation, and optional masking
- **Objects with expression substitution**: Resource fields use `${PARAMETER_NAME}` syntax for dynamic values

Templates are processed server-side by the Template Service. When you instantiate a Template, the server substitutes parameter values into the object definitions, then creates the resulting resources.

Templates live in namespaces and can be discovered, shared, and reused across projects. OpenShift ships with built-in templates for common applications (WordPress, Django, Node.js, etc.).

### Helm

Helm is the package manager for Kubernetes. A Helm chart is a collection of pre-configured manifests packaged together with:

- **Chart.yaml**: Metadata about the chart (name, version, maintainer)
- **values.yaml**: Default configuration values
- **templates/**: Go-template-based manifest files
- **Chart lifecycle hooks**: Init jobs, pre/post-install hooks, test hooks

Helm renders templates client-side using values, then submits the resulting manifests to the API server. Helm tracks releases, enabling versioned upgrades and rollbacks.

### ReplicaSets

A ReplicaSet is a Kubernetes resource that ensures a specified number of pod replicas are running at all times. It is the intermediate layer between Deployments and Pods:

```
Deployment → creates → ReplicaSet → manages → Pods
```

When you create a Deployment, OpenShift automatically creates a ReplicaSet. Each time you update a Deployment (e.g., change the image), a new ReplicaSet is created, and the old one is scaled down. This enables rolling updates and rollbacks.

You rarely create ReplicaSets directly. They are managed by Deployments, but understanding them is essential for troubleshooting.

### Labels and Selectors

Labels are key-value pairs attached to resources for organization and selection. Selectors are expressions used to filter resources by their labels. This label-selector model is the fundamental targeting mechanism in Kubernetes and OpenShift.

- **Equality-based selector**: `app=nginx` or `app != nginx`
- **Set-based selector**: `env in (dev, staging)` or `tier notin (frontend, backend)`

Deployments use selectors to identify which pods they manage. Services use selectors to identify which pods receive traffic.

### Services

A Service provides stable network access to a dynamically changing set of pods. Since pods are ephemeral and get new IP addresses on each restart, Services provide:

- A stable ClusterIP (virtual IP)
- DNS-based service discovery
- Load balancing across backend pods
- Health checking via endpoint slicing

Service types in OpenShift:

| Type | Description |
|------|-------------|
| `ClusterIP` | Internal-only access (default) |
| `NodePort` | Exposes service on a static port on each node |
| `LoadBalancer` | Requests an external load balancer (cloud providers) |
| `ExternalName` | Maps service to a CNAME record |

### Routes

Routes are OpenShift's Layer 7 load balancer, built on HAProxy. A Route exposes a Service externally by:

- Assigning a DNS hostname
- Terminating TLS (edge, passthrough, or re-encryption)
- Routing based on host, path, or header matching
- Supporting wildcard and subdomain routing

Routes are the primary mechanism for exposing HTTP/HTTPS applications in OpenShift.

### Service Discovery and DNS

OpenShift includes an internal DNS service (based on CoreDNS). Every Service automatically gets a DNS record:

```
<service-name>.<namespace>.svc.cluster.local
```

Short forms work within the same namespace:

```
<service-name>
```

Cross-namespace access requires the fully qualified name or the `<service-name>.<namespace>` shorthand.

---

## Architecture and Internal Operation

### Deployment Update Flow

When you update a Deployment, the following sequence occurs:

```
1. User updates Deployment spec (e.g., new image)
        |
        v
2. Deployment controller detects pod template hash change
        |
        v
3. New ReplicaSet created with updated template
        |
        v
4. New ReplicaSet begins scaling up (maxSurge pods)
        |
        v
5. New pods pass readiness probes
        |
        v
6. Old ReplicaSet begins scaling down (maxUnavailable pods)
        |
        v
7. Old ReplicaSet reaches 0 replicas
        |
        v
8. Rollout complete
```

The `maxSurge` and `maxUnavailable` parameters control the pace and availability during this transition.

### Service Traffic Flow

```
Client → Service ClusterIP → kube-proxy → iptables/IPVS rules → Pod IP
```

kube-proxy runs on every node and programs network rules to route traffic from the Service IP to healthy pod IPs. OpenShift uses IPVS mode by default for better load distribution at scale.

### Route Traffic Flow

```
External Client → Router (HAProxy) → TLS Termination → Service → Pod
```

The Router runs as a Deployment in the `openshift-ingress` namespace. Each node runs a host-networked router pod that accepts external traffic on ports 80 and 443.

### Template Processing Flow

```
1. User requests Template instantiation
        |
        v
2. Template Service validates parameters
        |
        v
3. Parameter defaults applied where user did not provide values
        |
        v
4. Parameter validation rules evaluated (required, from, options, etc.)
        |
        v
5. Expression substitution: ${PARAM_NAME} replaced in object templates
        |
        v
6. Processed objects submitted to API server for creation
```

### DNS Resolution Flow

```
Pod → node local DNS cache → CoreDNS (openshift-dns namespace)
                              |
                              ├── Service records (from API server watch)
                              ├── Pod records (if enabled)
                              └── Upstream forwarding (external DNS)
```

---

## Components and Terminology

| Term | Description |
|------|-------------|
| Template | OpenShift server-side parameterized application blueprint |
| Helm Chart | Package of pre-configured Kubernetes manifests |
| Helm Release | A single deployment of a chart instance |
| ReplicaSet | Ensures a specified number of pod replicas are running |
| Label | Key-value pair attached to resources for organization |
| Selector | Expression used to filter resources by labels |
| Service | Stable network endpoint for a set of pods |
| Route | OpenShift Layer 7 load balancer for external access |
| ClusterIP | Internal virtual IP assigned to a Service |
| NodePort | Static port opened on each node for external access |
| Readiness Probe | Determines if a pod should receive traffic |
| Liveness Probe | Determines if a pod should be restarted |
| Rolling Update | Gradual pod replacement strategy |
| HAProxy Router | OpenShift's ingress controller implementing Routes |
| CoreDNS | Cluster-internal DNS service |
| Endpoint | Set of pod IPs backing a Service |
| EndpointSlice | Scalable replacement for Endpoints resources |

---

## Administration Tasks

### Deploying from OpenShift Templates

#### Listing Available Templates

```bash
# Templates in current project
oc get templates

# Templates across all namespaces
oc get templates -A

# Built-in templates in openshift namespace
oc get templates -n openshift
```

#### Examining a Template

```bash
oc describe template <template-name> -n <namespace>
oc get template <template-name> -n <namespace> -o yaml
```

#### Instantiating a Template

```bash
# Process and view the generated objects without creating them
oc process <template-name> -n <namespace>

# Process with custom parameter values
oc process <template-name> -n <namespace> \
  -p APPLICATION_NAME=myapp \
  -p NAMESPACE=myproject

# Process and apply in one step
oc process <template-name> -n <namespace> | oc apply -f -

# Create directly from template
oc new-app <template-name> -n <namespace>
```

#### Creating a Custom Template

```bash
oc apply -f custom-template.yaml
```

### Deploying from Helm Charts

#### Adding a Helm Repository

```bash
helm repo add <repo-name> <repo-url>
helm repo update
```

#### Searching for Charts

```bash
helm search repo <keyword>
```

#### Installing a Chart

```bash
# Install with default values
helm install <release-name> <repo-name>/<chart-name> -n <namespace>

# Install with custom values
helm install <release-name> <repo-name>/<chart-name> -n <namespace> \
  --set image.tag=1.25 \
  --set replicaCount=3

# Install with values file
helm install <release-name> <repo-name>/<chart-name> -n <namespace> \
  -f custom-values.yaml

# Install and let Helm create the namespace
helm install <release-name> <repo-name>/<chart-name> --create-namespace -n <namespace>
```

#### Managing Helm Releases

```bash
# List releases
helm list -n <namespace>
helm list -A

# Upgrade a release
helm upgrade <release-name> <repo-name>/<chart-name> -n <namespace> \
  --set image.tag=1.26

# Rollback a release
helm rollback <release-name> <revision-number> -n <namespace>

# Uninstall a release
helm uninstall <release-name> -n <namespace>

# View release history
helm history <release-name> -n <namespace>

# View rendered manifests
helm get manifest <release-name> -n <namespace>

# View values used
helm get values <release-name> -n <namespace>
```

### Managing Application Deployments

#### Creating a Deployment

```bash
oc apply -f deployment.yaml
```

#### Scaling a Deployment

```bash
oc scale deployment/<name> --replicas=5
```

#### Updating a Deployment Image

```bash
oc set image deployment/<name> <container>=<new-image>:<tag>
```

#### Checking Deployment Status

```bash
oc get deployment/<name>
oc rollout status deployment/<name>
oc rollout history deployment/<name>
```

#### Rolling Back a Deployment

```bash
oc rollout undo deployment/<name>
oc rollout undo deployment/<name> --to-revision=2
```

#### Pausing and Resuming Rollouts

```bash
oc rollout pause deployment/<name>
# Make multiple changes...
oc rollout resume deployment/<name>
```

### Working with ReplicaSets

#### Listing ReplicaSets

```bash
oc get replicasets
oc get rs
```

#### Examining a ReplicaSet

```bash
oc describe replicaset <rs-name>
oc get replicaset <rs-name> -o yaml
```

#### Understanding ReplicaSet Relationship to Deployments

```bash
# See which Deployment owns a ReplicaSet
oc get rs <rs-name> -o jsonpath='{.ownerReferences[0].name}'

# See all ReplicaSets for a Deployment
oc get rs -l app=<app-label>
```

#### Manually Scaling a ReplicaSet (Not Recommended in Production)

```bash
oc scale replicaset/<rs-name> --replicas=3
```

Note: Scaling a ReplicaSet directly bypasses the Deployment controller. If the Deployment exists, it will reconcile the ReplicaSet back to its desired replica count.

### Working with Labels and Selectors

#### Adding Labels to Resources

```bash
oc label deployment/<name> environment=production --overwrite
oc label pod/<name> tier=frontend --overwrite
```

#### Removing Labels

```bash
oc label deployment/<name> environment-
```

The trailing `-` removes the label.

#### Selecting Resources by Label

```bash
# Equality-based
oc get pods -l app=myapp
oc get pods -l app=myapp,environment=production

# Set-based
oc get pods -l 'environment in (production,staging)'
oc get pods -l 'tier notin (database,cache)'
```

#### Using Labels in Deployments

The `spec.selector.matchLabels` field in a Deployment must match `spec.template.metadata.labels`. This binding cannot be changed after creation.

### Configuring Services

#### Creating a ClusterIP Service

```bash
oc apply -f service.yaml
```

Or imperatively:

```bash
oc expose deployment/<name> \
  --name=<service-name> \
  --port=80 \
  --target-port=8080 \
  --type=ClusterIP
```

#### Creating a NodePort Service

```bash
oc expose deployment/<name> \
  --name=<service-name> \
  --port=80 \
  --target-port=8080 \
  --type=NodePort
```

The assigned NodePort is in the range 30000-32767 (configurable).

#### Examining Services

```bash
oc get services
oc get svc
oc describe svc <service-name>
oc get endpoints <service-name>
oc get endpointslices
```

#### Service DNS Resolution

```bash
# Test DNS resolution from a pod
oc run dns-test --image=registry.redhat.io/ubi9/kubernetes-client:v1.31 \
  --restart=Never -i --tty --command -- \
  nslookup <service-name>.<namespace>.svc.cluster.local
```

### Exposing Applications via Routes

#### Creating a Route

```bash
oc apply -f route.yaml
```

Or imperatively:

```bash
oc expose svc/<service-name>
```

This creates a Route pointing to the specified Service with an auto-generated hostname.

#### Creating a Route with Custom Host

```bash
oc expose svc/<service-name> \
  --hostname=myapp.example.com
```

#### Creating a Route with TLS

```bash
oc expose svc/<service-name> \
  --hostname=myapp.example.com \
  --tls-termination=edge
```

#### Examining Routes

```bash
oc get routes
oc get route
oc describe route <route-name>
```

#### Route TLS Termination Types

```bash
# Edge termination (default for HTTPS) - TLS terminated at router
oc expose svc/<service-name> --tls-termination=edge

# Passthrough - TLS passed to backend pod unchanged
oc expose svc/<service-name> --tls-termination=passthrough

# Re-encryption - TLS terminated at router, re-encrypted to backend
oc expose svc/<service-name> --tls-termination=reencrypt
```

---

## YAML Examples

### OpenShift Template

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: myapp-template
  namespace: myapp-project
  annotations:
    description: "Deploy a sample nginx application"
    iconClass: "icon-nginx"
    tags: "nginx,web,example"
objects:
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ${APPLICATION_NAME}
    spec:
      replicas: ${REPLICAS}
      selector:
        matchLabels:
          app: ${APPLICATION_NAME}
      template:
        metadata:
          labels:
            app: ${APPLICATION_NAME}
        spec:
          containers:
            - name: nginx
              image: "${IMAGE_NAME}:${IMAGE_TAG}"
              ports:
                - containerPort: 8080
              resources:
                requests:
                  memory: "${MEMORY_REQUESTS}"
                  cpu: "${CPU_REQUESTS}"
                limits:
                  memory: "${MEMORY_LIMITS}"
                  cpu: "${CPU_LIMITS}"
  - apiVersion: v1
    kind: Service
    metadata:
      name: ${APPLICATION_NAME}
    spec:
      selector:
        app: ${APPLICATION_NAME}
      ports:
        - port: 80
          targetPort: 8080
  - apiVersion: route.openshift.io/v1
    kind: Route
    metadata:
      name: ${APPLICATION_NAME}
    spec:
      to:
        kind: Service
        name: ${APPLICATION_NAME}
      tls:
        termination: edge
parameters:
  - name: APPLICATION_NAME
    description: "Application name"
    required: true
    value: myapp
  - name: REPLICAS
    description: "Number of replicas"
    required: true
    value: "2"
  - name: IMAGE_NAME
    description: "Container image name"
    required: true
    value: registry.redhat.io/ubi9/nginx-124
  - name: IMAGE_TAG
    description: "Container image tag"
    required: true
    value: "latest"
  - name: MEMORY_REQUESTS
    description: "Memory requests"
    value: "128Mi"
  - name: MEMORY_LIMITS
    description: "Memory limits"
    value: "256Mi"
  - name: CPU_REQUESTS
    description: "CPU requests"
    value: "100m"
  - name: CPU_LIMITS
    description: "CPU limits"
    value: "200m"
```

### Deployment with Readiness and Liveness Probes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp-project
  labels:
    app: myapp
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
        - name: nginx
          image: registry.redhat.io/ubi9/nginx-124:latest
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
```

### ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp-project
  labels:
    app: myapp
spec:
  selector:
    app: myapp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

### NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
  namespace: myapp-project
spec:
  selector:
    app: myapp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080
  type: NodePort
```

### Route with Edge TLS

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp-route
  namespace: myapp-project
spec:
  to:
    kind: Service
    name: myapp-service
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None
```

### Route with Custom Host and Backend TLS

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp-secure-route
  namespace: myapp-project
spec:
  host: myapp.example.com
  to:
    kind: Service
    name: myapp-service
  port:
    targetPort: http
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: None
  tls:
    certificate: |
      -----BEGIN CERTIFICATE-----
      MIID...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN PRIVATE KEY-----
      MIIE...
      -----END PRIVATE KEY-----
    caCertificate: |
      -----BEGIN CERTIFICATE-----
      MIID...
      -----END CERTIFICATE-----
```

### Helm Values Override File

```yaml
replicaCount: 3

image:
  repository: registry.redhat.io/ubi9/nginx-124
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"

nodeSelector: {}

tolerations: []

affinity: {}
```

---

## Explanation of YAML

### Template Structure

The Template resource has two critical sections:

- **`objects`**: An array of Kubernetes resource definitions. Each object can use `${PARAMETER_NAME}` expressions that are substituted during processing. The objects are created in the order they appear.
- **`parameters`**: Defines configurable variables. Each parameter can have `name`, `description`, `required` (boolean), `value` (default), `from` (regex to auto-generate values), `generate` (expression type), and `options` (dropdown values).

The `annotations` field provides metadata for the web console display, including icons and search tags.

### Deployment with Probes

- **Readiness Probe**: Determines whether the pod should receive traffic. If the probe fails, the pod is removed from Service endpoints but is not restarted. The `initialDelaySeconds` gives the application time to start before probing begins.
- **Liveness Probe**: Determines whether the container should be restarted. If the probe fails repeatedly (`failureThreshold` times), the kubelet kills and restarts the container.
- **`maxSurge: 1`**: Allows one extra pod above the desired replica count during updates.
- **`maxUnavailable: 0`**: Ensures no pods are taken down before replacements are ready, guaranteeing zero downtime.

### Route with Edge TLS

- **`termination: edge`**: TLS is terminated at the HAProxy router. The router decrypts traffic and forwards plaintext to the backend Service. This is the most common configuration.
- **`insecureEdgeTerminationPolicy: Redirect`**: HTTP requests are automatically redirected to HTTPS with a 301 response.
- **`weight: 100`**: Used for traffic splitting. A weight of 100 means 100% of traffic goes to this backend.
- **`wildcardPolicy: None`**: Disables wildcard routing. Set to `Subdomain` to allow `*.example.com` routing.

---

## Verification Procedures

### Verify Template Instantiation

```bash
# Check template exists
oc get template myapp-template

# Process template and review output
oc process myapp-template -p APPLICATION_NAME=testapp

# Verify created resources
oc get all -l app=myapp
```

Expected: All objects defined in the template are created with correct parameter values.

### Verify Helm Deployment

```bash
helm list -n myapp-project
helm get manifest <release-name> -n myapp-project
oc get all -l app.kubernetes.io/instance=<release-name>
```

Expected: Release appears in Helm list, manifests are rendered correctly, and pods are running.

### Verify ReplicaSet Health

```bash
oc get rs
oc get rs -o custom-columns=NAME:.metadata.name,DESIRED:.spec.replicas,CURRENT:.status.replicas,READY:.status.readyReplicas
```

Expected: Active ReplicaSet shows `DESIRED=CURRENT=READY`. Old ReplicaSets show `DESIRED=0`.

### Verify Service Endpoints

```bash
oc get svc myapp-service
oc get endpoints myapp-service
```

Expected: Service has a ClusterIP, and endpoints lists pod IPs matching running pods.

### Verify Route Accessibility

```bash
oc get route myapp-route
ROUTE_HOST=$(oc get route myapp-route -o jsonpath='{.spec.host}')
curl -k https://${ROUTE_HOST}
```

Expected: Route has a valid host, and curl returns the application response.

### Verify DNS Resolution

```bash
oc run dns-test --image=registry.redhat.io/ubi9/kubernetes-client:v1.31 \
  --restart=Never -i --tty --rm --command -- \
  nslookup myapp-service.myapp-project.svc.cluster.local
```

Expected: DNS resolves to the Service ClusterIP.

### Verify Labels and Selectors

```bash
oc get pods -l app=myapp --show-labels
oc get deployment myapp -o jsonpath='{.spec.selector.matchLabels}'
```

Expected: Pod labels match the Deployment selector.

---

## Troubleshooting

### Pods Stuck in Pending After Deployment

**Cause**: Insufficient node resources, taints without tolerations, or node affinity conflicts.

**Diagnosis**:

```bash
oc describe pod <pod-name>
oc get events -n <namespace> --sort-by=.lastTimestamp | tail -20
oc adm top nodes
```

Look for `FailedScheduling` events. The scheduler explains why in the event message.

**Resolution**:
- Verify cluster has available resources
- Check node taints: `oc describe node <node-name> | grep -A 5 Taints`
- Verify resource requests do not exceed node capacity
- Check pod priority and preemption settings

### Deployment Fails to Roll Out

**Cause**: New image fails health checks, SCC denial, or resource constraints prevent new pods from starting.

**Diagnosis**:

```bash
oc rollout status deployment/<name> --timeout=300s
oc describe deployment/<name>
oc get pods -l app=<app-label>
oc logs <new-pod-name>
```

**Resolution**:
- Check new pod events for startup failures
- Verify readiness/liveness probe endpoints are correct
- Check if the new image exists and is pullable
- If stuck, pause and rollback: `oc rollout undo deployment/<name>`

### Service Has No Endpoints

**Cause**: Label selector mismatch, no running pods, or pods failing readiness checks.

**Diagnosis**:

```bash
oc get endpoints <service-name>
oc get svc <service-name> -o yaml | grep -A 5 selector
oc get pods --show-labels
```

**Resolution**:
- Verify Service `spec.selector` labels match pod labels exactly
- Ensure pods are in Running state and passing readiness probes
- Check for typos in label keys or values

### Route Returns 503 Service Unavailable

**Cause**: Backend Service has no ready endpoints, or route-to-service binding is broken.

**Diagnosis**:

```bash
oc describe route <route-name>
oc get endpoints <service-name>
oc get pods -l app=<app-label>
```

**Resolution**:
- Verify the Route points to the correct Service name
- Ensure the Service has ready endpoints
- Check router pod logs: `oc logs -n openshift-ingress deploy/router-default`

### Helm Install Fails

**Cause**: Invalid values, resource conflicts, or permission issues.

**Diagnosis**:

```bash
helm install <name> <chart> -n <namespace> --debug
helm history <name> -n <namespace>
```

**Resolution**:
- Check rendered manifests: `helm template <name> <chart> -n <namespace>`
- Verify namespace exists and user has permissions
- Check for name conflicts with existing resources
- Review chart compatibility with cluster version

### ReplicaSet Stuck Scaling

**Cause**: Pod template errors, insufficient resources, or SCC denial.

**Diagnosis**:

```bash
oc describe replicaset <rs-name>
oc get pods -l <rs-selector-labels>
```

**Resolution**:
- Check pod creation events in the ReplicaSet description
- Verify the pod template is valid
- Check node resource availability

### DNS Resolution Fails Inside Cluster

**Cause**: CoreDNS pods not running, incorrect DNS name format, or network policy blocking DNS.

**Diagnosis**:

```bash
oc get pods -n openshift-dns
oc get configmap -n openshift-dns
oc run dns-debug --image=registry.redhat.io/ubi9/kubernetes-client:v1.31 \
  --restart=Never -i --tty --rm --command -- \
  cat /etc/resolv.conf
```

**Resolution**:
- Ensure CoreDNS pods are running
- Verify DNS name format: `<service>.<namespace>.svc.cluster.local`
- Check network policies are not blocking port 53 UDP/TCP
- Verify the `defaultDNSDomain` cluster network configuration

### Template Parameter Validation Fails

**Cause**: Required parameter missing, value does not match `from` regex, or value not in `options` list.

**Diagnosis**:

```bash
oc process <template-name> -p PARAM=value 2>&1
oc describe template <template-name> | grep -A 20 parameters
```

**Resolution**:
- Provide all required parameters
- Match the `from` regex pattern if specified
- Use values from the `options` list if defined
- Check parameter names are case-exact

---

## Production Best Practices

### Deployment Strategies

- Use rolling updates with `maxUnavailable: 0` for zero-downtime deployments
- Configure readiness probes on all production workloads
- Set appropriate `initialDelaySeconds` to match application startup time
- Use `terminationGracePeriodSeconds` to allow graceful shutdown
- Test deployment updates in staging before production

### Service Configuration

- Use `ClusterIP` as the default Service type; only expose externally via Routes
- Name Service ports for clarity (`name: http`, `name: https`)
- Use named ports in Routes (`targetPort: http`) instead of numeric ports
- Avoid `NodePort` unless specifically required; Routes provide better TLS and routing control
- Implement endpoint slicing for large-scale deployments (enabled by default in OCP 4.18)

### Route Best Practices

- Always enable TLS with `termination: edge` for external-facing applications
- Set `insecureEdgeTerminationPolicy: Redirect` to enforce HTTPS
- Use custom hostnames for production; avoid auto-generated routes
- Implement backend pod authentication for re-encryption routes
- Monitor router pod resource usage and scale if needed

### Label Conventions

- Use consistent label keys across the organization
- Recommended label keys: `app`, `version`, `environment`, `tier`, `team`
- Avoid overly specific labels that limit future flexibility
- Use label selectors sparingly; broad selectors create larger update blast radius
- Never modify `spec.selector` on an existing Deployment (it is immutable)

### Helm Best Practices

- Pin chart versions in production deployments
- Store custom values files in version control
- Use `helm diff` plugin before upgrades to preview changes
- Set resource limits on all Helm-deployed workloads
- Regularly update charts to receive security patches
- Use Helm secrets plugins for sensitive values in production

### Template Best Practices

- Use templates for standardized application patterns
- Set sensible defaults for all parameters
- Mark critical parameters as `required: true`
- Use `from` regex for auto-generating unique values (passwords, names)
- Document templates with clear `description` and `annotations`
- Store custom templates in a dedicated namespace

### ReplicaSet Management

- Let Deployments manage ReplicaSets; avoid manual ReplicaSet manipulation
- Monitor ReplicaSet count — more than 2 active ReplicaSets per Deployment indicates a stuck rollout
- Old ReplicaSets consume no resources but clutter the namespace; consider cleanup policies
- Use `oc rollout history` to track revision changes

---

## Certification Exam Notes

### Exam-Relevant Commands You Must Know

```bash
# Templates
oc get templates
oc describe template <name>
oc process <template-name> [-p PARAM=value]
oc process <template-name> | oc apply -f -
oc new-app <template-name>

# Helm
helm install <release> <chart> -n <namespace>
helm upgrade <release> <chart> -n <namespace> --set key=value
helm rollback <release> <revision> -n <namespace>
helm list -n <namespace>
helm uninstall <release> -n <namespace>
helm history <release> -n <namespace>

# Deployments
oc get deployment
oc scale deployment/<name> --replicas=N
oc set image deployment/<name> <container>=<image>
oc rollout status deployment/<name>
oc rollout history deployment/<name>
oc rollout undo deployment/<name>
oc rollout pause deployment/<name>
oc rollout resume deployment/<name>

# ReplicaSets
oc get replicasets
oc describe replicaset <name>

# Labels
oc label <kind>/<name> key=value --overwrite
oc label <kind>/<name> key-
oc get <kind> -l <selector>

# Services
oc get services
oc get endpoints <service-name>
oc expose deployment/<name> --port=80 --target-port=8080
oc expose svc/<name>  # creates Route

# Routes
oc get routes
oc describe route <name>
```

### Common Exam Scenarios

1. **"Deploy an application from a template"** → `oc process <template> | oc apply -f -` or `oc new-app <template>`
2. **"Scale a deployment to 3 replicas"** → `oc scale deployment/<name> --replicas=3`
3. **"Expose a service externally"** → `oc expose svc/<service-name>`
4. **"Find which pods a service routes to"** → `oc get endpoints <service-name>`
5. **"Update deployment image and verify rollout"** → `oc set image deployment/<name> <c>=<img>` then `oc rollout status deployment/<name>`
6. **"Add a label to a resource"** → `oc label <kind>/<name> key=value`
7. **"Roll back a deployment"** → `oc rollout undo deployment/<name>`
8. **"Install a Helm chart"** → `helm install <release> <repo/chart> -n <namespace>`
9. **"Check ReplicaSet status"** → `oc get rs` or `oc describe rs <name>`
10. **"Create a Route with TLS"** → `oc expose svc/<name> --tls-termination=edge`

### Important Exam Details

- Routes are OpenShift-specific; Know the difference between `edge`, `passthrough`, and `reencrypt` TLS termination
- The `oc expose` command creates both Services and Routes depending on the input type
- ReplicaSets are created automatically by Deployments; you rarely create them manually
- Labels cannot be changed in `spec.selector` after a Deployment is created
- Helm is available in the exam environment; know basic `helm install`, `upgrade`, `rollback`, and `list`
- Template parameters are case-sensitive
- Service DNS follows the pattern `<service>.<namespace>.svc.cluster.local`

---

## Common Mistakes

### Mistake: Forgetting Readiness Probes

Without readiness probes, Services route traffic to pods immediately on creation, before the application is ready to handle requests. This causes initial request failures during deployments and pod restarts. Always configure readiness probes for production workloads.

### Mistake: Mixing Up `targetPort` and `port` in Services

The `port` field is what clients connect to. The `targetPort` field is where traffic is forwarded on the pod. If these differ, ensure the container actually listens on `targetPort`. A mismatch causes connection refused errors.

### Mistake: Creating Routes Before Services Have Endpoints

Creating a Route pointing to a Service with no ready endpoints results in 503 errors. Always verify `oc get endpoints <service-name>` shows pod IPs before testing the Route.

### Mistake: Modifying Deployment Selector Labels

The `spec.selector.matchLabels` field is immutable after Deployment creation. Attempting to change it results in a validation error. To change labels, create a new Deployment.

### Mistake: Using `oc expose deployment` Without Checking Existing Services

`oc expose deployment/<name>` creates a Service with the same name as the Deployment. If a Service with that name already exists, the command fails. Use `--name=<different-name>` to avoid conflicts.

### Mistake: Helm Release Name vs. Chart Name Confusion

The release name (`helm install myrelease bitnami/nginx`) is independent of the chart name. The release name is used for all subsequent operations (`helm upgrade myrelease`, `helm rollback myrelease`). Mixing them up causes "release not found" errors.

### Mistake: Template Parameter Name Typos

Template parameters are case-sensitive. `APPLICATION_NAME` is different from `application_name`. A typo results in the parameter not being substituted, leaving `${PARAM_NAME}` literally in the resource.

### Mistake: Not Checking Rollout Status After Updates

After updating a Deployment, assuming the change is complete without running `oc rollout status` can lead to testing against old pods. Always verify the rollout completed before proceeding.

---

## Chapter Summary

This chapter covered the complete application deployment workflow in OpenShift. You learned to deploy applications from Templates using parameterized blueprints, from Helm charts using package management, and from YAML manifests. You managed deployment lifecycles including scaling, image updates, rolling updates, and rollbacks. You understood how ReplicaSets sit between Deployments and Pods, managing the actual pod count. You worked with labels and selectors as the fundamental targeting mechanism for organizing and selecting resources. You configured Services for internal communication using ClusterIP and NodePort types, and exposed applications externally using Routes with edge TLS termination. You learned service discovery through CoreDNS, troubleshooting common deployment failures, and production best practices for zero-downtime deployments.

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| List templates | `oc get templates [-A]` |
| Describe template | `oc describe template <name>` |
| Process template | `oc process <name> [-p PARAM=value]` |
| Deploy from template | `oc process <name> \| oc apply -f -` |
| Helm install | `helm install <release> <chart> -n <ns>` |
| Helm upgrade | `helm upgrade <release> <chart> -n <ns> --set k=v` |
| Helm rollback | `helm rollback <release> <revision> -n <ns>` |
| Helm list | `helm list -n <ns>` |
| Helm uninstall | `helm uninstall <release> -n <ns>` |
| Scale deployment | `oc scale deployment/<name> --replicas=N` |
| Set image | `oc set image deployment/<name> <c>=<img>` |
| Rollout status | `oc rollout status deployment/<name>` |
| Rollout history | `oc rollout history deployment/<name>` |
| Rollback deployment | `oc rollout undo deployment/<name>` |
| Pause rollout | `oc rollout pause deployment/<name>` |
| Resume rollout | `oc rollout resume deployment/<name>` |
| List ReplicaSets | `oc get rs` |
| Add label | `oc label <kind>/<name> key=value --overwrite` |
| Remove label | `oc label <kind>/<name> key-` |
| Filter by label | `oc get pods -l key=value` |
| Create Service | `oc expose deployment/<name> --port=80 --target-port=8080` |
| Create Route | `oc expose svc/<name>` |
| Create Route with TLS | `oc expose svc/<name> --tls-termination=edge` |
| Check endpoints | `oc get endpoints <service-name>` |
| List Routes | `oc get routes` |
| Describe Route | `oc describe route <name>` |

### Service Types Comparison

| Type | Internal Access | External Access | Use Case |
|------|----------------|----------------|----------|
| ClusterIP | Yes | No | Backend services, databases |
| NodePort | Yes | Yes (nodeIP:nodePort) | Debugging, direct node access |
| LoadBalancer | Yes | Yes (cloud LB IP) | Cloud environments |
| ExternalName | No | No | Alias to external service |

### Route TLS Termination Types

| Type | TLS at Router | TLS to Backend | Use Case |
|------|-------------|----------------|----------|
| edge | Terminated | Plaintext | Most applications |
| passthrough | None | Preserved | Apps that manage their own TLS |
| reencrypt | Terminated | Re-encrypted | Apps requiring mutual TLS |

### Label Selector Syntax

| Syntax | Example | Meaning |
|--------|---------|---------|
| `=` | `app=nginx` | Exact match |
| `!=` | `app!=nginx` | Not equal |
| `in` | `env in (dev,prod)` | Set membership |
| `notin` | `tier notin (db,cache)` | Set exclusion |
| `exists` | `app` | Key exists |
| `!exists` | `!app` | Key does not exist |

---

## Review Questions

1. What is the difference between an OpenShift Template and a Helm chart?

2. How does a ReplicaSet relate to a Deployment?

3. What happens when you change the container image in a Deployment?

4. What is the difference between a readiness probe and a liveness probe?

5. How do you expose an internal Service to external users?

6. What is the DNS name format for a Service named `myapp` in the namespace `production`?

7. What does `oc expose svc/myapp` do?

8. How do you add the label `environment=production` to a Deployment named `myapp`?

9. What is the difference between edge, passthrough, and reencrypt TLS termination on a Route?

10. How do you check which pods are receiving traffic from a Service?

11. What command rolls back a Helm release to revision 2?

12. Why is `spec.selector.matchLabels` immutable in a Deployment?

13. What does `maxSurge: 1` and `maxUnavailable: 0` mean in a rolling update strategy?

14. How do you preview the resources a Template will create without actually creating them?

15. What is the purpose of the HAProxy router in OpenShift?

---

## Answers

1. OpenShift Templates are server-side resources processed by the Template Service with `${PARAM}` substitution. Helm charts are client-side packages using Go templates, managed by the Helm CLI with versioned releases. Templates are OpenShift-native; Helm is Kubernetes-agnostic and industry-standard.

2. A Deployment creates and manages ReplicaSets. Each Deployment update creates a new ReplicaSet with the updated pod template. The old ReplicaSet is scaled down as the new one scales up. ReplicaSets directly manage the pod count.

3. A new ReplicaSet is created with the updated image. New pods start with the new image. Once new pods pass readiness checks, old pods are terminated. The old ReplicaSet is retained for rollback capability.

4. A readiness probe determines if a pod should receive traffic. Failure removes the pod from Service endpoints but does not restart it. A liveness probe determines if a container should be restarted. Repeated failure causes the kubelet to kill and restart the container.

5. Create a Route pointing to the Service using `oc expose svc/<service-name>` or by applying a Route YAML manifest. The Route assigns a DNS hostname and routes external traffic through the HAProxy router.

6. `myapp.production.svc.cluster.local`

7. It creates a Route resource that exposes the Service externally through the HAProxy router, auto-generating a hostname based on the Service name and project.

8. `oc label deployment/myapp environment=production --overwrite`

9. **Edge**: TLS terminated at the router; plaintext forwarded to backend. **Passthrough**: TLS traffic passed unchanged to the backend pod; the router does not decrypt. **Reencrypt**: TLS terminated at the router, then re-encrypted with a different certificate before forwarding to the backend.

10. `oc get endpoints <service-name>` shows the pod IPs that the Service is currently routing traffic to.

11. `helm rollback <release-name> 2 -n <namespace>`

12. The selector identifies which pods belong to the Deployment. Changing it would break the relationship between the Deployment and its managed pods, potentially causing the Deployment to adopt unrelated pods or orphan existing ones.

13. `maxSurge: 1` allows creating 1 extra pod above the desired replica count during updates. `maxUnavailable: 0` ensures no pods are terminated before replacements are ready, guaranteeing zero downtime during the rolling update.

14. `oc process <template-name> [-p PARAM=value]` renders the template and outputs the resulting resources to stdout without creating them.

15. The HAProxy router is OpenShift's ingress controller that implements Routes. It runs on every node in host network mode, accepts external HTTP/HTTPS traffic on ports 80 and 443, terminates TLS, and forwards traffic to the appropriate backend Service based on Route configuration.
