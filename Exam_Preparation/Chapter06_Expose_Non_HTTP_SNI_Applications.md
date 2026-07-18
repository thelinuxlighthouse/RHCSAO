# Chapter 6: Expose Non-HTTP/SNI Applications

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure LoadBalancer services for external application access
- Expose non-HTTP/SNI applications to external users
- Apply appropriate service types for different use cases
- Understand the limitations of OpenShift's Route-based external access
- Configure external application access for databases and internal services
- Use NodePort services as an alternative to Routes
- Implement external access patterns for TCP-based applications
- Troubleshoot external access issues
- Apply external access best practices in production environments

---

## OpenShift Concepts

### OpenShift External Access Patterns

OpenShift provides multiple mechanisms for exposing applications externally:

| Mechanism | Protocol | Use Case |
|-----------|----------|----------|
| **Route** | HTTP/HTTPS (SNI) | Web applications, APIs |
| **LoadBalancer Service** | Any protocol | TCP/UDP services, databases |
| **NodePort Service** | Any protocol | Development, debugging, direct node access |
| **Port Forwarding** | Any protocol | Temporary access, debugging |

### LoadBalancer Service

A LoadBalancer Service requests an external load balancer from your cloud provider or network infrastructure. When applied to a Service:

1. The cloud provider provisions an external load balancer
2. The load balancer receives traffic on the LoadBalancer port
3. Traffic is forwarded to the Service's ClusterIP
4. The ClusterIP routes to backend pods

In OpenShift, LoadBalancer services work similarly to Kubernetes, but with OpenShift-specific enhancements for better integration.

### NodePort Service

A NodePort Service exposes a Service on a static port (30000-32767) on each node's IP address. Access pattern:

```
External → Node IP:NodePort → Service ClusterIP → Pods
```

NodePort is useful for:
- Development and testing
- Applications that don't support HTTP/HTTPS
- Services that need direct node access
- When LoadBalancer is not available

### Software SNI (Server Name Indication)

SNI is an extension to the TLS protocol that allows multiple SSL certificates to be hosted on the same IP address and port. OpenShift Routes use SNI to route TLS traffic to different backends based on the hostname in the TLS handshake.

**Non-SNI applications** are applications that:
- Do not use HTTPS/TLS
- Use custom protocols (TCP, UDP, gRPC without TLS)
- Require direct IP-based routing without hostname-based TLS routing

These applications cannot use OpenShift Routes and require alternative external access methods.

### HAProxy Router Limitations

The OpenShift HAProxy router:
- **Supports**: HTTP and HTTPS with SNI
- **Does NOT support**: Raw TCP, UDP, or non-SNI TLS connections

For non-HTTP/SNI applications, you must use LoadBalancer or NodePort services.

---

## Architecture and Internal Operation

### LoadBalancer Service Flow

```
External Request → Cloud Provider LB
                          │
                          ├── Load Balancer IP
                          │
                          └── Openshift Node
                                  │
                                  └── Openshift LB Service
                                          │
                                          ├── kube-proxy (iptables/IPVS mode)
                                          │
                                          └── Openshift Service ClusterIP
                                                  │
                                                  └── Openshift Pods
```

In OpenShift, the LoadBalancer Service works through:

1. **Cloud Provider Integration**: The cloud provider (AWS, Azure, GCP, etc.) provisions the external load balancer
2. **kube-proxy**: Programs iptables or IPVS rules to forward traffic from the LoadBalancer IP to the Service IP
3. **Service ClusterIP**: Routes traffic to matching pods based on selectors
4. **Pods**: Receive traffic on the targetPort

### NodePort Service Flow

```
External Request → Any Node IP:NodePort
                          │
                          └── Node Network Stack
                                  │
                                  ├── iptables/IPVS rules
                                  │
                                  └── Openshift Service ClusterIP
                                          │
                                          └── Openshift Pods
```

NodePort uses:

1. **Node Port Range**: Each node exposes the Service on ports 30000-32767
2. **kube-proxy**: Programs rules to forward to the Service ClusterIP
3. **Service ClusterIP**: Routes to pods
4. **Multiple Endpoints**: Each node can access the Service independently

### LoadBalancer vs. NodePort Comparison

```
LoadBalancer Service:
├── External Load Balancer IP (cloud-provided)
├── Single entry point
├── Managed by cloud provider
├── Higher cost (LB instance)
└── Best for production external access

NodePort Service:
├── Node IP:Port per node
├── Multiple entry points
├── Managed by OpenShift
├── Lower cost
└── Best for development, debugging
```

---

## Components and Terminology

| Term | Description |
|------|-------------|
| LoadBalancer Service | Kubernetes Service type that provisions external LB |
| NodePort Service | Kubernetes Service type exposing port on each node |
| SNI | Server Name Indication for hostname-based TLS routing |
| HAProxy | OpenShift's Layer 7 load balancer software |
| kube-proxy | Kubernetes network proxy for Service implementation |
| IPVS | Virtual Server kernel module for load balancing |
| iptables | Linux packet filtering and NAT tool |
| Port Forwarding | Manual TCP forwarding from localhost to cluster |
| External IP | IP address of a LoadBalancer or NodePort service |
| ClusterIP | Internal virtual IP for cluster-wide Service access |
| TargetPort | Port on the pod that receives traffic |
| NodePort | Static port on each node for NodePort services |
| ExternalTrafficPolicy | Cluster (default) or Local routing policy |
| Cloud Provider | AWS, Azure, GCP, or other cloud infrastructure |

---

## Administration Tasks

### Configuring a LoadBalancer Service

#### Creating a Basic LoadBalancer Service

```bash
oc apply -f loadbalancer-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
  namespace: myapp-project
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: http
  type: LoadBalancer
```

#### Creating LoadBalancer for Multiple Ports

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
  namespace: myapp-project
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: http
    - protocol: TCP
      port: 443
      targetPort: 8443
      name: https
  type: LoadBalancer
```

#### Creating LoadBalancer for Database Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-lb
  namespace: database-project
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
      name: postgres
  type: LoadBalancer
```

#### Checking LoadBalancer External IP

```bash
oc get svc myapp-lb
oc get svc myapp-lb -o wide
```

The external IP column shows the cloud-provided load balancer IP.

#### Using ExternalTrafficPolicy

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Cluster
  # or
  externalTrafficPolicy: Local
```

- **Cluster**: Load balancer can forward to any pod (default)
- **Local**: Load balancer only forwards to pods on the same node (better for stateful apps)

### Configuring NodePort Service

#### Creating a NodePort Service

```bash
oc apply -f nodeport-service.yaml
```

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
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080
      name: http
  type: NodePort
```

#### Creating NodePort with Custom Port

```yaml
spec:
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30100
      name: app
```

#### Accessing NodePort Service

```bash
# Get node IP
oc get nodes -o wide

# Access via node IP
curl http://<node-ip>:30080

# Access from another node
curl http://<other-node-ip>:30080
```

#### Viewing NodePort Endpoints

```bash
oc get endpoints myapp-nodeport
oc get endpoints myapp-nodeport -o wide
```

### Configuring External Application Access

#### Port Forwarding for Temporary Access

```bash
# Forward port from localhost to Service
oc port-forward svc/myapp-service 8080:80 -n myapp-project

# Forward to pod directly
oc port-forward pod/myapp-pod-12345 8080:8080

# Forward with multiple ports
oc port-forward svc/myapp-service 8080:80 8443:443 -n myapp-project
```

#### Using External DNS with LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
  namespace: myapp-project
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: external-dns-config
  namespace: external-dns
data:
  config.yml: |
    access-key-id: KEY
    secret-access-key: SECRET
    zone-name: example.com
    txt-owner-id: OWNER
```

#### Accessing Internal Services from Outside Cluster

```yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-api
  namespace: internal-project
spec:
  selector:
    app: api
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: LoadBalancer
```

### Exposing Non-HTTP Applications

#### TCP Service (Database)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: database-project
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
  type: LoadBalancer
```

#### UDP Service (DNS)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: dns-service
  namespace: dns-project
spec:
  selector:
    app: dns
  ports:
    - protocol: UDP
      port: 53
      targetPort: 53
  type: LoadBalancer
```

#### gRPC Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grpc-service
  namespace: api-project
spec:
  selector:
    app: grpc-api
  ports:
    - protocol: TCP
      port: 50051
      targetPort: 50051
      name: grpc
  type: LoadBalancer
```

---

## YAML Examples

### LoadBalancer Service for Web Application

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-lb
  namespace: myapp-project
  labels:
    app: webapp
    environment: production
spec:
  selector:
    app: webapp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
  type: LoadBalancer
  externalTrafficPolicy: Cluster
  sessionAffinity: None
```

### LoadBalancer Service for Database

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgresql-lb
  namespace: database-project
  labels:
    app: postgresql
    tier: database
spec:
  selector:
    app: postgresql
  ports:
    - name: postgres
      protocol: TCP
      port: 5432
      targetPort: 5432
  type: LoadBalancer
  externalTrafficPolicy: Local
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

### NodePort Service for Custom Protocol

```yaml
apiVersion: v1
kind: Service
metadata:
  name: custom-protocol
  namespace: myapp-project
spec:
  selector:
    app: myapp
  ports:
    - name: api
      protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30080
    - name: admin
      protocol: TCP
      port: 9090
      targetPort: 9090
      nodePort: 30090
  type: NodePort
```

### LoadBalancer with Multiple Protocols

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-protocol
  namespace: myapp-project
spec:
  selector:
    app: myapp
  ports:
    - name: tcp
      protocol: TCP
      port: 80
      targetPort: 8080
    - name: udp
      protocol: UDP
      port: 53
      targetPort: 53
    - name: grpc
      protocol: TCP
      port: 50051
      targetPort: 50051
  type: LoadBalancer
```

### LoadBalancer with ExternalTrafficPolicy Local

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-app
  namespace: stateful-project
spec:
  selector:
    app: stateful
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
  externalTrafficPolicy: Local
  publishNotReadyAddresses: true
```

### NodePort with External DNS Integration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-access
  namespace: myapp-project
  annotations:
    external-dns.alpha.kubernetes.io/hostname: myapp.example.com
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080
  type: NodePort
```

---

## Explanation of YAML

### LoadBalancer Service Fields

- **`type: LoadBalancer`**: Tells OpenShift to provision an external load balancer
- **`selector`**: Matches pods to include in the Service's endpoints
- **`ports`**: Defines port mappings between Service port and pod targetPort
- **`externalTrafficPolicy: Cluster`**: Default policy allowing traffic to any node's pods
- **`externalTrafficPolicy: Local`**: Restricts traffic to pods on the same node, preserving client IP
- **`sessionAffinity: ClientIP`**: Enables sticky sessions based on client IP
- **`sessionAffinityConfig.clientIP.timeoutSeconds`**: Duration for sticky session (default 10800s = 3 hours)

### NodePort Service Fields

- **`type: NodePort`**: Exposes Service on each node's IP
- **`nodePort`**: Specific port number (30000-32767) on each node
- Without `nodePort`, OpenShift assigns a random port in the range

### ExternalTrafficPolicy Impact

| Policy | Behavior | Use Case |
|--------|----------|----------|
| Cluster (default) | Load balancer can forward to any node | Stateless applications, better load distribution |
| Local | Load balancer only forwards to pods on same node | Stateful applications, preserve client IP |

---

## Verification Procedures

### Verify LoadBalancer External IP

```bash
oc get svc myapp-lb
oc get svc myapp-lb -o wide

# Wait for external IP to be assigned
watch oc get svc myapp-lb -o wide
```

Expected: External IP column shows a valid IP address.

### Verify NodePort Access

```bash
oc get nodes -o wide
oc get svc myapp-nodeport

# Test from outside cluster
curl http://<node-ip>:30080
```

Expected: Service shows NodePort, and curl returns application response.

### Verify External Application Connectivity

```bash
# Get Service IP
oc get svc myapp-lb -o jsonpath='{.spec.clusterIP}'

# Get endpoint IPs
oc get endpoints myapp-lb

# Test connectivity
curl http://<external-ip>
```

Expected: All endpoints show running pod IPs, and curl succeeds.

### Verify LoadBalancer Traffic Distribution

```bash
# Check Service endpoints
oc get endpoints myapp-lb -o wide

# Check pod distribution across nodes
oc get pods -o wide -l app=myapp
```

Expected: Pods are distributed across nodes, and endpoints list matches.

### Verify NodePort Endpoints

```bash
oc get endpoints myapp-nodeport -o yaml
```

Expected: Endpoints section shows pod IPs for each port.

### Verify External Access from Different Nodes

```bash
# Get node IPs
oc get nodes -o custom-columns=NODE:.metadata.name,INTERNAL-IP:.status.addresses[?@.type=="InternalIP"].address

# Test from each node
curl http://<node1-ip>:30080
curl http://<node2-ip>:30080
```

Expected: All nodes respond with the same Service content.

---

## Troubleshooting

### LoadBalancer External IP Not Assigned

**Symptom**: Service shows `<pending>` in external IP column.

**Diagnosis**:

```bash
oc get svc myapp-lb
oc get svc myapp-lb -o yaml
oc get cloudprovider
oc get pods -n openshift-cloud-network-operator
```

**Resolution**:
- Verify cloud provider integration is configured
- Check cloud provider operator status
- Ensure cloud provider credentials are valid
- For on-premise clusters, LoadBalancer may not work without external LB

### LoadBalancer Service Not Routing Traffic

**Symptom**: External IP responds, but requests fail.

**Diagnosis**:

```bash
# Check endpoints
oc get endpoints myapp-lb

# Check pod health
oc get pods -l app=myapp

# Check Service selector
oc get svc myapp-lb -o yaml | grep -A 5 selector

# Check cloud provider logs
oc logs -n openshift-cloud-network-operator deploy/...
```

**Resolution**:
- Verify pods match the Service selector
- Ensure pods are in Running state
- Check cloud provider load balancer health
- Verify targetPort matches container port

### NodePort Not Accessible

**Symptom**: Cannot connect to NodePort from outside cluster.

**Diagnosis**:

```bash
# Check Service configuration
oc get svc myapp-nodeport

# Check firewall on node
ssh <node> firewall-cmd --list-all

# Check node iptables
ssh <node> iptables -L -n | grep myapp-nodeport

# Check Service endpoints
oc get endpoints myapp-nodeport
```

**Resolution**:
- Verify firewall allows NodePort range (30000-32767)
- Ensure Service has endpoints
- Check node is accessible from client
- Verify targetPort is correct

### ExternalTrafficPolicy Local Issues

**Symptom**: LoadBalancer only routes to pods on specific nodes.

**Diagnosis**:

```bash
oc get svc myapp-lb -o yaml | grep externalTrafficPolicy
oc get endpoints myapp-lb -o wide
```

**Resolution**:
- This is expected behavior for Local policy
- If pods don't exist on the node, traffic fails
- Consider Cluster policy for better distribution

### Port Forwarding Not Working

**Symptom**: `oc port-forward` fails or doesn't connect.

**Diagnosis**:

```bash
oc port-forward svc/myapp-service 8080:80
# Check if port is in use
lsof -i :8080
netstat -tlnp | grep 8080
```

**Resolution**:
- Ensure port 8080 is not already in use
- Verify Service exists and has endpoints
- Check pod health
- Use different local port if needed

### Non-HTTP Application Cannot Connect

**Symptom**: TCP/UDP applications cannot reach external endpoints.

**Diagnosis**:

```bash
# Check Service type
oc get svc myapp-service

# Check node firewall
ssh <node> firewall-cmd --list-all

# Check network connectivity
telnet <external-ip> <port>
nc -vz <external-ip> <port>
```

**Resolution**:
- Ensure firewall allows the port
- Verify cloud provider LB supports the protocol
- Check for network policies blocking traffic

---

## Production Best Practices

### LoadBalancer Service Best Practices

- **Use LoadBalancer for production external access**: Provides managed, scalable external access
- **Configure externalTrafficPolicy appropriately**: Use Local for stateful apps, Cluster for stateless
- **Set sessionAffinity for stateful applications**: Maintains client connections
- **Monitor external IP assignment**: Ensure cloud provider LB is provisioned
- **Use descriptive names**: Include environment and application in service name
- **Document external IPs**: Track load balancer IPs for monitoring and alerts

### NodePort Service Best Practices

- **Use NodePort for development and debugging**: Not for production external access
- **Document assigned ports**: Track NodePort assignments
- **Avoid port conflicts**: Use unique ports for different services
- **Consider firewall rules**: NodePort opens ports on all nodes
- **Plan for node scaling**: NodePort works with any node count

### External Access Security

- **Use TLS for all external access**: Configure HTTPS with proper certificates
- **Implement authentication**: Require valid credentials for external access
- **Use rate limiting**: Protect against DDoS attacks
- **Monitor access logs**: Track external access patterns
- **Implement IP whitelisting**: Restrict access to known IPs when possible
- **Use WAF**: Deploy Web Application Firewall for HTTP traffic

### Network Security

- **Apply NetworkPolicies**: Control pod-to-pod traffic
- **Use externalTrafficPolicy: Local**: For stateful applications
- **Restrict NodePort range**: Use specific ports instead of defaults
- **Implement firewall rules**: Control node-level access
- **Monitor traffic patterns**: Detect anomalies and unauthorized access

---

## Certification Exam Notes

### Exam-Relevant Commands You Must Know

```bash
# Create LoadBalancer service
oc apply -f loadbalancer-service.yaml
oc expose svc/<name> --type=LoadBalancer

# Create NodePort service
oc apply -f nodeport-service.yaml
oc expose svc/<name> --type=NodePort

# Check Service status
oc get svc
oc get svc -o wide

# Check external IP
oc get svc <name> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Access NodePort
curl http://<node-ip>:<nodeport>

# Port forwarding
oc port-forward svc/<name> <local-port>:<port>

# Edit Service
oc edit svc/<name>

# Patch Service
oc patch svc/<name> -p='{"spec":{"type":"LoadBalancer"}}'
```

### Common Exam Scenarios

1. **"Create a LoadBalancer service for an application"** → `oc apply -f service.yaml` with `type: LoadBalancer`
2. **"Check the external IP of a LoadBalancer service"** → `oc get svc <name> -o wide`
3. **"Create a NodePort service"** → `oc apply -f service.yaml` with `type: NodePort`
4. **"Access a NodePort service from outside cluster"** → `curl http://<node-ip>:<nodeport>`
5. **"Port forward a Service"** → `oc port-forward svc/<name> 8080:80`
6. **"Change a Service to LoadBalancer type"** → `oc patch svc <name> -p='{"spec":{"type":"LoadBalancer"}}'`
7. **"Configure externalTrafficPolicy: Local"** → Add to Service spec YAML
8. **"Expose a database externally"** → Create LoadBalancer service with database port
9. **"Check if LoadBalancer external IP is assigned"** → `oc get svc <name> | grep -i pending`
10. **"Create NodePort with custom port"** → `nodePort: 30080` in Service spec

### Important Exam Details

- LoadBalancer services require cloud provider integration
- NodePort uses ports 30000-32767 by default
- LoadBalancer external IP may take time to be assigned
- ExternalTrafficPolicy: Local preserves client IP
- Port forwarding works for any Service type
- Routes only work for HTTP/HTTPS (SNI)
- LoadBalancer and NodePort work for any protocol (TCP, UDP, etc.)

---

## Common Mistakes

### Mistake: Using Routes for Non-HTTP Applications

Attempting to expose a TCP/UDP application with a Route will fail because Routes only support HTTP/HTTPS. Use LoadBalancer or NodePort services for non-HTTP applications.

### Mistake: Forgetting to Wait for External IP

LoadBalancer services show `<pending>` for external IP immediately after creation. It can take minutes for the cloud provider to provision the load balancer. Check periodically with `watch`.

### Mistake: Using NodePort for Production External Access

NodePort exposes the Service on every node, which is less secure and harder to manage than a single LoadBalancer IP. Use LoadBalancer for production external access.

### Mistake: Wrong TargetPort Configuration

The `targetPort` must match the container's listening port. If the container listens on 8080 but targetPort is 80, traffic won't reach the application.

### Mistake: Not Checking NodePort Range

NodePort values must be in the range 30000-32767. Using a port outside this range causes the Service to not work correctly.

### Mistake: Ignoring ExternalTrafficPolicy

Not understanding the difference between Cluster and Local can cause unexpected behavior with stateful applications. Use Local for stateful apps to preserve client IP.

### Mistake: Assuming LoadBalancer Works on All Clusters

LoadBalancer services require cloud provider integration. On-premise clusters or clusters without cloud provider configuration may not support LoadBalancer services.

### Mistake: Forgetting to Update Firewall Rules

NodePort services require firewall rules to allow traffic on the NodePort range. Without proper firewall configuration, external access will fail.

---

## Chapter Summary

This chapter covered exposing non-HTTP/SNI applications in OpenShift Container Platform. You learned to configure LoadBalancer services for external access through cloud provider load balancers and NodePort services for direct node access. You understood the limitations of OpenShift Routes, which only support HTTP/HTTPS with SNI, making them unsuitable for TCP/UDP or non-SNI applications. You configured LoadBalancer services with appropriate externalTrafficPolicy, sessionAffinity, and port mappings. You created NodePort services with custom ports and verified external access from different nodes. You configured external application access for databases and other internal services using LoadBalancer services. You implemented TCP, UDP, and gRPC service exposure patterns. You diagnosed LoadBalancer external IP assignment issues, routing failures, NodePort accessibility problems, and port forwarding failures. Production best practices emphasized LoadBalancer for production external access, appropriate externalTrafficPolicy selection, security configurations, and firewall management.

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Create LoadBalancer service | `oc apply -f service.yaml` with `type: LoadBalancer` |
| Create NodePort service | `oc apply -f service.yaml` with `type: NodePort` |
| List Services | `oc get svc` |
| List Services with external IP | `oc get svc -o wide` |
| Check external IP | `oc get svc <name> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` |
| Access NodePort | `curl http://<node-ip>:<nodeport>` |
| Port forward | `oc port-forward svc/<name> <local-port>:<port>` |
| Edit Service | `oc edit svc <name>` |
| Patch Service type | `oc patch svc <name> -p='{"spec":{"type":"LoadBalancer"}}'` |
| Get node IPs | `oc get nodes -o wide` |
| Check endpoints | `oc get endpoints <service-name>` |
| Test connectivity | `curl http://<external-ip>` |
| Check firewall | `firewall-cmd --list-all` |

### Service Types Comparison

| Type | External Access | Protocol | Best For |
|------|----------------|----------|----------|
| LoadBalancer | Yes (single IP) | Any | Production external access |
| NodePort | Yes (per node) | Any | Development, debugging |
| ClusterIP | No | Any | Internal cluster services |
| ExternalName | Yes (DNS) | Any | External service alias |

### LoadBalancer Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| `externalTrafficPolicy` | Cluster | Cluster or Local |
| `sessionAffinity` | None | None or ClientIP |
| `type` | ClusterIP | ClusterIP, NodePort, LoadBalancer |

### NodePort Port Range

- **Range**: 30000-32767
- **Default**: Automatically assigned if not specified
- **Custom**: Must be within range

### Port Forwarding Syntax

```bash
oc port-forward svc/<service-name> <local-port>:<remote-port> -n <namespace>
oc port-forward pod/<pod-name> <local-port>:<remote-port>
oc port-forward svc/<name> <local-port-1>:<remote-1> <local-port-2>:<remote-2>
```

---

## Review Questions

1. What is the difference between a LoadBalancer service and a Route?

2. When would you use a NodePort service instead of a LoadBalancer service?

3. How do you expose a TCP-based database to external users?

4. What does `externalTrafficPolicy: Local` do?

5. How do you access a NodePort service from outside the cluster?

6. What is the default port range for NodePort services?

7. Why can't you use a Route for a UDP application?

8. How do you check if a LoadBalancer service has an external IP assigned?

9. What command forwards port 8080 on your local machine to port 80 on a Service?

10. What is the difference between ClusterIP and LoadBalancer Service types?

11. How do you create a LoadBalancer service with sticky sessions?

12. What happens if you try to expose a non-HTTP application with `oc expose`?

13. How do you access a NodePort service from a different node?

14. What is the purpose of the `targetPort` field in a Service?

15. When would you use `sessionAffinity: ClientIP`?

---

## Answers

1. A **LoadBalancer service** provisions an external load balancer (cloud provider or on-premise) that provides a single external IP for the Service. A **Route** is an OpenShift Layer 7 load balancer that only supports HTTP/HTTPS with SNI and provides DNS-based routing.

2. Use a **NodePort service** for development, debugging, or when LoadBalancer is not available. Use **LoadBalancer** for production external access because it provides a single, managed external IP.

3. Create a **LoadBalancer service** with the database port: `type: LoadBalancer`, `port: 5432`, `targetPort: 5432`. The cloud provider load balancer will accept TCP connections on port 5432 and forward to the database pods.

4. **`externalTrafficPolicy: Local`** restricts the load balancer to only forward traffic to pods on the same node as the client. This preserves client IP and is required for stateful applications that store data locally.

5. Get the node IP with `oc get nodes -o wide`, then access via `http://<node-ip>:<nodeport>`. Any node IP can access the NodePort service.

6. The default range is **30000-32767**. If you don't specify a custom `nodePort`, OpenShift assigns one from this range.

7. Routes use the HAProxy router, which only supports **HTTP/HTTPS with SNI**. UDP applications don't use HTTP and don't support SNI, so Routes cannot handle them.

8. Use `oc get svc <name> -o wide` and look for a value in the EXTERNAL-IP column instead of `<pending>`. Or check with `oc get svc <name> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'`.

9. `oc port-forward svc/<service-name> 8080:80 -n <namespace>`

10. **ClusterIP** is the default internal-only Service type accessible only within the cluster. **LoadBalancer** provisions an external load balancer and provides external access.

11. Add `sessionAffinity: ClientIP` to the Service spec. Optionally configure `sessionAffinityConfig.clientIP.timeoutSeconds` to set the sticky session duration.

12. The `oc expose` command will **fail** for non-HTTP applications because Routes only support HTTP/HTTPS. Use `oc apply -f service.yaml` with `type: LoadBalancer` or `type: NodePort` instead.

13. NodePort services are accessible from **any node's IP**. Access via `http://<other-node-ip>:<nodeport>`. The Service routes traffic regardless of which node the request reaches.

14. **`targetPort`** specifies the port on the container that receives traffic. It must match the container's listening port for the Service to work correctly.

15. Use **`sessionAffinity: ClientIP`** for stateful applications (databases, sessions) where you want the same client to always reach the same pod. This maintains connection state across requests.
