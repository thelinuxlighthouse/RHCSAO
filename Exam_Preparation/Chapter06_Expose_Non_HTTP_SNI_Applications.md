# Chapter 6: Expose Non-HTTP or Non-SNI Applications

> **Exam baseline:** EX280 on OpenShift Container Platform 4.18.  
> **Reviewed:** 2026-07-19.  
> A standard OpenShift Route handles HTTP, HTTPS, or TLS protocols that provide SNI. Use a `LoadBalancer` Service for ordinary TCP/UDP or TLS clients without SNI when the platform supplies a load-balancer implementation.

## What You Must Be Able to Do

By the end of this chapter, you should be able to:

- decide whether a Route or Service is the correct exposure method;
- configure and verify a `LoadBalancer` Service;
- understand on-premises load balancing with MetalLB at a high level;
- configure external DNS and firewall access around the assigned address;
- use `NodePort` and TCP port-forwarding as controlled alternatives;
- explain `externalTrafficPolicy: Cluster` versus `Local`; and
- troubleshoot `<pending>`, missing endpoints, wrong ports, and unreachable addresses.

---

## 1. Choose the Correct Exposure Method

| Method | Protocol/use | Important limitation |
|---|---|---|
| OpenShift `Route` | HTTP, HTTPS, and TLS protocols whose client sends SNI | Not a generic raw TCP/UDP proxy. |
| `LoadBalancer` Service | TCP, UDP, SCTP, or mixed ports when supported by the provider | Needs cloud/provider integration or an on-premises implementation such as MetalLB. |
| `NodePort` Service | Exposes a port through eligible node addresses | Requires network reachability/firewall design; not a hostname router. |
| `oc port-forward` | Temporary TCP access from the administrator's workstation | Process-bound and non-persistent; not UDP and not production exposure. |
| `ClusterIP` Service | Internal stable IP/DNS | Not reachable directly from ordinary external clients. |

### SNI in easy English

With TLS, encryption starts before an HTTP `Host` header can be read. Server Name Indication puts the requested hostname in the TLS ClientHello, allowing a shared router to select a certificate and backend.

A passthrough Route can carry a TLS application other than HTTPS, but the client must send SNI and the router must be configured for the connection. Raw PostgreSQL, arbitrary TCP, UDP, and old/non-SNI TLS clients normally need a Service-based exposure.

gRPC is based on HTTP/2. Do not automatically classify it as “raw non-HTTP.” A gRPC application can use a Route when the TLS/HTTP2, ALPN, backend, and Ingress Controller configuration support it. A `LoadBalancer` Service is still an option when direct L4 exposure is required.

---

## 2. How a LoadBalancer Service Works

```text
external client
      |
assigned IP or hostname
      |
cloud load balancer or MetalLB-advertised address
      |
OpenShift Service port
      |
ready EndpointSlice address:targetPort
      |
pod application
```

Creating `type: LoadBalancer` is a request to the platform. Kubernetes does not create a physical load balancer by itself.

- On supported cloud platforms, the cloud controller/provider normally creates the load balancer.
- On bare-metal or similar networks, MetalLB can advertise addresses from administrator-defined pools after the MetalLB Operator and its configuration are installed.
- Without an implementation, the Service can remain at `EXTERNAL-IP <pending>` indefinitely.

OVN-Kubernetes implements Service traffic on OpenShift; do not assume a `kube-proxy` daemon or IPVS design from a generic Kubernetes diagram.

---

## 3. Create a LoadBalancer Service

Assume a Deployment has pods labeled `app: payments` and listens on TCP 8443.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-lb
  namespace: payments
spec:
  type: LoadBalancer
  selector:
    app: payments
  ports:
  - name: payments
    protocol: TCP
    port: 443
    targetPort: 8443
```

```bash
oc apply -f payments-lb.yaml
oc get service/payments-lb -n payments -o wide
oc describe service/payments-lb -n payments
oc get endpointslice -n payments \
  -l kubernetes.io/service-name=payments-lb -o wide
```

Meanings:

- `port: 443` is the port offered by the Service/load balancer.
- `targetPort: 8443` is the port on selected pods. It can also be a named container port.
- `selector` must match pod labels.
- `status.loadBalancer.ingress` is set by the provider and can contain an IP **or hostname**.

Print both possible status forms:

```bash
oc get service/payments-lb -n payments \
  -o jsonpath='{range .status.loadBalancer.ingress[*]}{.ip}{.hostname}{"\n"}{end}'
```

### UDP example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: syslog-udp
  namespace: logging-app
spec:
  type: LoadBalancer
  selector:
    app: syslog
  ports:
  - name: syslog
    protocol: UDP
    port: 514
    targetPort: 1514
```

Provider support for UDP and mixed protocols varies. Check Service events and the environment requirements.

### Multi-port Service

Every port must have a unique name when a Service defines several ports:

```yaml
spec:
  type: LoadBalancer
  selector:
    app: dns-server
  ports:
  - name: dns-udp
    protocol: UDP
    port: 53
    targetPort: 5353
  - name: dns-tcp
    protocol: TCP
    port: 53
    targetPort: 5353
```

Confirm the provider supports this mixed-protocol load balancer. Two Services can be needed on some infrastructures.

---

## 4. External Traffic Policy

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
```

| Value | Behavior |
|---|---|
| `Cluster` | Incoming traffic can be forwarded to ready endpoints anywhere in the cluster. This gives broad endpoint availability but can add a hop and can affect observed source address depending on implementation. |
| `Local` | A node handles external traffic only for endpoints local to that node. It commonly preserves the original source IP, but a node without a local ready endpoint cannot serve the application traffic. |

`Local` does not mean “send the client to a pod on the client's node,” and it is not automatically required for stateful applications. Use it when source-IP preservation or a specific traffic topology is required, then distribute ready pods and verify provider health checks.

Check the setting and pod placement:

```bash
oc get service/payments-lb -n payments \
  -o jsonpath='{.spec.externalTrafficPolicy}{"\n"}'
oc get pod -n payments -l app=payments -o wide
```

---

## 5. NodePort

A `NodePort` Service allocates a port from the configured node-port range. The usual default is 30000–32767, but a cluster administrator can configure another range. Query the assigned value rather than assuming it.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-nodeport
  namespace: payments
spec:
  type: NodePort
  selector:
    app: payments
  ports:
  - name: payments
    protocol: TCP
    port: 8443
    targetPort: 8443
```

```bash
oc apply -f payments-nodeport.yaml
oc get service/payments-nodeport -n payments -o wide
oc get service/payments-nodeport -n payments \
  -o jsonpath='{.spec.ports[0].nodePort}{"\n"}'
oc get nodes -o wide
```

An address shown by `oc get nodes -o wide` is not necessarily reachable from the external client. Firewalls, security groups, routing, node address selection, and platform policies still matter. Do not change a node firewall manually unless the task explicitly authorizes node-level changes and the cluster architecture requires them.

### Explicit NodePort

Use an explicit port only when the task requires it and the port is free and in the cluster's configured range:

```yaml
ports:
- name: payments
  port: 8443
  targetPort: 8443
  nodePort: 30443
```

Omitting `nodePort` lets the cluster allocate a valid free value and is normally safer.

---

## 6. MetalLB for a Home or Bare-Metal Lab

MetalLB provides `LoadBalancer` Service addresses on networks without a cloud load-balancer provider. The rough responsibility chain is:

1. A cluster administrator installs the supported MetalLB Operator.
2. An `IPAddressPool` defines addresses that the cluster is allowed to advertise.
3. An `L2Advertisement` or BGP configuration defines how those addresses are announced.
4. A `LoadBalancer` Service receives an address from the pool.

Example objects after the Operator and `MetalLB` instance are correctly installed:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.122.200-192.168.122.220
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - lab-pool
```

The address range must be unused, reachable on the local L2 network, and outside conflicting DHCP allocations. This is cluster/network administration; do not paste the example range into a different network.

Verify:

```bash
oc get ipaddresspool,l2advertisement -n metallb-system
oc get pods -n metallb-system
oc describe service/payments-lb -n payments
```

---

## 7. DNS, TLS, and Security Around the Service

A load balancer address does not automatically create public DNS. Create the required A/AAAA record for an IP or CNAME for an assigned provider hostname, following the DNS design.

Test resolution:

```bash
getent hosts payments.example.com
```

For TLS, the application behind an L4 Service normally terminates TLS itself, or the external load balancer terminates it according to provider configuration. A Kubernetes Service does not create an application certificate.

Before exposing a database or administration protocol, require authentication and TLS, restrict source networks at the provider/firewall, use NetworkPolicy, and consider private rather than internet-facing exposure. The existence of `type: LoadBalancer` does not make the application safe.

---

## 8. Temporary TCP Port Forwarding

```bash
oc port-forward service/payments 9443:8443 -n payments
```

In another terminal:

```bash
curl -vk https://127.0.0.1:9443/
```

Port-forwarding:

- handles TCP streams, not UDP;
- stops when the command exits or connectivity changes;
- binds locally (loopback by default); and
- is a diagnostic convenience, not persistent external access.

Multiple TCP ports can be forwarded in one command:

```bash
oc port-forward service/payments 9443:8443 9090:9090 -n payments
```

---

## 9. Troubleshooting in the Right Order

### `EXTERNAL-IP` stays `<pending>`

```bash
oc get service/<name> -n <project> -o yaml
oc describe service/<name> -n <project>
oc get events -n <project> --sort-by=.lastTimestamp
oc get infrastructure.config.openshift.io/cluster -o yaml
```

Check:

- whether the platform has a load-balancer implementation;
- cloud credentials/controller health on an integrated cloud;
- provider-specific Service annotations;
- address-pool availability and MetalLB health on bare metal; and
- quota or policy that blocks creation.

There is no generic `oc get cloudprovider` resource.

### Address assigned, but connection fails

First verify the backend:

```bash
oc get pod -n <project> -l app=<label> -o wide
oc get service/<name> -n <project> -o yaml
oc get endpointslice -n <project> \
  -l kubernetes.io/service-name=<name> -o yaml
oc get networkpolicy -n <project> -o yaml
```

Then test the correct protocol from a permitted external client:

```bash
nc -vz <external-address> <tcp-port>
openssl s_client -connect <external-address>:<tls-port> -servername <expected-name>
```

For UDP, use an application-aware client; `curl` and `nc -z` TCP tests do not prove UDP behavior.

### Service has no endpoints

```bash
oc get service/<name> -n <project> -o jsonpath='{.spec.selector}'
echo
oc get pod -n <project> --show-labels
oc describe pod/<pod> -n <project>
```

Fix label selection and readiness before examining the external load balancer.

### Wrong port mapping

```bash
oc get service/<name> -n <project> \
  -o custom-columns=PORT:.spec.ports[*].port,TARGET:.spec.ports[*].targetPort,NODEPORT:.spec.ports[*].nodePort
```

Confirm that the application actually listens on `targetPort`. `containerPort` in a pod spec is descriptive and useful for named ports, but it does not make the process listen.

### `externalTrafficPolicy: Local` works only on some nodes

```bash
oc get pod -n <project> -l app=<label> -o wide
oc get service/<name> -n <project> -o yaml
```

This is expected when traffic reaches a node without a local ready endpoint. Spread replicas appropriately, correct readiness, or use `Cluster` if source preservation is not required.

---

## 10. Hands-On Exam Lab

1. Deploy a TCP test server with two ready replicas and a Service selector.
2. Create a `LoadBalancer` Service that maps external port 443 to pod port 8443.
3. Prove the assigned IP or hostname and the ready EndpointSlices.
4. Change `externalTrafficPolicy` to `Local`; inspect endpoint placement and explain the effect.
5. Create a NodePort Service without specifying `nodePort`; report the assigned value.
6. Use TCP port-forwarding and prove local access.
7. Break the selector, diagnose the empty EndpointSlice, and repair it.
8. If the lab uses MetalLB, identify the pool that supplied the address. If it has no load-balancer provider, explain why `<pending>` is correct instead of inventing an IP.

---

## 11. Exam Traps

- A LoadBalancer Service is a request to infrastructure; it can remain pending without a provider.
- Status can contain a hostname, not only an IP.
- `externalTrafficPolicy: Local` means node-local endpoints, not “local to the client,” and not “required for stateful apps.”
- NodePort's usual range is configurable. Query the actual assigned port.
- Not every node address is externally reachable.
- `oc port-forward` is TCP-only and temporary.
- `ExternalName` is an internal DNS alias to an external name; it does not expose a cluster workload to outside clients.
- gRPC is HTTP/2 and can sometimes use a Route; classify the actual protocol/TLS behavior.
- A Service selector selects pods; `targetPort` selects the application port.
- Do not expose a database merely because a LoadBalancer manifest is easy to write.

---

## 12. Quick Reference

| Task | Command |
|---|---|
| List Services | `oc get service -n NS -o wide` |
| Show provider status/events | `oc describe service/NAME -n NS` |
| Show assigned address | `oc get service/NAME -n NS -o jsonpath='{range .status.loadBalancer.ingress[*]}{.ip}{.hostname}{"\n"}{end}'` |
| Show endpoints | `oc get endpointslice -n NS -l kubernetes.io/service-name=NAME` |
| Change type | `oc patch service/NAME -n NS --type=merge -p '{"spec":{"type":"LoadBalancer"}}'` |
| Show NodePort | `oc get service/NAME -n NS -o jsonpath='{.spec.ports[*].nodePort}'` |
| Temporary TCP access | `oc port-forward service/NAME LOCAL:SERVICEPORT -n NS` |
| Show platform | `oc get infrastructure.config.openshift.io/cluster -o yaml` |

---

## 13. Review Questions and Answers

1. **Why can a LoadBalancer Service remain pending?** No provider has fulfilled the request, or provider/address-pool configuration failed.
2. **Can a standard Route expose UDP?** No. Use a Service-based L4 method.
3. **Can a passthrough Route expose a non-HTTP TLS protocol?** Yes when the client sends SNI and the Ingress Controller supports the flow.
4. **What is the difference between `port` and `targetPort`?** `port` is offered by the Service; `targetPort` is the backend pod port/name.
5. **What proves the Service selected a ready backend?** Ready addresses in its EndpointSlice objects.
6. **Why might `Local` drop traffic on a node?** That node has no local ready endpoint.
7. **Does port-forward persist after logout/reboot?** No.
8. **What supplies on-premises LoadBalancer addresses in a common OpenShift design?** A configured MetalLB installation and address pool.

---

## Verified References

- EX280 objectives: <https://www.redhat.com/en/services/training/red-hat-certified-openshift-administrator-exam>
- OpenShift 4.18 ingress and load balancing: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/ingress_and_load_balancing/index>
- Routes and non-HTTP/SNI alternatives: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/ingress_and_load_balancing/configuring-routes>
- OpenShift 4.18 MetalLB Operator: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/networking_operators/metallb-operator>
