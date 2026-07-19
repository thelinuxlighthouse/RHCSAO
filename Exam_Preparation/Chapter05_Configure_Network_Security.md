# Chapter 5: Configure Network Security

> **Exam baseline:** EX280 on OpenShift Container Platform 4.18.  
> **Reviewed:** 2026-07-19.  
> OpenShift 4.18 uses OVN-Kubernetes. OpenShift SDN was removed, and OVN-Kubernetes uses **Geneve**, not VXLAN, for its default overlay.

## What You Must Be Able to Do

By the end of this chapter, you should be able to:

- inspect the installed cluster, service, and machine network configuration;
- verify the Cluster Network Operator and OVN-Kubernetes workloads;
- troubleshoot DNS, Service, EndpointSlice, Route, and pod connectivity in layers;
- create and edit Routes with custom host, path, and TLS termination;
- create default-deny and allow-list NetworkPolicies;
- allow ingress-controller traffic through a default-deny policy;
- use service-serving certificates for internal TLS; and
- distinguish certificate, routing, readiness, and policy failures.

---

## 1. The Networks in an OpenShift Cluster

Do not treat “the network” as one thing.

| Network | Purpose |
|---|---|
| Machine network | Addresses of cluster nodes and infrastructure interfaces. |
| Cluster/pod network | Addresses assigned to pods. Usually not routed directly outside the cluster. |
| Service network | Virtual ClusterIP addresses used by Services. |
| Ingress network path | External clients reach routers/Ingress Controllers, which proxy to Service endpoints. |
| Secondary networks | Optional additional pod/VM interfaces created through Multus and another CNI. |

View the actual installed values instead of memorizing example CIDRs:

```bash
oc get network.config.openshift.io/cluster -o yaml
oc get network.operator.openshift.io/cluster -o yaml
```

Useful status fields include `status.networkType`, `status.clusterNetwork`, and `status.serviceNetwork`. Values are chosen for the cluster and can differ from classroom examples.

### OVN-Kubernetes

OVN-Kubernetes is the default network provider in OCP 4.18. It uses Open Virtual Network (OVN) and Open vSwitch (OVS) to implement logical switching, routing, Services, and NetworkPolicy. The default inter-node overlay uses Geneve encapsulation.

Verify the supported components:

```bash
oc get clusteroperator/network
oc get pods -n openshift-ovn-kubernetes -o wide
oc get daemonset -n openshift-ovn-kubernetes
oc get events -n openshift-ovn-kubernetes --sort-by=.lastTimestamp
```

Do not look for an `ovs` ClusterOperator or `openshift-sdn` namespace on 4.18. The cluster Operator is named `network`.

Changing the default CNI is not a routine `oc edit` exercise. Network migration and CIDR changes have strict, version-specific procedures and can disrupt the cluster. For EX280, inspect and troubleshoot the installed configuration unless a task explicitly supplies a supported change procedure.

---

## 2. Service and Route Traffic

### Service traffic

A Service uses a selector to find pods. EndpointSlice objects contain ready backend addresses.

```text
client pod -> Service DNS name -> ClusterIP -> EndpointSlice -> ready pod IP:targetPort
```

Important fields:

- `Service.spec.port`: port offered by the Service;
- `Service.spec.targetPort`: named or numeric port on the pod;
- `Service.spec.selector`: labels that must match the pods; and
- `EndpointSlice.endpoints.conditions.ready`: whether the endpoint is ready.

Verify all layers:

```bash
oc get service/<service> -n <project> -o yaml
oc get pod -n <project> --show-labels
oc get endpointslice -n <project> \
  -l kubernetes.io/service-name=<service> -o yaml
```

No endpoints usually means selector mismatch, no matching pods, or failing readiness—not an Ingress Controller problem.

### Route traffic

```text
external DNS -> load balancer/node -> Ingress Controller (router)
             -> Route host/path match -> Service -> ready endpoint -> pod
```

The Ingress Operator manages `IngressController` resources and router deployments:

```bash
oc get clusteroperator/ingress
oc get ingresscontroller/default -n openshift-ingress-operator -o yaml
oc get deployment,pod -n openshift-ingress -o wide
```

Router pod count and placement depend on `IngressController` configuration. Routers do not necessarily run on every node.

---

## 3. Create and Edit Routes

### Unsecured HTTP Route

```bash
oc expose service/web -n apps
oc get route/web -n apps
```

Or use an explicit manifest:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: web
  namespace: apps
spec:
  host: web.apps.example.com
  to:
    kind: Service
    name: web
    weight: 100
  port:
    targetPort: http
```

The Service must have a port named `http`, or `targetPort` must use the correct numeric Service port.

### Edge TLS

At edge termination, the router presents the external certificate and sends HTTP to the backend.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: web-edge
  namespace: apps
spec:
  host: web.apps.example.com
  to:
    kind: Service
    name: web
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

If `certificate` and `key` are absent, the router's default certificate is used. Its certificate names must match the requested host or clients report a hostname error.

Create the same pattern with the route generator:

```bash
oc create route edge web-edge \
  --service=web \
  --hostname=web.apps.example.com \
  --port=http \
  --insecure-policy=Redirect \
  -n apps
```

### Edge Route with a Custom Certificate

The Route API field names are `certificate`, `key`, and optionally `caCertificate`:

```yaml
spec:
  host: web.apps.example.com
  to:
    kind: Service
    name: web
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
    certificate: |
      -----BEGIN CERTIFICATE-----
      <server certificate followed by any needed intermediates>
      -----END CERTIFICATE-----
    key: |
      -----BEGIN PRIVATE KEY-----
      <matching private key>
      -----END PRIVATE KEY-----
    caCertificate: |
      -----BEGIN CERTIFICATE-----
      <optional CA chain>
      -----END CERTIFICATE-----
```

Do not use invented fields such as `tlsCertificate` or `tlsPrivateKey` in a Route.

### Passthrough TLS

The router selects the Route from TLS Server Name Indication (SNI) and passes encrypted bytes to the backend. The pod presents the certificate.

```bash
oc create route passthrough web-pass \
  --service=web-tls \
  --hostname=web.apps.example.com \
  --port=https \
  -n apps
```

The Route has no certificate/key because the router does not terminate TLS. Passthrough requires a TLS client that sends SNI.

### Re-encrypt TLS

The router terminates the client TLS connection and opens a new TLS connection to the backend. `destinationCACertificate` tells the router which CA to trust for the backend certificate.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: web-reencrypt
  namespace: apps
spec:
  host: web.apps.example.com
  to:
    kind: Service
    name: web-tls
  port:
    targetPort: https
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: Redirect
    destinationCACertificate: |
      -----BEGIN CERTIFICATE-----
      <CA that signed the service certificate>
      -----END CERTIFICATE-----
```

The external side uses the router default certificate unless `certificate` and `key` are added.

### Path Route

Path matching requires `spec.path`:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: api
  namespace: apps
spec:
  host: portal.apps.example.com
  path: /api
  to:
    kind: Service
    name: api
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

### Wildcard Route

For `wildcardPolicy: Subdomain`, set a base host such as `apps.example.com`. The route can admit subdomains beneath that host, subject to Ingress Controller policy and route ownership rules:

```yaml
spec:
  host: apps.example.com
  wildcardPolicy: Subdomain
  to:
    kind: Service
    name: wildcard-backend
```

Do not put `*.example.com` in `spec.host` and assume it is valid.

### Route verification

```bash
oc get route/<route> -n <project> -o yaml
oc describe route/<route> -n <project>
HOST=$(oc get route/<route> -n <project> -o jsonpath='{.spec.host}')
curl -vk "https://$HOST/"
```

Inspect `status.ingress[*].conditions`. An admitted Route normally has an `Admitted=True` condition.

---

## 4. NetworkPolicy Mental Model

By default, a pod is non-isolated for ingress and egress. A pod becomes isolated for a direction when at least one NetworkPolicy selects it for that direction. Allowed traffic is the **union** of all applicable policies. Policies add allowed paths; their order does not matter, and one allow policy does not override another deny policy because there are no explicit deny rules in the standard API.

For a packet to pass when both ends are isolated:

```text
source pod egress must allow it
                  AND
destination pod ingress must allow it
```

NetworkPolicy is namespaced and selects pods only in its own namespace. Peer selectors identify sources/destinations.

### Default-deny ingress and egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: apps
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

No `ingress` or `egress` allow rules means none are allowed for selected pods in those directions. This policy can also block DNS and traffic from the OpenShift router, so add the required allow policies.

### Allow same-project traffic to one application

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-project-web
  namespace: apps
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 8080
```

Within a peer, a `podSelector` by itself selects pods in the policy's namespace.

### Allow a client namespace

This policy belongs in the **destination** namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-clients
  namespace: apps
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: clients
      podSelector:
        matchLabels:
          role: approved-client
    ports:
    - protocol: TCP
      port: 8080
```

Because `namespaceSelector` and `podSelector` are in the **same** `from` item, both must match. If written as two list items, they mean either selected namespaces or selected local pods—usually a much wider rule.

### Allow DNS egress

After default-deny egress, allow DNS to the cluster DNS pods. The exact labels are cluster-managed, so inspect them before writing the policy:

```bash
oc get namespace openshift-dns --show-labels
oc get pod -n openshift-dns --show-labels
```

Example using stable namespace name label plus observed pod label:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: apps
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-dns
      podSelector:
        matchLabels:
          dns.operator.openshift.io/daemonset-dns: default
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

Confirm the DNS pod label on the target cluster; do not memorize it as universal.

### Allow traffic from OpenShift Ingress

With default-deny ingress, Routes can return 503 because router-to-pod traffic is blocked. OpenShift labels the ingress namespace for policy selection:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
  namespace: apps
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          policy-group.network.openshift.io/ingress: ""
    ports:
    - protocol: TCP
      port: 8080
```

Use the backend pod port, not the external Route port.

### NetworkPolicy limits

- It does not filter traffic for pods using `hostNetwork` in the same way as ordinary pod-network traffic.
- It is primarily L3/L4 policy: IP/peer, protocol, and port—not HTTP URL or user identity.
- It does not make TLS unnecessary.
- It cannot fix a Service selector, readiness, DNS, or application listening error.

---

## 5. Internal TLS with the OpenShift Service CA

OpenShift's service CA can sign a certificate for a Service DNS name and place it in a Secret.

Annotate the Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: apps
  annotations:
    service.beta.openshift.io/serving-cert-secret-name: api-serving-cert
spec:
  selector:
    app: api
  ports:
  - name: https
    port: 8443
    targetPort: 8443
```

Apply it and wait for the Secret:

```bash
oc apply -f service.yaml
oc get secret/api-serving-cert -n apps
oc get secret/api-serving-cert -n apps -o jsonpath='{.type}'
echo
```

The Secret contains `tls.crt` and `tls.key`. Mount it into the server pod and configure the application to listen with TLS. The certificate is normally valid for service DNS names; it is not automatically a public Route certificate.

Inject the service CA bundle into a ConfigMap for internal clients:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: service-ca
  namespace: clients
  annotations:
    service.beta.openshift.io/inject-cabundle: "true"
```

The controller writes a `service-ca.crt` key. Mount that CA into clients and configure them to trust it. The injection controller owns that key; do not hand-edit it.

For a re-encrypt Route to a service-serving certificate, provide the service CA as the Route's `destinationCACertificate`, or use the supported service-CA integration behavior available on the cluster and verify the route status.

---

## 6. Layered Troubleshooting

### Step 1: Prove pod health and listening port

```bash
oc get pod -n <project> -o wide
oc describe pod/<pod> -n <project>
oc logs pod/<pod> -n <project> --all-containers
```

`connection refused` often means the destination was reached but nothing is listening on that address/port. A timeout more often suggests policy, routing, or a dropped packet, although applications can also time out.

### Step 2: Prove Service selection and endpoints

```bash
oc get service/<service> -n <project> -o yaml
oc get pod -n <project> --show-labels
oc get endpointslice -n <project> \
  -l kubernetes.io/service-name=<service> -o wide
```

### Step 3: Prove DNS and in-cluster access

From an existing diagnostic/client pod that contains the needed tools:

```bash
oc exec -n <project> <client-pod> -- \
  getent hosts <service>.<project>.svc.cluster.local

oc exec -n <project> <client-pod> -- \
  curl -sv http://<service>:<port>/
```

If the image lacks `getent` or `curl`, use an approved diagnostic image available in the lab; do not interpret “command not found” as a network failure.

### Step 4: Inspect policies on both ends

```bash
oc get networkpolicy -n <source-project> -o yaml
oc get networkpolicy -n <destination-project> -o yaml
oc get namespace --show-labels
oc get pod -n <source-project> --show-labels
oc get pod -n <destination-project> --show-labels
```

Check the selected pods, both directions, namespace labels, list indentation, and ports.

### Step 5: Prove Route and router health

```bash
oc get route/<route> -n <project> -o yaml
oc get clusteroperator/ingress
oc get pod -n openshift-ingress
oc logs deployment/router-default -n openshift-ingress --all-containers
```

Common HTTP results:

| Result | Likely area |
|---|---|
| DNS name does not resolve | External/wildcard DNS. |
| TLS hostname/unknown-CA error | Route/default certificate or client trust. |
| `503` from router | Route admitted but no usable backend, wrong Service/port, readiness, or NetworkPolicy. |
| `404` from router | No matching admitted host/path Route. |
| Application's own `404` | Routing worked; application path is wrong. |

### Step 6: Check cluster network only after application layers

```bash
oc get clusteroperator/network -o yaml
oc get pods -n openshift-ovn-kubernetes -o wide
oc get events -n openshift-ovn-kubernetes --sort-by=.lastTimestamp
```

Direct OVS/OVN database commands are advanced, version-dependent, and usually not the first EX280 step. Use `oc debug node/<node>` only when the task permits node debugging and higher-level evidence points to a node problem.

---

## 7. Hands-On Exam Lab

1. Create projects `net-client` and `net-app`.
2. Deploy a two-replica HTTP application in `net-app`, create a Service with a named port, and prove EndpointSlices are ready.
3. Create an edge Route with a custom host and redirect HTTP to HTTPS.
4. Add default-deny ingress and egress in `net-app`.
5. Add only the rules needed for DNS, the approved client pod, and OpenShift ingress.
6. Prove an approved client can connect and an unapproved client cannot.
7. Annotate an internal Service for a service-serving certificate and verify the generated Secret.
8. Deliberately break the Service selector, diagnose the empty endpoints, and restore it.
9. Deliberately break the Route `targetPort`, diagnose the 503, and restore it.

Record the exact commands and evidence for each result.

---

## 8. Exam Traps

- The OCP 4.18 ClusterOperator is `network`, not `ovs`.
- The OVN namespace is `openshift-ovn-kubernetes`, not `openshift-sdn`.
- OVN-Kubernetes uses Geneve, not VXLAN, for the default overlay.
- NetworkPolicies isolate selected pods by direction; it is not simply “a namespace has a policy, so the namespace is isolated.”
- Allow policies are additive. There is no ordered last-rule-wins behavior.
- A peer with namespace and pod selectors in one item means AND; separate items mean OR.
- Default-deny can block DNS and router traffic.
- A Route path requires `spec.path`.
- A wildcard Route uses a normal base host plus `wildcardPolicy: Subdomain`, not `host: "*.example.com"`.
- Custom Route TLS fields are `certificate` and `key`.
- Router pods do not have to run on every node.
- Never memorize example network CIDRs as cluster defaults; query the cluster.

---

## 9. Quick Reference

| Task | Command |
|---|---|
| View network configuration | `oc get network.config.openshift.io/cluster -o yaml` |
| Check network Operator | `oc get clusteroperator/network` |
| Check OVN pods | `oc get pods -n openshift-ovn-kubernetes -o wide` |
| Check ingress Operator | `oc get clusteroperator/ingress` |
| Inspect default IngressController | `oc get ingresscontroller/default -n openshift-ingress-operator -o yaml` |
| Create edge Route | `oc create route edge NAME --service=SVC --hostname=HOST --port=PORT --insecure-policy=Redirect` |
| Create passthrough Route | `oc create route passthrough NAME --service=SVC --hostname=HOST --port=PORT` |
| Show Route admission | `oc get route/NAME -o yaml` |
| Show backend endpoints | `oc get endpointslice -l kubernetes.io/service-name=SVC` |
| List policies | `oc get networkpolicy -n NS -o yaml` |
| Apply policy | `oc apply -f policy.yaml` |

---

## 10. Review Questions and Answers

1. **Which encapsulation does OVN-Kubernetes use for the default overlay?** Geneve.
2. **When is a pod isolated for ingress?** When at least one NetworkPolicy selecting that pod includes ingress policy behavior.
3. **How are several allow policies combined?** By union; traffic allowed by any applicable rule is allowed, subject also to the source's egress policy.
4. **Why can a Route return 503 after default deny?** The router may be unable to reach selected backend pods, or the Service has no ready endpoints/wrong target port.
5. **Where does a cross-namespace ingress policy live?** In the destination pod's namespace.
6. **What does passthrough require from the client?** TLS with SNI so the router can choose a Route without decrypting the stream.
7. **What signs an internal service-serving certificate?** The OpenShift service CA controller after the Service annotation requests a Secret.
8. **What is the first check for a Service with no traffic?** Inspect its selector, matching pod labels, readiness, ports, and EndpointSlices.

---

## Verified References

- EX280 objectives: <https://www.redhat.com/en/services/training/red-hat-certified-openshift-administrator-exam>
- OVN-Kubernetes architecture and Geneve: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/ovn-kubernetes_network_plugin/about-ovn-kubernetes>
- OpenShift 4.18 network security and NetworkPolicy: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/network_security/index>
- Routes, Ingress, and load balancing: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/ingress_and_load_balancing/configuring-routes>
- Service-serving certificates: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/security_and_compliance/configuring-certificates>
