# Chapter 7: Enable Developer Self-Service

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure cluster-wide resource quotas
- Configure project (namespace) quotas to limit resource consumption
- Set resource requirements for projects
- Create and manage LimitRanges to define resource bounds
- Configure and use project templates for standardized application deployment
- Understand the relationship between quotas, limit ranges, and resource requests/limits
- Enable developers to self-provision resources within defined constraints
- Troubleshoot quota and resource limitation issues
- Apply developer self-service best practices in production environments

---

## OpenShift Concepts

### Resource Quotas

Resource Quotas are Kubernetes/OpenShift resources that limit the aggregate resource consumption within a namespace (project). They prevent users from consuming excessive cluster resources and ensure fair resource allocation across the cluster.

**Cluster Resource Quotas** apply across the entire cluster, limiting total pod count, total CPU, total memory, and other resources cluster-wide.

**Project Resource Quotas** apply within individual namespaces, limiting what developers can consume in their project.

Quotas enforce limits at the namespace level, checking against the sum of all resource requests in that namespace.

### LimitRanges

LimitRanges define minimum and maximum constraints for individual resources within a namespace. Unlike Quotas, which limit aggregate consumption, LimitRanges constrain each resource independently.

LimitRanges validate:
- Container CPU requests and limits
- Container memory requests and limits
- PersistentVolume requests
- Service account tokens
- ConfigMap and Secret data size

Every namespace can have at most one LimitRange, and it applies to all resources in that namespace.

### Resource Requests and Limits

Understanding the difference between requests and limits is crucial for resource management:

- **Requests**: The minimum guaranteed resources allocated to a container. The scheduler uses requests to place pods on nodes.
- **Limits**: The maximum resources a container can consume. If exceeded, the container is throttled (CPU) or killed (memory).

**Best Practice**: Set requests to your baseline usage and limits to your peak usage.

### Project Templates

Project Templates are parameterized YAML manifests that allow developers to deploy standardized applications quickly. Templates support:

- **Parameters**: Configurable values with validation
- **Built-in parameters**: Image, replicas, ports, storage
- **Parameter substitution**: `${PARAM_NAME}` syntax
- **Parameter types**: String, boolean, integer, image, port, disk, route, secret

Templates enable consistent application deployment patterns across the organization.

### Developer Self-Service Model

The developer self-service model in OpenShift empowers developers to:

1. **Create projects**: Automatically provisioned with appropriate quotas and limit ranges
2. **Deploy applications**: Use templates or imperative commands
3. **Manage resources**: Within defined quota and limit constraints
4. **Monitor usage**: View resource consumption through the web console

This model reduces administrative overhead while maintaining resource governance.

---

## Architecture and Internal Operation

### Resource Quota Enforcement Flow

```
Developer creates pod with resource requests
        │
        v
Scheduler checks namespace quota
        │
        ├── Quota has capacity → Schedule pod
        │
        └── Quota exhausted → Reject with "exceeded quota"
```

When a pod is created, the API server checks if the namespace has available quota for the requested resources. If quota is exceeded, the pod creation fails with a QuotaExceededError.

### LimitRange Validation Flow

```
Pod created in namespace with LimitRange
        │
        v
API server validates against LimitRange
        │
        ├── Min requests met → Accept
        │
        ├── Max limits exceeded → Reject
        │
        └── Valid → Create pod
```

LimitRange validation occurs at pod creation time. The API server checks each container's requests and limits against the LimitRange constraints.

### Quota and LimitRange Interaction

```
┌─────────────────────────────────────────────────────────┐
│                    Namespace                            │
├─────────────────────────────────────────────────────────┤
│  Quota:    Total CPU: 10 cores, Total Memory: 32GB      │
│  LimitRange: Min CPU: 100m, Max CPU: 2 cores            │
│                Min Memory: 64Mi, Max Memory: 4Gi        │
├─────────────────────────────────────────────────────────┤
│  Pod 1: CPU request=500m, Memory request=256Mi ✓        │
│  Pod 2: CPU request=1 core, Memory request=1Gi ✓        │
│  Pod 3: CPU request=4 cores (fails quota) ✗             │
│  Pod 4: CPU request=50m (fails LimitRange) ✗            │
└─────────────────────────────────────────────────────────┘
```

Both Quota and LimitRange must be satisfied for pod creation to succeed.

### Developer Project Provisioning

```
Developer runs: oc new-project myapp
        │
        v
OpenShift creates project with:
        │
        ├── Default resource quota (cluster-admin defined)
        ├── LimitRange (cluster-wide defaults)
        ├── Service account
        └── Project role bindings
```

Cluster administrators can configure default quotas and limit ranges that apply to all new projects.

---

## Components and Terminology

| Term | Description |
|------|-------------|
| ResourceQuota | Kubernetes resource limiting aggregate namespace consumption |
| LimitRange | Kubernetes resource defining min/max constraints per resource |
| Resource Request | Minimum guaranteed resources for a container |
| Resource Limit | Maximum resources a container can consume |
| QuotaExceededError | Error when namespace quota is exceeded |
| Project Template | Parameterized application blueprint |
| Parameter | Configurable value in a template |
| Template Parameter | String, integer, boolean, image, port, etc. |
| Cluster Quota | Quota applied across entire cluster |
| Project Quota | Quota applied to single namespace |
| `oc adm quota` | OpenShift CLI for managing cluster quotas |
| `oc adm limitrange` | OpenShift CLI for managing limit ranges |
| `oc adm template` | OpenShift CLI for template management |
| Default Quota | Quota automatically applied to new projects |
| Hard Limit | Maximum value enforced by LimitRange |
| Soft Limit | Warning threshold (not enforced) |

---

## Administration Tasks

### Configuring Cluster Resource Quotas

#### Creating a Cluster-Wide Resource Quota

```bash
oc create quota cluster-quota --hard=cpu=50,memory=200Gi,pods=1000 -n cluster
```

Note: Cluster-level quotas are managed through the `oc adm` commands.

#### Viewing Cluster Quotas

```bash
oc get quotas
oc get quotas -A
```

#### Editing Cluster Quotas

```bash
oc edit quota cluster-quota
```

#### Creating Quota via YAML

```bash
oc apply -f cluster-quota.yaml
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cluster-quota
  namespace: cluster
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 200Gi
    limits.cpu: "100"
    limits.memory: 400Gi
    pods: "1000"
```

### Configuring Project Quotas

#### Creating a Project Quota

```bash
oc create quota myapp-quota --hard=cpu=4,memory=16Gi,pods=20 -n myapp-project
```

#### Creating Quota with Multiple Resources

```bash
oc create quota myapp-quota \
  --hard=cpu=4 \
  --hard=memory=16Gi \
  --hard=pods=20 \
  --hard=requests.storage=100Gi \
  --hard=services=10 \
  --hard=configmaps=20 \
  --hard=secrets=20 \
  --hard=replicationcontrollers=5 \
  --hard=deployments=10 \
  --hard=replicasets=10 \
  --hard=secretsecrets=20
```

#### Creating Quota via YAML

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: myapp-quota
  namespace: myapp-project
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 16Gi
    limits.cpu: "8"
    limits.memory: 32Gi
    pods: "20"
    services: "10"
    configmaps: "20"
    secrets: "20"
    persistentvolumeclaims: "10"
    deployments: "10"
    replicasets: "10"
    replicationcontrollers: "5"
    persistentvolumeclaims: "10"
    secrets: "20"
```

#### Setting Default Quota for New Projects

```bash
# Edit the cluster-wide default quota
oc edit quota default -n cluster
```

Or set via cluster configuration:

```bash
oc edit cluster
```

### Configuring Project Resource Requirements

#### Setting Default Resource Requirements

```bash
# Set default resource limits for new deployments
oc adm policy set-default-limits \
  --requests-cpu=100m \
  --requests-memory=128Mi \
  --limits-cpu=500m \
  --limits-memory=512Mi
```

#### Viewing Default Limits

```bash
oc get limitranges -n myapp-project
```

### Configuring Project LimitRanges

#### Creating a LimitRange

```bash
oc create limitrange myapp-limits \
  --default=cpu=100m,memory=128Mi \
  --default-request=cpu=100m,memory=128Mi \
  --default-max=cpu=2,memory=4Gi \
  --max=cpu=4,memory=8Gi
```

#### Creating LimitRange with Multiple Types

```bash
oc create limitrange myapp-limits \
  --default=cpu=100m,memory=128Mi \
  --default-request=cpu=100m,memory=128Mi \
  --default-max=cpu=2,memory=4Gi \
  --min=cpu=50m,memory=64Mi \
  --max=cpu=4,memory=8Gi \
  --max-limit=cpu=4,memory=8Gi \
  --max-pvc-storage=10Gi
```

#### Creating LimitRange via YAML

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: myapp-limits
  namespace: myapp-project
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: 2
        memory: 4Gi
      min:
        cpu: 50m
        memory: 64Mi
    - type: Pod
      max:
        cpu: 4
        memory: 8Gi
    - type: PersistentVolumeClaim
      max:
        storage: 10Gi
```

#### Editing LimitRange

```bash
oc edit limitrange myapp-limits -n myapp-project
```

### Configuring Project Templates

#### Creating a Basic Template

```bash
oc apply -f template.yaml
```

#### Creating a Template from a Deployment

```bash
# Export current deployment as template
oc apply -f template.yaml
```

#### Instantiating a Template

```bash
oc process -f template.yaml -p PARAM=value
oc process -f template.yaml | oc apply -f -
```

#### Creating a Template via CLI

```bash
oc new-app nginx --template=my-template
```

---

## YAML Examples

### Cluster Resource Quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cluster-quota
  namespace: cluster
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 200Gi
    limits.cpu: "100"
    limits.memory: 400Gi
    pods: "1000"
    services: "100"
    configmaps: "500"
    secrets: "500"
    persistentvolumeclaims: "200"
    replicationcontrollers: "100"
    deployments: "200"
    replicasets: "200"
```

### Project Resource Quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: myapp-quota
  namespace: myapp-project
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 16Gi
    limits.cpu: "8"
    limits.memory: 32Gi
    pods: "20"
    services: "10"
    configmaps: "20"
    secrets: "20"
    persistentvolumeclaims: "10"
    replicationcontrollers: "5"
    deployments: "10"
    replicasets: "10"
    horizontalpodautoscalers: "5"
    networkpolicies: "20"
```

### LimitRange - Container Defaults

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: myapp-project
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: 2
        memory: 4Gi
      min:
        cpu: 50m
        memory: 64Mi
```

### LimitRange - Pod and PVC

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limits
  namespace: myapp-project
spec:
  limits:
    - type: Pod
      max:
        cpu: 4
        memory: 8Gi
    - type: PersistentVolumeClaim
      max:
        storage: 10Gi
      min:
        storage: 1Gi
```

### Project Template

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: webapp-template
  annotations:
    description: "A web application template with database"
    iconClass: "icon-nginx"
    tags: "template,web,example,database"
objects:
  - apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: ${SERVICE_ACCOUNT_NAME}
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ${APP_NAME}
      labels:
        app: ${APP_NAME}
    spec:
      replicas: ${REPLICAS}
      selector:
        matchLabels:
          app: ${APP_NAME}
      template:
        metadata:
          labels:
            app: ${APP_NAME}
        spec:
          serviceAccountName: ${SERVICE_ACCOUNT_NAME}
          containers:
            - name: ${APP_NAME}
              image: "${IMAGE}:${IMAGE_TAG}"
              ports:
                - containerPort: ${PORT}
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
      name: ${APP_NAME}-svc
    spec:
      selector:
        app: ${APP_NAME}
      ports:
        - port: ${SERVICE_PORT}
          targetPort: ${PORT}
  - apiVersion: route.openshift.io/v1
    kind: Route
    metadata:
      name: ${APP_NAME}-route
    spec:
      to:
        kind: Service
        name: ${APP_NAME}-svc
      port:
        targetPort: ${SERVICE_PORT}
      tls:
        termination: edge
        insecureEdgeTerminationPolicy: Redirect
parameters:
  - name: APP_NAME
    description: "Application name"
    required: true
    value: myapp
  - name: REPLICAS
    description: "Number of replicas"
    required: true
    value: "2"
  - name: IMAGE
    description: "Container image"
    required: true
    value: registry.redhat.io/ubi9/nginx-124
  - name: IMAGE_TAG
    description: "Image tag"
    required: true
    value: "latest"
  - name: PORT
    description: "Container port"
    required: true
    value: "8080"
  - name: SERVICE_PORT
    description: "Service port"
    required: true
    value: "80"
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
  - name: SERVICE_ACCOUNT_NAME
    description: "Service account name"
    required: true
    value: default
```

---

## Explanation of YAML

### ResourceQuota Structure

The ResourceQuota has three key sections:

- **`metadata.name`**: Must be unique within the namespace
- **`spec.hard`**: Defines the maximum aggregate resources allowed
- **Resource types**: `requests.cpu`, `requests.memory`, `limits.cpu`, `limits.memory`, `pods`, `services`, `configmaps`, `secrets`, `persistentvolumeclaims`, `deployments`, `replicasets`, `replicationcontrollers`, `horizontalpodautoscalers`, `networkpolicies`

Values can be specified with units: `100m` (CPU millicores), `1Gi` (gibibytes), `20` (count).

### LimitRange Structure

The LimitRange uses a different structure:

- **`spec.limits`**: Array of limit definitions
- **`type`**: Container, Pod, PersistentVolumeClaim, ConfigMap, Secret, ServiceAccountToken
- **`default`**: Default values if not specified in pod spec
- **`defaultRequest`**: Default request values
- **`min`**: Minimum allowed values (enforced)
- **`max`**: Maximum allowed values (enforced)
- **`maxLimit`**: Maximum limit value (prevents limit < request)

### Template Structure

Templates have:

- **`metadata.annotations`**: Descriptive metadata for discovery
- **`objects`**: Array of Kubernetes resources
- **`parameters`**: Configurable values
- **Parameter validation**: `required: true`, `from` (regex), `generate` (expression)

---

## Verification Procedures

### Verify Resource Quota

```bash
# Check quota exists
oc get quota myapp-quota -n myapp-project

# Check quota usage
oc describe quota myapp-quota -n myapp-project

# Check hard limits
oc get quota myapp-quota -o jsonpath='{.spec.hard}'
```

Expected: Quota shows available and used resources.

### Verify LimitRange

```bash
oc get limitrange myapp-limits -n myapp-project
oc describe limitrange myapp-limits -n myapp-project
```

Expected: LimitRange shows default, min, and max values.

### Verify Quota Enforcement

```bash
# Try to create pod exceeding quota
oc create deployment test-quota --replicas=100 --image=nginx

# Should fail with "exceeded quota"
```

Expected: Pod creation fails with quota exceeded error.

### Verify LimitRange Enforcement

```bash
# Try to create pod with invalid resources
oc create deployment test-limits --image=nginx --requests=cpu=50m

# Should fail with "must specify a request cpu value greater than or equal to 50m"
```

Expected: Pod creation fails with LimitRange validation error.

### Verify Template Instantiation

```bash
oc process -f template.yaml -p APP_NAME=myapp | oc apply -f -
oc get all -l app=myapp
```

Expected: All template objects are created with substituted parameters.

### Verify Resource Usage

```bash
# Check resource usage in namespace
oc describe quota myapp-quota -n myapp-project

# Check pod resource usage
oc top pods -n myapp-project
```

Expected: Quota shows usage, and pods show resource consumption.

---

## Troubleshooting

### Pod Creation Fails with "exceeded quota"

**Symptom**: `Error creating: pods "test-pod" is forbidden: exceeded quota`

**Diagnosis**:

```bash
oc get quota -n myapp-project
oc describe quota -n myapp-project
oc get pods -n myapp-project -o wide
```

**Resolution**:
- Check current quota usage vs. hard limits
- Delete unused resources to free quota
- Increase quota limits if appropriate
- Reduce resource requests in deployment

### Pod Creation Fails with "must specify" Error

**Symptom**: `Error creating: pods "test-pod" is forbidden: must specify a request cpu value greater than or equal to 50m`

**Diagnosis**:

```bash
oc get limitrange -n myapp-project
oc describe limitrange -n myapp-project
```

**Resolution**:
- Check LimitRange min values
- Update deployment with valid resource requests
- Adjust LimitRange if constraints are too restrictive

### Template Parameter Validation Fails

**Symptom**: Template processing fails with parameter validation errors.

**Diagnosis**:

```bash
oc process -f template.yaml 2>&1
oc describe template my-template
```

**Resolution**:
- Check required parameters are provided
- Verify parameter values match `from` regex patterns
- Ensure values are within expected ranges

### Quota Not Enforcing Properly

**Symptom**: Quota appears to not be enforced.

**Diagnosis**:

```bash
oc get quota -n myapp-project
oc describe quota -n myapp-project
oc get limitrange -n myapp-project
```

**Resolution**:
- Verify quota is in correct namespace
- Check quota name matches
- Ensure namespace has at least one resource to track
- Review quota spec for typos

### LimitRange Default Not Applied

**Symptom**: Pod creation succeeds without resource specifications despite LimitRange.

**Diagnosis**:

```bash
oc get limitrange -n myapp-project -o yaml
oc get pods -n myapp-project -o yaml
```

**Resolution**:
- LimitRange defaults only apply to new pods
- Existing pods cannot be retroactively updated
- Restart pods to apply defaults

---

## Production Best Practices

### Resource Quota Best Practices

- **Set realistic quotas**: Base quotas on actual resource usage patterns
- **Monitor quota usage**: Regularly review quota consumption
- **Use tiered quotas**: Different quotas for dev, staging, production
- **Set appropriate limits**: Don't set quotas too low to cause constant rejections
- **Document quotas**: Maintain documentation of quota policies

### LimitRange Best Practices

- **Enforce minimums**: Prevent undersized containers
- **Set reasonable maximums**: Prevent resource monopolization
- **Use sensible defaults**: Help developers get started quickly
- **Review periodically**: Adjust based on application needs

### Template Best Practices

- **Use parameter validation**: Enforce required fields and value ranges
- **Document templates**: Clear descriptions and usage examples
- **Use defaults**: Provide sensible default values
- **Version control**: Track template changes
- **Test templates**: Validate in staging before production

### Developer Self-Service Best Practices

- **Automate project creation**: Use templates for consistent provisioning
- **Provide documentation**: Help developers understand constraints
- **Monitor and alert**: Set up alerts for quota limits
- **Review regularly**: Audit developer resource usage
- **Iterate policies**: Adjust quotas and limits based on feedback

---

## Certification Exam Notes

### Exam-Relevant Commands You Must Know

```bash
# Quota management
oc create quota <name> --hard=cpu=4,memory=8Gi,pods=10
oc get quota
oc describe quota <name> -n <namespace>
oc edit quota <name>
oc delete quota <name> -n <namespace>

# LimitRange management
oc create limitrange <name> --default=cpu=100m,memory=128Mi
oc get limitrange
oc describe limitrange <name> -n <namespace>
oc edit limitrange <name>
oc delete limitrange <name> -n <namespace>

# Template management
oc apply -f template.yaml
oc process -f template.yaml -p PARAM=value
oc process -f template.yaml | oc apply -f -
oc get templates
oc describe template <name>

# Resource usage
oc describe quota <name> -n <namespace>
oc get quota -n <namespace>
oc top pods -n <namespace>
```

### Common Exam Scenarios

1. **"Create a quota with CPU and memory limits"** → `oc create quota <name> --hard=cpu=4,memory=8Gi`
2. **"Check quota usage"** → `oc describe quota <name> -n <namespace>`
3. **"Create a limitrange with defaults"** → `oc create limitrange <name> --default=cpu=100m,memory=128Mi`
4. **"Instantiate a template"** → `oc process -f template.yaml -p PARAM=value | oc apply -f -`
5. **"View templates"** → `oc get templates`
6. **"Increase quota limits"** → `oc edit quota <name>`
7. **"Check resource usage"** → `oc describe quota <name>`
8. **"Create quota with multiple resources"** → `--hard=pods=10,cpu=4,memory=8Gi,deployments=5`
9. **"Create limitrange with min/max"** → `--min=cpu=50m --max=cpu=2`
10. **"Process template with parameters"** → `oc process -f template.yaml -p NAME=value`

### Important Exam Details

- Quotas are enforced at **namespace level**
- LimitRanges apply to **all resources** in a namespace
- Quota names must be **unique per namespace**
- LimitRange can have only **one per namespace**
- Template parameters are **case-sensitive**
- Resource values use **Kubernetes units** (m, k, M, G, Gi, Ti)
- `oc process` outputs YAML; `oc apply -f -` applies it
- Default values in LimitRange only apply to **new pods**

---

## Common Mistakes

### Mistake: Confusing Quota with LimitRange

Quotas limit aggregate consumption across all resources in a namespace. LimitRanges constrain individual resource specifications. Both are needed for proper resource management.

### Mistake: Forgetting Units in Resource Values

Resource values require units: `100m` (not `100`), `1Gi` (not `1073741824`). Omitting units causes validation errors.

### Mistake: Setting Quota Too Low

Setting quotas too low causes developers to constantly fail pod creation. Base quotas on actual usage patterns and allow some headroom.

### Mistake: Not Testing Templates

Templates with validation errors fail silently during creation. Test template instantiation before production deployment.

### Mistake: Ignoring LimitRange Defaults

LimitRange defaults only apply to new pods. Existing pods won't get default values unless restarted.

### Mistake: Using Wrong Namespace for Quota

Quotas are namespace-scoped. Creating a quota in the wrong namespace means it doesn't apply to the intended resources.

### Mistake: Hard-Coding Resource Values in Templates

Templates should use parameters for resource values, allowing customization. Hard-coding limits flexibility.

### Mistake: Not Monitoring Quota Usage

Without monitoring, quotas become ineffective. Regular quota reviews prevent resource exhaustion.

---

## Chapter Summary

This chapter covered enabling developer self-service in OpenShift Container Platform. You learned to configure cluster-wide and project-level resource quotas to limit aggregate resource consumption. You created LimitRanges to define minimum and maximum constraints for individual resources. You set default resource requirements for new deployments. You configured project templates with parameters for standardized application deployment. You understood the interaction between quotas, limit ranges, and resource requests/limits. You enabled developers to self-provision projects with appropriate constraints. You diagnosed quota and limit range enforcement failures, template parameter issues, and resource validation errors. Production best practices emphasized realistic quotas, regular monitoring, documented templates, and iterative policy adjustment.

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Create quota | `oc create quota <name> --hard=cpu=4,memory=8Gi,pods=10` |
| View quotas | `oc get quota [-A]` |
| Describe quota | `oc describe quota <name> -n <namespace>` |
| Edit quota | `oc edit quota <name>` |
| Delete quota | `oc delete quota <name> -n <namespace>` |
| Create limitrange | `oc create limitrange <name> --default=cpu=100m,memory=128Mi` |
| View limitranges | `oc get limitrange [-A]` |
| Describe limitrange | `oc describe limitrange <name> -n <namespace>` |
| Edit limitrange | `oc edit limitrange <name>` |
| Delete limitrange | `oc delete limitrange <name> -n <namespace>` |
| Create template | `oc apply -f template.yaml` |
| View templates | `oc get templates [-A]` |
| Describe template | `oc describe template <name>` |
| Process template | `oc process -f template.yaml -p PARAM=value` |
| Apply template | `oc process -f template.yaml \| oc apply -f -` |
| Check usage | `oc describe quota <name>` |
| Check resource usage | `oc top pods -n <namespace>` |

### Resource Quota Types

| Type | Description |
|------|-------------|
| `requests.cpu` | Total CPU requests across all pods |
| `requests.memory` | Total memory requests across all pods |
| `limits.cpu` | Total CPU limits across all pods |
| `limits.memory` | Total memory limits across all pods |
| `pods` | Total number of pods |
| `services` | Total number of services |
| `configmaps` | Total number of configmaps |
| `secrets` | Total number of secrets |
| `persistentvolumeclaims` | Total number of PVCs |
| `deployments` | Total number of deployments |
| `replicasets` | Total number of replicasets |
| `replicationcontrollers` | Total number of replication controllers |
| `horizontalpodautoscalers` | Total number of HPA objects |
| `networkpolicies` | Total number of network policies |

### LimitRange Types

| Type | Description |
|------|-------------|
| `Container` | Container resource constraints |
| `Pod` | Pod-level resource constraints |
| `PersistentVolumeClaim` | PVC storage constraints |
| `ConfigMap` | ConfigMap data size constraints |
| `Secret` | Secret data size constraints |
| `ServiceAccountToken` | Service account token constraints |

### Resource Value Units

| Unit |       Meaning                     |
|------|-----------------------------------|
| `m`  | Millicores (1/1000 of a CPU core) |
| `k`  | Kilo (1000)                       |
| `M`  | Mega (1,000,000)                  |
| `G`  | Giga (1,000,000,000)              |
| `Gi` | Gibi (1,073,741,824)              |
| `Ti` | Tebi (1,099,511,627,776)          |

### Template Parameter Types

| Type | Description |
|------|-------------|
| `string` | Text value |
| `boolean` | True/false |
| `integer` | Whole number |
| `image` | Container image |
| `port` | Port number |
| `disk` | Storage size |
| `route` | Route configuration |
| `secret` | Secret reference |

---

## Review Questions

1. What is the difference between a ResourceQuota and a LimitRange?

2. How do you create a project quota with CPU, memory, and pod limits?

3. What happens when a pod's resource requests exceed the namespace quota?

4. How do you set default resource limits for a namespace?

5. What is the purpose of a LimitRange?

6. How do you instantiate a template with custom parameter values?

7. What happens if you try to create a pod without specifying resource requests in a namespace with a LimitRange?

8. How do you check the current usage of a resource quota?

9. What command lists all templates in the cluster?

10. How do you increase the CPU quota limit for a namespace?

11. What is the difference between `spec.hard` in a ResourceQuota and `spec.limits` in a LimitRange?

12. How do you create a template from an existing deployment?

13. What happens if you try to deploy more pods than the quota allows?

14. How do you verify that a LimitRange is properly configured?

15. What are the key fields in a ResourceQuota YAML manifest?

---

## Answers

1. A **ResourceQuota** limits the aggregate consumption of resources across all pods and objects in a namespace (e.g., total CPU, total pods). A **LimitRange** defines minimum and maximum constraints for individual resources (e.g., each container must have at least 100m CPU).

2. `oc create quota myapp-quota --hard=cpu=4,memory=8Gi,pods=10 -n myapp-project`

3. The pod creation fails with an error like "exceeded quota: myapp-quota". The API server rejects the pod because the namespace has no available quota for the requested resources.

4. Use `oc adm policy set-default-limits --requests-cpu=100m --requests-memory=128Mi --limits-cpu=500m --limits-memory=512Mi` or create a LimitRange with `default` values.

5. A LimitRange enforces minimum and maximum resource constraints on individual pods and containers within a namespace. It prevents undersized or oversized resource allocations.

6. `oc process -f template.yaml -p PARAM1=value1 -p PARAM2=value2 | oc apply -f -`

7. If the LimitRange has `defaultRequest` or `default` values, those values are applied. If no defaults exist and the namespace has a LimitRange with min values, the pod creation fails.

8. `oc describe quota <name> -n <namespace>` shows the Used and Hard columns for each resource.

9. `oc get templates` or `oc get templates -A` for all namespaces.

10. `oc edit quota <name>` and modify the `hard.cpu` field, or use `oc patch quota <name> -p='{"spec":{"hard":{"cpu":"5"}}}'`.

11. **`spec.hard`** in ResourceQuota defines the maximum aggregate resources for the namespace. **`spec.limits`** in LimitRange defines the maximum/minimum for individual container or pod resources.

12. Export the deployment YAML, add template annotations and parameters, then apply with `oc apply -f template.yaml`.

13. The pod creation fails with "exceeded quota" error. Developers must delete existing pods or increase the quota.

14. `oc describe limitrange <name>` shows the configured min, max, default, and defaultRequest values. Test by creating pods with various resource specifications.

15. Key fields: `metadata.name` (unique per namespace), `spec.hard` (resource limits), and resource types (cpu, memory, pods, services, etc.).
