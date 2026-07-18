# Chapter 5: Configure Network Security

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure networking components including the CNI plugin and ingress controller
- Troubleshoot Software Defined Networking (SDN) issues in the cluster
- Create and edit external Routes for custom hostname and TLS requirements
- Control cluster network ingress using NetworkPolicies
- Secure external traffic using TLS certificates with Routes
- Secure internal traffic using TLS and mutual authentication
- Configure application-level NetworkPolicies for pod-to-pod communication
- Diagnose SDN and ingress controller failures
- Apply network security best practices in production environments

---

## OpenShift Concepts

### Container Network Interface (CNI)

A Container Network Interface (CNI) plugin provides networking for containers in Kubernetes and OpenShift. OpenShift 4.18 uses **OVNKubernetes** by default, which provides:

- **Pod-to-pod networking**: Encapsulated overlay network allowing pods on different nodes to communicate directly
- **Network isolation**: Separate networks for different workloads or namespaces
- **Network policies**: Security policies that control pod-to-pod traffic
- **Service networking**: Stable ClusterIP endpoints for Services
- **Multi-tenancy**: Support for multiple network attachments

OVS (Open vSwitch) is the underlying virtual switch that programs flow rules on each node.

### Ingress Controller

The Ingress Controller is a Layer 7 (application layer) load balancer that handles:

- **HTTP/HTTPS routing**: Based on host, path, or header rules
- **TLS termination**: Decrypting incoming HTTPS traffic
- **Host-based routing**: Directing traffic to different Services based on hostname
- **Path-based routing**: Segmenting traffic by URL path
- **Header-based routing**: Routing based on HTTP headers
- **Wildcard support**: Matching multiple hostnames with wildcards

OpenShift uses HAProxy as its ingress controller, deployed as a Deployment in the `openshift-ingress` namespace.

### Software Defined Networking (SDN)

OpenShift's SDN uses OVN (Open Virtual Network) to provide network services across nodes:

- **Logical network abstraction**: Networks exist as logical entities independent of physical topology
- **Encapsulation**: Pod traffic is encapsulated with VXLAN headers for node-to-node transport
- **Gateway services**: Each node runs a gateway that handles pod subnet routing
- **Controller**: The OVN controller manages network state and flow programming

### NetworkPolicies

NetworkPolicies are Kubernetes resources that define traffic rules for pods:

- **Default deny**: No traffic allowed unless explicitly permitted
- **Ingress rules**: Control incoming traffic to pods
- **Egress rules**: Control outgoing traffic from pods
- **Pod selectors**: Target specific pods by label
- **Namespace selectors**: Target pods in other namespaces

NetworkPolicies only apply to namespaces that have at least one NetworkPolicy defined.

### TLS in OpenShift

OpenShift supports multiple TLS configurations:

- **Edge termination**: TLS terminated at the ingress router; traffic to backend is plaintext
- **Passthrough**: TLS passed through to backend unchanged; backend handles TLS
- **Re-encryption**: TLS terminated at router, re-encrypted for backend
- **mTLS (Mutual TLS)**: Both client and server present certificates for authentication

---

## Architecture and Internal Operation

### OVNKubernetes Network Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                      OpenShift Cluster                        │
├───────────────────────────────────────────────────────────────┤
│  Node 1                    Node 2                    Node 3   │
│  ┌─────────┐              ┌─────────┐              ┌───────┐  │
│  │  OVS    │              │  OVS    │              │  OVS  │  │
│  │  Host   │              │  Host   │              │  Host │  │
│  └────┬────┘              └────┬────┘              └────┬──┘  │
│       │                        │                        │     │
│  Pod  │                        │                        Pod   │
│  Subnet    Pod  Subnet    Pod  Subnet    Pod  Subnet Pod      │
│  172.18.0.0/24   172.18.1.0/24   172.18.2.0/24   172.18.3.0/24│
└───────┼────────────────────────┼────────────────────────┼─────┘
        │                        │                        │
        └────────────────────────┼────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │  OVN Controller         │
                    │  (openshift-plane)      │
                    └─────────────────────────┘
```

Each node runs OVS with multiple bridges:

- **br-int**: Integration bridge for pod traffic
- **br-ex**: External bridge for node connectivity
- **tunl0**: Tunnel interface for pod routing

### Ingress Router Architecture

```
External Traffic → Router Pod (Host Network)
        │
        ├── Port 80 (HTTP)
        │       │
        │       └── HAProxy Router
        │               │
        │               ├── Host-based routing
        │               ├── Path-based routing
        │               └── Header-based routing
        │
        └── Port 443 (HTTPS)
                │
                ├── TLS Termination
                │   ├── Edge: Decrypt, forward plaintext
                │   ├── Passthrough: Forward encrypted
                │   └── Re-encrypt: Decrypt, re-encrypt
                │
                └── Backend Services
```

Each router pod runs on every node in host network mode, listening on ports 80 and 443.

### NetworkPolicy Enforcement Flow

```
Pod A (Source) → egress rules → Pod B (Destination)

1. OVS receives packet on Pod A's interface
2. OVS checks egress NetworkPolicy rules
3. If denied, drop packet
4. If allowed, route to Pod B's subnet
5. OVS checks ingress NetworkPolicy rules on Pod B
6. If denied, drop packet
7. If allowed, deliver to Pod B
```

NetworkPolicies are enforced by OVS flow rules programmed when the policy is created or updated.

### TLS Termination Flow

```
Client → HTTPS Request (TLS Encrypted)
        │
        ├── Router (HAProxy)
        │   │
        │   ├── Edge Termination
        │   │   ├── Decrypt with router certificate
        │   │   └── Forward plaintext to backend
        │   │
        │   └── Passthrough
        │       └── Forward encrypted to backend
        │
        └── Backend Service
```

---

## Components and Terminology

| Term | Description |
|------|-------------|
| OVNKubernetes | Default CNI plugin providing SDN |
| OVS | Open vSwitch, underlying virtual switch |
| br-int | Integration bridge for pod traffic |
| br-ex | External bridge for node connectivity |
| VXLAN | Encapsulation protocol for pod traffic |
| Ingress Controller | HAProxy-based Layer 7 load balancer |
| Router | HAProxy instance on each node |
| Route | OpenShift resource for external access |
| NetworkPolicy | Kubernetes resource for pod-to-pod traffic control |
| NetworkAttachmentDefinition | Custom network for pods |
| TLS Edge Termination | TLS terminated at router |
| TLS Passthrough | TLS passed to backend unchanged |
| TLS Re-encryption | TLS terminated and re-encrypted |
| mTLS | Mutual TLS with client certificates |
| ClusterNetwork | Defines IP pool and network overlay |
| MachineNetwork | Defines host IP network |
| IngressController | Operator managing router deployment |
| HAProxy | Load balancer software |
| Flow Rules | OVS rules for network enforcement |
| EndpointSlices | Modern endpoint representation |

---

## Administration Tasks

### Configuring Networking Components

#### Checking CNI Plugin Status

```bash
# Check OVNKubernetes operator
oc get clusteroperators | grep ovs

# Check router pods
oc get pods -n openshift-ingress
oc get pods -n openshift-sdn

# Check node network status
oc get nodes -o custom-columns=NODE:.metadata.name,OVN:.status.conditions[?@.type=="OVNReady"].status
```

Expected: All clusteroperators show `AVAILABLE=True`, and router pods are Running.

#### Checking NetworkStatus

```bash
oc get clusteroperator ovs -o yaml | grep -A 20 "conditions:"
```

#### Configuring a Different CNI

```bash
# Edit the cluster network configuration
oc edit clusternetwork
```

Add a new network configuration with the desired CNI plugin.

#### Updating Ingress Controller Configuration

```bash
# Edit ingress operator configuration
oc edit ingresscontroller.operator.cluster
```

Configure TLS certificates and routing behavior.

### Troubleshooting Software Defined Networking

#### Pods Cannot Communicate

**Symptom**: `curl` from one pod to another fails with connection refused or timeout.

**Diagnosis**:

```bash
# Check node network status
oc get nodes -o custom-columns=NODE:.metadata.name,OVN:.status.conditions[?@.type=="OVNReady"].status

# Check OVS status on a node
ssh <node-name> ovs-vsctl show

# Check pod network CIDR
oc get cluster -o yaml | grep -A 5 "clusterNetwork"

# Check pod IP assignment
oc get pods -o wide

# Test connectivity from a pod
oc exec <pod-name> -- nslookup <other-pod-name>.<namespace>.svc.cluster.local
```

**Resolution**:
- If OVNReady is False, check the operator logs
- Verify the cluster network CIDR is valid and not overlapping
- Check for network policies blocking traffic
- Verify pods are in the same network or have routing rules

#### Router Pod Issues

**Symptom**: Router pods in CrashLoopBackOff or not starting.

**Diagnosis**:

```bash
oc get pods -n openshift-ingress
oc logs -n openshift-ingress deploy/router-default
oc describe pod -n openshift-ingress <router-pod-name>
```

**Resolution**:
- Check if the ingress operator is healthy: `oc get clusteroperator ingress`
- Verify TLS certificates are valid and not expired
- Check for certificate permission issues
- Review HAProxy configuration for errors

#### Network Policy Not Working

**Symptom**: Pods can communicate despite NetworkPolicy denying traffic.

**Diagnosis**:

```bash
# Check NetworkPolicies exist
oc get networkpolicies

# Check OVS flow rules
ssh <node-name> ovs-appctl flow/list

# Check pod network namespace
oc exec <pod-name> -- ip link

# Test connectivity
oc exec <source-pod> -- ping <destination-pod-ip>
```

**Resolution**:
- Verify the NetworkPolicy has selectors matching the target pods
- Check that the namespace has at least one NetworkPolicy (policies don't apply to namespaces without any)
- Verify OVS flow rules are installed
- Check for conflicting policies

### Creating and Editing External Routes

#### Creating a Basic Route

```bash
oc expose svc/<service-name>
oc expose svc/<service-name> --hostname=myapp.example.com
```

#### Creating a Route with Custom Host and TLS

```bash
oc apply -f route.yaml
```

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

#### Editing an Existing Route

```bash
oc edit route <route-name>
```

Or patch specific fields:

```bash
oc patch route myapp-route -p '{"spec":{"tls":{"termination":"edge"}}}'
```

#### Creating a Route with Custom Certificate

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp-route
spec:
  to:
    kind: Service
    name: myapp-service
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: None
    destinationCACertificate: |
      -----BEGIN CERTIFICATE-----
      MIID...
      -----END CERTIFICATE-----
    tlsCertificate: |
      -----BEGIN CERTIFICATE-----
      MIID...
      -----END CERTIFICATE-----
    tlsPrivateKey: |
      -----BEGIN PRIVATE KEY-----
      MIIE...
      -----END PRIVATE KEY-----
```

#### Creating a Route with Host-Based Routing

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: frontend
spec:
  to:
    kind: Service
    name: frontend-service
  host: frontend.example.com
  tls:
    termination: edge
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: backend
spec:
  to:
    kind: Service
    name: backend-service
  host: backend.example.com
  tls:
    termination: edge
```

#### Wildcard Routes

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: wildcard
spec:
  to:
    kind: Service
    name: backend-service
  host: "*.example.com"
  tls:
    termination: edge
  wildcardPolicy: Subdomain
```

#### Path-Based Routing

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: api-route
spec:
  to:
    kind: Service
    name: api-service
  host: myapp.example.com
  tls:
    termination: edge
  port:
    targetPort: api
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: web-route
spec:
  to:
    kind: Service
    name: web-service
  host: myapp.example.com
  tls:
    termination: edge
  port:
    targetPort: web
```

### Controlling Cluster Network Ingress

#### Creating a NetworkPolicy

```bash
oc apply -f networkpolicy.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: myapp-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific-ingress
  namespace: myapp-project
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
        - podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 8080
```

#### Denying All Traffic (Default Deny)

```bash
oc apply -f deny-all.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: myapp-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

#### Allowing Pod-to-Pod Communication

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database
  namespace: myapp-project
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432
```

#### Allowing Egress Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
  namespace: myapp-project
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
    - to:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9090
```

#### Allowing Cross-Namespace Communication

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: myapp-project
spec:
  podSelector:
    matchLabels:
      app: app-pod
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: monitoring
        - podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090
```

### Securing External Traffic with TLS Certificates

#### Generating Self-Signed Certificates

```bash
# Generate private key
openssl genrsa -out tls.key 2048

# Generate certificate
openssl req -new -x509 -key tls.key -out tls.crt -days 365 \
  -subj "/CN=myapp.example.com/O=MyOrg/C=US"
```

#### Creating Secret with TLS Certificates

```bash
oc create secret tls myapp-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n myapp-project
```

#### Using Secret with Route

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp-route
spec:
  to:
    kind: Service
    name: myapp-service
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: None
    destinationCACertificate: |
      $(oc get secret myapp-tls-secret -o jsonpath='{.data.ca.crt}' | base64 -d)
    tlsCertificate: |
      $(oc get secret myapp-tls-secret -o jsonpath='{.data.tls.crt}' | base64 -d)
    tlsPrivateKey: |
      $(oc get secret myapp-tls-secret -o jsonpath='{.data.tls.key}' | base64 -d)
```

Or simpler with the secret reference:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp-route
spec:
  to:
    kind: Service
    name: myapp-service
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: None
    destinationCACertificate: myapp-tls-secret:ca.crt
    tlsCertificate: myapp-tls-secret:tls.crt
    tlsPrivateKey: myapp-tls-secret:tls.key
```

#### Configuring Passthrough TLS

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp-route
spec:
  to:
    kind: Service
    name: myapp-service
  tls:
    termination: passthrough
    insecureEdgeTerminationPolicy: Allow
```

#### Configuring mTLS (Mutual TLS)

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp-route
spec:
  to:
    kind: Service
    name: myapp-service
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: None
    destinationCACertificate: myapp-tls-secret:ca.crt
```

### Securing Internal Traffic

#### Configuring Service Mesh for mTLS

OpenShift supports service mesh integration for internal mTLS:

```bash
# Install Istio service mesh (example)
oc apply -f istio-operator.yaml
oc apply -f istio-config.yaml
```

#### Configuring Pod Security Context for mTLS

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          image: registry.example.com/myapp:v1
          securityContext:
            enableDynamicServiceAccountToken: true
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            runAsUser: 1000
      serviceAccountName: myapp-sa
```

#### Configuring NetworkPolicy for Internal Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal
  namespace: myapp-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

---

## YAML Examples

### OVNKubernetes Network Configuration

```yaml
apiVersion: config.openshift.io/v1
kind: ClusterNetwork
metadata:
  name: cluster
networks:
- cidr: 172.18.0.0/14
hostPrefix: 23
serviceNetwork:
- 10.128.0.0/14
- 10.132.0.0/16
machineCIDR: 10.0.0.0/16
```

### IngressController Configuration

```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress
spec:
  domain: cluster.example.com
  wildcardPolicy: Subdomain
  termination: edge
  insecureEdgeTerminationPolicy: Redirect
  router:
    replicas: 2
  defaultCertificate:
    name: default-tls
  httpRoutePolicy: Standard
```

### NetworkPolicy - Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: myapp-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: myapp-project
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### NetworkPolicy - Allow Specific Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: myapp-project
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432
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

### Route with Re-encryption TLS

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp-secure-route
  namespace: myapp-project
spec:
  to:
    kind: Service
    name: myapp-service
  port:
    targetPort: https
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: None
    destinationCACertificate: myapp-tls-secret:ca.crt
    tlsCertificate: myapp-tls-secret:tls.crt
    tlsPrivateKey: myapp-tls-secret:tls.key
```

### Route with Host-Based Routing

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: frontend-route
  namespace: myapp-project
spec:
  host: frontend.example.com
  to:
    kind: Service
    name: frontend-service
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: backend-route
  namespace: myapp-project
spec:
  host: backend.example.com
  to:
    kind: Service
    name: backend-service
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

### Route with Wildcard Host

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: api-gateway
  namespace: myapp-project
spec:
  host: "*.api.example.com"
  to:
    kind: Service
    name: api-service
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: Subdomain
```

### NetworkPolicy - Allow External Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
  namespace: myapp-project
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
        - protocol: UDP
          port: 53
```

### NetworkPolicy - Allow Cross-Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: myapp-project
spec:
  podSelector:
    matchLabels:
      app: app-pod
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: monitoring
        - namespaceSelector:
            matchLabels:
              name: logging
      ports:
        - protocol: TCP
          port: 9090
        - protocol: UDP
          port: 514
```

---

## Explanation of YAML

### NetworkPolicy Structure

A NetworkPolicy has three critical sections:

- **`spec.podSelector`**: Selects which pods this policy applies to. An empty selector (`{}`) matches all pods in the namespace.
- **`spec.policyTypes`**: Declares which types of policies are defined (Ingress, Egress, or both).
- **`spec.ingress`**: Rules for incoming traffic. Each rule specifies source pods/namespace and allowed ports.
- **`spec.egress`**: Rules for outgoing traffic. Each rule specifies destination pods/namespace/IPs and allowed ports.

When no `ingress` rules are specified, all ingress traffic is allowed. When no `egress` rules are specified, all egress traffic is allowed.

### Route TLS Configuration

- **`termination: edge`**: TLS terminates at the router. The router uses its own certificate. Backend receives plaintext.
- **`termination: passthrough`**: TLS passes through to backend. Backend must handle TLS.
- **`termination: reencrypt`**: Router terminates TLS, then re-encrypts with backend certificate.
- **`insecureEdgeTerminationPolicy: Redirect`**: HTTP requests are redirected to HTTPS.
- **`insecureEdgeTerminationPolicy: Allow`**: HTTP requests remain on HTTP.
- **`insecureEdgeTerminationPolicy: None`**: HTTP requests are rejected with 426 error.

### Host-Based Routing

Multiple Routes can point to the same Service with different `host` values. The router matches incoming requests based on the Host header. Wildcard patterns (`*.example.com`) match multiple hostnames.

---

## Verification Procedures

### Verify OVNKubernetes Status

```bash
oc get clusteroperators | grep ovs
oc get pods -n openshift-sdn
```

Expected: OVS clusteroperator shows `AVAILABLE=True` with no `DEGRADED=True`.

### Verify Router Pods

```bash
oc get pods -n openshift-ingress
oc get routes -o wide
```

Expected: Router pods are Running, and routes show valid hostnames.

### Verify Route Accessibility

```bash
ROUTE_HOST=$(oc get route myapp-route -o jsonpath='{.spec.host}')
curl -k https://${ROUTE_HOST}
```

Expected: Route resolves and returns application response.

### Verify NetworkPolicy

```bash
# Check policies exist
oc get networkpolicies

# Check OVS flow rules
ssh <node> ovs-appctl flow/list

# Test connectivity
oc exec <source-pod> -- curl <destination-pod-ip>
```

Expected: OVS shows flow rules, and connectivity test respects policy rules.

### Verify TLS Configuration

```bash
curl -kI https://myapp.example.com
oc get secret myapp-tls-secret -o yaml
```

Expected: HTTPS responds with 200, and secret contains valid certificates.

### Verify Ingress Controller

```bash
oc get clusteroperator ingress
oc get pods -n openshift-ingress
```

Expected: Ingress clusteroperator shows `AVAILABLE=True`.

---

## Troubleshooting

### Route Not Accessible from Outside Cluster

**Symptom**: External users cannot access the Route.

**Diagnosis**:

```bash
oc get route <name> -o yaml
oc get pods -n openshift-ingress
oc get clusteroperator ingress
oc logs -n openshift-ingress deploy/router-default
```

**Resolution**:
- Verify the Route has a valid host and is not restricted by wildcardPolicy
- Check if the router pod is running on all nodes
- Verify the Route's Service exists and has endpoints
- Check for certificate expiration
- Verify firewall allows ports 80/443

### Route Returns 503 Error

**Symptom**: Route resolves but returns 503 Service Unavailable.

**Diagnosis**:

```bash
oc get route <name>
oc get endpoints <service-name>
oc get pods -l app=<app-label>
oc describe route <name>
```

**Resolution**:
- Verify the Service has ready endpoints
- Check the Route points to the correct Service name
- Ensure pods pass readiness probes

### NetworkPolicy Blocking Legitimate Traffic

**Symptom**: Pods cannot communicate despite expected connectivity.

**Diagnosis**:

```bash
oc get networkpolicies -n <namespace> -o yaml
oc exec <pod> -- cat /etc/cni/net.d/*
oc adm policy who-can -n <namespace>
```

**Resolution**:
- Verify selectors match the target pods
- Check that at least one NetworkPolicy exists in the namespace
- Review flow rules on OVS
- Ensure port numbers and protocols are correct

### Pod Cannot Reach External Services

**Symptom**: Pods cannot access internet or external APIs.

**Diagnosis**:

```bash
oc exec <pod> -- ping 8.8.8.8
oc exec <pod> -- curl https://example.com
oc get networkpolicies -n <namespace> -o yaml
```

**Resolution**:
- Check for egress NetworkPolicies blocking external traffic
- Verify IPBlock CIDR ranges allow external access
- Check for firewall rules on nodes

### TLS Certificate Expiry

**Symptom**: Route returns certificate errors or 503.

**Diagnosis**:

```bash
openssl x509 -in tls.crt -noout -dates
oc get secret <tls-secret> -o yaml
```

**Resolution**:
- Rotate expired certificates
- Configure certificate auto-renewal
- Update Route with new certificate secret

### Router Pod Crashing

**Symptom**: HAProxy router pods in CrashLoopBackOff.

**Diagnosis**:

```bash
oc get pods -n openshift-ingress
oc logs -n openshift-ingress deploy/router-default --previous
oc describe pod -n openshift-ingress <router-pod>
```

**Resolution**:
- Check for invalid TLS certificate format
- Verify HAProxy configuration syntax
- Check for memory issues on router pods
- Review ingress operator logs

### OVNKubernetes Not Ready

**Symptom**: OVS clusteroperator shows `DEGRADED=True`.

**Diagnosis**:

```bash
oc get clusteroperator ovs -o yaml
oc get pods -n openshift-sdn
oc get nodes -o custom-columns=NODE:.metadata.name,OVN:.status.conditions[?@.type=="OVNReady"].status
```

**Resolution**:
- Check OVN controller logs
- Verify node connectivity and network configuration
- Check for OVS version compatibility
- Review cluster network configuration

---

## Production Best Practices

### Network Security Best Practices

- **Implement default deny policies**: Start with deny-all and allow only what's needed
- **Segment networks**: Separate development, staging, and production workloads
- **Use NetworkPolicies**: Control pod-to-pod communication
- **Enable TLS everywhere**: Use edge termination with redirect for HTTPS
- **Rotate certificates**: Implement automated certificate rotation
- **Monitor traffic**: Use Prometheus metrics to detect anomalies

### Route Configuration Best Practices

- Use custom hostnames instead of auto-generated
- Always enable TLS with edge termination
- Set `insecureEdgeTerminationPolicy: Redirect` for HTTPS enforcement
- Use specific host values instead of wildcards when possible
- Implement custom certificates for sensitive applications
- Regularly audit Route configurations

### NetworkPolicy Best Practices

- Implement default deny as a baseline
- Use explicit allow rules for required traffic
- Document NetworkPolicy purposes clearly
- Review and remove unused policies
- Test policies in staging before production
- Monitor policy effectiveness with traffic analysis

### TLS Certificate Best Practices

- Use Let's Encrypt or enterprise CA for production certificates
- Implement automated certificate renewal
- Store certificates in Secrets, not plain text
- Use re-encryption for backend applications requiring mTLS
- Monitor certificate expiration dates
- Implement certificate pinning for critical services

### Ingress Controller Best Practices

- Configure multiple router replicas for high availability
- Set appropriate replica count based on traffic volume
- Monitor HAProxy resource usage
- Implement circuit breakers for backend failures
- Use health checks for backend services

---

## Certification Exam Notes

### Exam-Relevant Commands You Must Know

```bash
# Check networking status
oc get clusteroperators | grep ovs
oc get pods -n openshift-ingress
oc get routes
oc get networkpolicies

# Create Route
oc expose svc/<name>
oc expose svc/<name> --hostname=<host>
oc apply -f route.yaml

# Edit Route
oc edit route <name>
oc patch route <name> -p='{"spec":{...}}'

# Check NetworkPolicy
oc get networkpolicies -n <namespace>
oc apply -f networkpolicy.yaml

# Test Route
ROUTE_HOST=$(oc get route <name> -o jsonpath='{.spec.host}')
curl -k https://${ROUTE_HOST}

# Check OVS (requires node access)
ssh <node> ovs-vsctl show
ssh <node> ovs-appctl flow/list
```

### Common Exam Scenarios

1. **"Create a Route for an internal Service"** → `oc expose svc/<service-name>`
2. **"Create a Route with custom hostname"** → `oc expose svc/<name> --hostname=<host>`
3. **"Configure edge TLS on a Route"** → Add `tls: {termination: edge}` to Route YAML
4. **"Configure passthrough TLS"** → `tls: {termination: passthrough}`
5. **"Create a NetworkPolicy to deny all ingress"** → Apply deny-all NetworkPolicy YAML
6. **"Allow specific pods to access database"** → Create NetworkPolicy with ingress rules
7. **"Allow external egress on port 443"** → Add egress rule with ipBlock
8. **"Check if OVNKubernetes is healthy"** → `oc get clusteroperators | grep ovs`
9. **"View router pod logs"** → `oc logs -n openshift-ingress deploy/router-default`
10. **"Check Route endpoints"** → `oc get endpoints <service-name>`

### Important Exam Details

- Routes are OpenShift-specific; Services are Kubernetes
- `oc expose` creates both Service and Route when used with a Service
- Edge TLS is the default for Routes; backend receives plaintext
- NetworkPolicies only apply to namespaces with at least one policy
- Router pods run in the `openshift-ingress` namespace
- Default network CIDR is `172.18.0.0/14`
- Service network is `10.128.0.0/14`
- OVS runs on each node as a system service
- HAProxy is the underlying router software

---

## Common Mistakes

### Mistake: Forgetting That NetworkPolicies Are Namespace-Scoped

NetworkPolicies only apply to the namespace where they are created. A NetworkPolicy in `myapp-project` does not affect pods in `production`. Policies must be created in each namespace that requires them.

### Mistake: Assuming NetworkPolicies Apply to All Namespaces

NetworkPolicies only affect namespaces that have at least one NetworkPolicy. If a namespace has no policies, all traffic is allowed by default. Creating a default-deny policy is required to restrict traffic.

### Mistake: Using Wrong Selector in NetworkPolicy

The `podSelector` must match the target pods. Using an empty selector (`{}`) applies the policy to all pods in the namespace. Using a specific selector only affects matching pods.

### Mistake: Not Specifying PolicyTypes

If you only specify `ingress` rules but forget `policyTypes: [Ingress]`, the policy may not work as expected. Always explicitly declare which policy types you're defining.

### Mistake: Forgetting Egress Rules

By default, pods can send traffic anywhere. If you create an ingress-deny policy but forget an egress policy, pods still have unrestricted outbound access. Specify both policy types when needed.

### Mistake: Using Wildcard Routes Without WildcardPolicy

Wildcard routes require `wildcardPolicy: Subdomain` to be set. Without it, wildcard routes are rejected.

### Mistake: Not Checking OVS Flow Rules

When troubleshooting NetworkPolicies, checking OVS flow rules directly is the most reliable way to verify the policy is being enforced.

### Mistake: Hardcoding IP Addresses in NetworkPolicies

Using IP addresses in ipBlock rules can break when pods move between nodes. Prefer selectors and namespace selectors for dynamic targeting.

---

## Chapter Summary

This chapter covered network security configuration in OpenShift Container Platform. You learned about OVNKubernetes as the default CNI plugin and how it provides software-defined networking across nodes. You configured the ingress controller and created Routes with various TLS termination modes (edge, passthrough, re-encryption). You created and edited external Routes for custom hostnames and TLS requirements. You controlled cluster network ingress using NetworkPolicies, implementing default deny policies and allowing specific pod-to-pod traffic. You secured external traffic using TLS certificates, including generating self-signed certificates and configuring re-encryption routes. You secured internal traffic using NetworkPolicies and service mesh concepts. You diagnosed SDN and ingress controller failures, checked OVS status, and verified Route accessibility. Production best practices emphasized network segmentation, default deny policies, TLS everywhere, and certificate management. Troubleshooting covered route accessibility issues, NetworkPolicy blocking legitimate traffic, certificate expiry, and router pod failures.

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Check CNI status | `oc get clusteroperators | grep ovs` |
| Check router pods | `oc get pods -n openshift-ingress` |
| List Routes | `oc get routes` |
| Expose Service as Route | `oc expose svc/<name>` |
| Expose with hostname | `oc expose svc/<name> --hostname=<host>` |
| Edit Route | `oc edit route <name>` |
| Patch Route | `oc patch route <name> -p='{...}'` |
| List NetworkPolicies | `oc get networkpolicies -n <ns>` |
| Create NetworkPolicy | `oc apply -f networkpolicy.yaml` |
| Check OVS | `ssh <node> ovs-vsctl show` |
| Check OVS flows | `ssh <node> ovs-appctl flow/list` |
| Test Route | `curl -k https://$(oc get route <name> -o jsonpath='{.spec.host}')` |
| Check ingress operator | `oc get clusteroperator ingress` |
| Check endpoints | `oc get endpoints <service>` |

### TLS Termination Modes

| Mode | Description | Backend Traffic |
|------|-------------|-----------------|
| `edge` | TLS terminated at router | Plaintext |
| `passthrough` | TLS passed to backend | Encrypted |
| `reencrypt` | TLS terminated and re-encrypted | Encrypted |

### Route Host-Based Routing

| Configuration | Use Case |
|---------------|----------|
| Single host | One Service per hostname |
| Wildcard (`*.example.com`) | Multiple subdomains |
| Multiple Routes | Different Services per hostname |
| Path-based | Same Service, different paths |

### NetworkPolicy PolicyTypes

| Type | Controls |
|------|----------|
| `Ingress` | Incoming traffic to pods |
| `Egress` | Outgoing traffic from pods |
| (both) | Both ingress and egress |

### NetworkPolicy Selectors

| Selector | Matches |
|----------|---------|
| `{}` (empty) | All pods in namespace |
| `matchLabels: app=myapp` | Pods with label `app=myapp` |
| `namespaceSelector: name=monitoring` | Pods in namespace with label `name=monitoring` |
| `ipBlock: cidr=0.0.0.0/0` | Any IP address |
| `ipBlock: cidr=10.0.0.0/8, except=10.1.0.0/16` | IP range excluding subnet |

### Common Network CIDRs

| CIDR | Purpose |
|------|---------|
| `172.18.0.0/14` | Pod subnet (OVNKubernetes default) |
| `10.128.0.0/14` | Service network |
| `10.132.0.0/16` | Cluster service network |
| `10.0.0.0/16` | Machine network |

---

## Review Questions

1. What is OVNKubernetes and why is it used in OpenShift?

2. Explain the difference between edge, passthrough, and re-encryption TLS termination.

3. What does a NetworkPolicy with `podSelector: {}` and `policyTypes: [Ingress]` do?

4. How do you create a Route with a custom hostname and edge TLS?

5. What is the difference between `oc expose service` and `oc expose deployment`?

6. How do you allow pods in one namespace to access pods in another namespace?

7. What happens if you create a NetworkPolicy in a namespace but no pods match the selector?

8. How do you check if the OVNKubernetes operator is healthy?

9. What command lists all Routes in the cluster?

10. How do you configure a Route to accept both HTTP and HTTPS requests?

11. What is the purpose of the `insecureEdgeTerminationPolicy` field in a Route?

12. How do you create a default-deny NetworkPolicy for a namespace?

13. What does the `ipBlock` selector match in a NetworkPolicy?

14. How do you verify that a Route is accessible externally?

15. What namespace contains the HAProxy router pods?

---

## Answers

1. OVNKubernetes is the default CNI (Container Network Interface) plugin in OpenShift 4.18. It provides software-defined networking using OVS, enabling pod-to-pod communication across nodes, network isolation, and NetworkPolicy enforcement through VXLAN encapsulation.

2. **Edge termination**: TLS is terminated at the HAProxy router, and plaintext is forwarded to the backend. **Passthrough**: TLS traffic is passed unchanged to the backend, which handles TLS. **Re-encryption**: TLS is terminated at the router and re-encrypted with a different certificate for the backend.

3. It denies all incoming traffic to all pods in the namespace. The empty selector matches all pods, and with only Ingress in policyTypes, no ingress traffic is allowed unless explicitly permitted by another NetworkPolicy.

4. `oc expose svc/<service-name> --hostname=<custom-hostname>`. Or create a Route YAML with `host: <custom-hostname>` and `tls: {termination: edge}`.

5. `oc expose service` creates a Service and a Route pointing to it. `oc expose deployment` creates a Service, a Route, and deploys the application.

6. Create a NetworkPolicy in the source namespace that allows egress to the destination namespace using `namespaceSelector: {matchLabels: {name: <destination-namespace>}}`.

7. The NetworkPolicy exists but has no effect. NetworkPolicies require matching pods to be present in the namespace to have any enforcement impact.

8. `oc get clusteroperators | grep ovs` or `oc get clusteroperator ovs`. The operator should show `AVAILABLE=True` with no `DEGRADED=True`.

9. `oc get routes` or `oc get routes -A` for all namespaces.

10. Configure `insecureEdgeTerminationPolicy: Allow` so HTTP is accepted and forwarded as-is (not redirected).

11. It controls the behavior of HTTP (non-HTTPS) requests: Allow (forward as HTTP), Redirect (301 to HTTPS), or None (reject with 426).

12. Create a NetworkPolicy with `podSelector: {}`, `policyTypes: [Ingress]` (and/or `[Egress]`) and no ingress/egress rules.

13. The `ipBlock` selector matches IP addresses within the specified CIDR range. The `except` field excludes specific subnets from the range.

14. `curl -k https://$(oc get route <name> -o jsonpath='{.spec.host}')` from an external machine.

15. The `openshift-ingress` namespace contains the HAProxy router pods.
