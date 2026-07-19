# Chapter 7: Enable Developer Self-Service

> **Exam baseline:** EX280 on OpenShift Container Platform 4.18.  
> **Reviewed:** 2026-07-19.  
> This chapter teaches the OpenShift **project-request template**. That is different from an application deployment `Template`, which is covered in Chapter 3.

## What You Must Be Able to Do

By the end of this chapter, you should be able to:

- configure a `ClusterResourceQuota` that selects several projects;
- configure a `ResourceQuota` for one project;
- set CPU and memory requests and limits on workloads;
- configure one or more `LimitRange` objects in a project;
- customize the template that OpenShift uses when a developer requests a project;
- verify enforcement from both an administrator and developer point of view; and
- troubleshoot rejected project, pod, and workload requests.

These are hands-on skills. Reading the commands is not enough. Run the labs on a disposable cluster.

---

## 1. The Self-Service Idea

Self-service does not mean unlimited access. It means that an administrator creates safe boundaries, and developers work independently inside those boundaries.

A useful mental model is:

```text
cluster administrator defines policy
              |
              +-- project-request template creates standard objects
              +-- ClusterResourceQuota limits a selected group of projects
              +-- ResourceQuota limits one project
              +-- LimitRange validates/defaults each container, pod, image, or PVC
              |
developer creates projects and workloads inside those rules
```

OpenShift normally allows authenticated users to request projects through the `projectrequests.project.openshift.io` API. The `oc new-project` command uses that API. A cluster administrator can customize what is created, or can remove self-provisioning permission.

### Project and namespace

A Kubernetes `Namespace` isolates namespaced objects. An OpenShift `Project` adds user-facing metadata and project-request behavior around a namespace. In daily work, `oc project`, `oc new-project`, and `oc get project` are convenient OpenShift commands, while most workload objects still have a Kubernetes namespace.

---

## 2. Requests, Limits, Scheduling, and Runtime Behavior

Each container can declare resource **requests** and **limits**:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

- A **request** is the amount used by the scheduler when it decides where the pod can fit. It is not a reservation of dedicated CPU unless additional features are configured.
- A **CPU limit** is enforced by throttling CPU time.
- A **memory limit** is enforced by terminating a container that exceeds it; the container might report `OOMKilled`.
- `100m` CPU means one tenth of a CPU core. `1` means one full core.
- `Mi` and `Gi` are binary memory units. Memory written as `400m` means 0.4 bytes and is almost certainly a mistake.

Requests and limits affect the pod's quality-of-service class:

| QoS class | Simple meaning |
|---|---|
| `Guaranteed` | Every container has equal CPU request/limit and equal memory request/limit. |
| `Burstable` | At least one request or limit is set, but the pod is not `Guaranteed`. |
| `BestEffort` | No container has CPU or memory requests or limits. |

Under node memory pressure, lower-priority and lower-QoS workloads are generally more likely to be evicted. Do not use quotas as a substitute for correct per-container requests.

### Set resources on a Deployment

```bash
oc set resources deployment/api \
  --containers=api \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=512Mi \
  -n team-a

oc rollout status deployment/api -n team-a
oc get pod -n team-a \
  -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
```

`oc set resources` changes the pod template, so the Deployment creates a new ReplicaSet and performs a rollout.

---

## 3. ResourceQuota: One Project

A Kubernetes `ResourceQuota` is namespaced. It limits the aggregate usage of selected resources in one project.

Common quota keys include:

- `requests.cpu`, `requests.memory`, and `requests.storage`;
- `limits.cpu` and `limits.memory`;
- object counts such as `pods`, `services`, `secrets`, and `persistentvolumeclaims`; and
- `count/<resource>.<api-group>` for countable API resources, for example `count/deployments.apps`.

### Create a project quota imperatively

```bash
oc create quota team-a-quota \
  --hard=requests.cpu=4,requests.memory=8Gi,limits.cpu=8,limits.memory=16Gi,pods=20,services=10,secrets=30 \
  -n team-a
```

The comma-separated form is less error-prone than repeating flags. Always inspect the result:

```bash
oc get resourcequota -n team-a
oc describe resourcequota/team-a-quota -n team-a
oc get resourcequota/team-a-quota -n team-a -o yaml
```

### Declarative example

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    secrets: "30"
    persistentvolumeclaims: "8"
    requests.storage: 100Gi
    count/deployments.apps: "10"
```

Apply and verify it:

```bash
oc apply -f team-a-quota.yaml
oc describe quota/team-a-quota -n team-a
```

### Important quota behavior

If a quota tracks `requests.cpu` or `requests.memory`, a new pod normally needs those requests, either explicitly or from a `LimitRange` default. If it tracks limits, the matching limits must be present. This is why quotas and limit ranges are commonly configured together.

Quota admission happens when an API object is created or updated. The scheduler does not reject an already accepted pod because of project quota; quota admission has already happened by then.

Deleting an object does not always make quota usage fall instantly. Controllers and quota accounting need a short time to observe the change.

---

## 4. ClusterResourceQuota: Several Selected Projects

`ClusterResourceQuota` is an OpenShift resource in `quota.openshift.io/v1`. It is cluster-scoped, but it does **not** automatically mean “one quota for the whole cluster.” Its selector chooses projects by label, annotation, or both. Usage is aggregated across the selected projects.

### Select projects by label

Label the projects first:

```bash
oc label namespace team-a quota-tier=standard --overwrite
oc label namespace team-b quota-tier=standard --overwrite
```

Create one quota shared by every matching project:

```bash
oc create clusterquota standard-tier \
  --project-label-selector=quota-tier=standard \
  --hard=pods=40 \
  --hard=requests.cpu=8 \
  --hard=requests.memory=16Gi
```

`clusterquota` and `clusterresourcequota` refer to the same resource type in this command family.

### Select projects requested by one user

The default project-request process records the requester in the `openshift.io/requester` annotation. You can select it:

```bash
oc create clusterquota alice-projects \
  --project-annotation-selector=openshift.io/requester=alice \
  --hard=pods=20 \
  --hard=secrets=40
```

### Declarative example

```yaml
apiVersion: quota.openshift.io/v1
kind: ClusterResourceQuota
metadata:
  name: standard-tier
spec:
  selector:
    labels:
      matchLabels:
        quota-tier: standard
  quota:
    hard:
      pods: "40"
      requests.cpu: "8"
      requests.memory: 16Gi
```

### Verify as cluster administrator

```bash
oc get clusterresourcequota
oc describe clusterresourcequota/standard-tier
oc get clusterresourcequota/standard-tier -o yaml
```

Look at both the selector and status. Confirm that the expected namespaces appear and that total usage is sensible.

### Verify from a selected project

A project administrator cannot change the cluster-scoped quota, but can see the projection that applies to the current project:

```bash
oc project team-a
oc get appliedclusterresourcequota
oc describe appliedclusterresourcequota
```

If no projects match the selector, the object can exist without enforcing anything. A selector typo is therefore a dangerous silent error.

---

## 5. LimitRange: Per-Object Defaults and Bounds

A `LimitRange` is namespaced. Every create or update request is evaluated against **every** `LimitRange` in the project. More than one `LimitRange` can exist, but overlapping rules can be hard to understand, so prefer one clearly documented policy unless separation is useful.

Core useful types are:

- `Container`: CPU and memory default, default request, minimum, maximum, and maximum limit-to-request ratio;
- `Pod`: aggregate CPU and memory bounds across all containers; and
- `PersistentVolumeClaim`: minimum and maximum requested storage.

OpenShift also supports limits for `openshift.io/Image` and `openshift.io/ImageStream`. A `LimitRange` does **not** set the size of ConfigMap data, Secret data, or service-account tokens.

### Correct LimitRange manifest

There is no standard `oc create limitrange` generator. Write YAML, use `oc create -f`, and verify it.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    min:
      cpu: 50m
      memory: 64Mi
    max:
      cpu: "2"
      memory: 4Gi
    maxLimitRequestRatio:
      cpu: "10"
      memory: "4"
  - type: Pod
    max:
      cpu: "4"
      memory: 8Gi
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 20Gi
```

```bash
oc apply -f team-a-limits.yaml
oc get limitrange -n team-a
oc describe limitrange/team-a-limits -n team-a
```

The short resource name `limits` is also accepted by the CLI:

```bash
oc get limits -n team-a
```

### Defaulting behavior

Defaults are added only when a new pod is admitted. Existing pods are not modified. A Deployment's existing pod-template YAML also does not magically change; inspect the created pod to see admitted defaults:

```bash
oc get pod <pod-name> -n team-a \
  -o jsonpath='{.spec.containers[*].resources}'
echo
```

If a container violates a minimum, maximum, or ratio, the API rejects the owning pod. If a Deployment is accepted but its ReplicaSet cannot create pods, the error appears in Deployment, ReplicaSet, and event details.

---

## 6. The Project-Request Template

This is the project-template skill named in the EX280 objectives.

When a user runs `oc new-project`, OpenShift processes a special OpenShift `Template`. It can create the Project, the administrator RoleBinding, and standard policy objects such as a `ResourceQuota`, `LimitRange`, and `NetworkPolicy`.

The standard parameters are:

| Parameter | Meaning |
|---|---|
| `PROJECT_NAME` | Requested project name. |
| `PROJECT_DISPLAYNAME` | Optional display name. |
| `PROJECT_DESCRIPTION` | Optional description. |
| `PROJECT_ADMIN_USER` | User who receives project administration. |
| `PROJECT_REQUESTING_USER` | User who made the request. |

### Step 1: Generate the bootstrap template

Run this as a cluster administrator:

```bash
oc adm create-bootstrap-project-template -o yaml > project-request-template.yaml
```

Do not invent the required Project and RoleBinding objects from memory. Generate the supported starting point, then add policy objects under `objects:`.

### Step 2: Add standard quota and limits

The following excerpt shows the extra objects. Keep the generated objects as well.

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: project-request
  namespace: openshift-config
objects:
# Keep the generated Project and RoleBinding objects here.
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: starter-quota
    namespace: ${PROJECT_NAME}
  spec:
    hard:
      requests.cpu: "2"
      requests.memory: 4Gi
      limits.cpu: "4"
      limits.memory: 8Gi
      pods: "15"
- apiVersion: v1
  kind: LimitRange
  metadata:
    name: starter-limits
    namespace: ${PROJECT_NAME}
  spec:
    limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: default-deny-ingress
    namespace: ${PROJECT_NAME}
  spec:
    podSelector: {}
    policyTypes:
    - Ingress
parameters:
- name: PROJECT_NAME
- name: PROJECT_DISPLAYNAME
- name: PROJECT_DESCRIPTION
- name: PROJECT_ADMIN_USER
- name: PROJECT_REQUESTING_USER
```

Do not copy only this excerpt as the complete template. The bootstrap output contains the Project and RoleBinding that make the request work.

### Step 3: Store the template in `openshift-config`

The referenced template must be in `openshift-config`:

```bash
oc apply -f project-request-template.yaml -n openshift-config
oc get template/project-request -n openshift-config
```

If the file contains another `metadata.namespace`, make it agree with `openshift-config`.

### Step 4: Point cluster configuration to it

```bash
oc patch project.config.openshift.io/cluster \
  --type=merge \
  -p '{"spec":{"projectRequestTemplate":{"name":"project-request"}}}'

oc get project.config.openshift.io/cluster \
  -o jsonpath='{.spec.projectRequestTemplate.name}'
echo
```

The equivalent interactive command is:

```bash
oc edit project.config.openshift.io/cluster
```

The relevant YAML is:

```yaml
spec:
  projectRequestTemplate:
    name: project-request
```

### Step 5: Test as a normal user

Do not test only as `cluster-admin`; that can hide permission problems.

```bash
oc login -u alice https://api.<cluster-domain>:6443
oc new-project template-test
oc get resourcequota,limitrange,networkpolicy
oc auth can-i create deployments
```

Delete the test project only after checking every expected object:

```bash
oc delete project template-test
```

### Roll back safely

Record the original cluster setting before changing it:

```bash
oc get project.config.openshift.io/cluster -o yaml > project-config.before.yaml
```

To return to the built-in default behavior, remove the reference:

```bash
oc patch project.config.openshift.io/cluster \
  --type=json \
  -p='[{"op":"remove","path":"/spec/projectRequestTemplate"}]'
```

Use the JSON patch only when the field exists. If it is already absent, no rollback is needed.

---

## 7. Project Self-Provisioning Permission

The `self-provisioner` ClusterRole allows project requests. By default, the `self-provisioners` ClusterRoleBinding grants it to `system:authenticated:oauth`.

Inspect before changing anything:

```bash
oc get clusterrole/self-provisioner
oc get clusterrolebinding/self-provisioners -o yaml
oc auth can-i create projectrequests.project.openshift.io --as=alice
```

Removing self-provisioning is a cluster policy change and is not required merely to customize the template. If a task explicitly asks you to disable it, remove the role from the group using the supported policy command and verify the effect. Keep a recovery login with cluster administration privileges.

---

## 8. Troubleshooting Method

### A project request is forbidden

```bash
oc auth can-i create projectrequests.project.openshift.io --as=<user>
oc get clusterrolebinding/self-provisioners -o yaml
oc get project.config.openshift.io/cluster -o yaml
```

- `no` from `can-i` means the user lacks self-provisioning permission.
- A custom message in `spec.projectRequestMessage` can explain the organization's process.
- A cluster administrator can still create a project with `oc adm new-project` even when normal self-provisioning is disabled.

### A new project is missing standard objects

```bash
oc get project.config.openshift.io/cluster \
  -o jsonpath='{.spec.projectRequestTemplate.name}'
echo
oc get template -n openshift-config
oc get template/project-request -n openshift-config -o yaml
```

Check the template name, namespace, indentation, `objects:` list, and `${PROJECT_NAME}` namespaces. Existing projects are not retroactively changed when the template changes.

### A pod is rejected by quota

```bash
oc describe resourcequota -n <project>
oc get appliedclusterresourcequota -n <project>
oc describe appliedclusterresourcequota -n <project>
oc get events -n <project> --sort-by=.lastTimestamp
```

Compare **Used** with **Hard**. Check both namespaced and cluster resource quotas.

### A Deployment exists but creates no pod

```bash
oc describe deployment/<name> -n <project>
oc get replicaset -n <project>
oc describe replicaset/<name> -n <project>
oc get events -n <project> --sort-by=.lastTimestamp
```

Controllers often accept the parent object and then fail to create a child pod because of quota or `LimitRange` policy. The child controller's events usually contain the exact rejection.

### Cluster quota matches the wrong projects

```bash
oc get clusterresourcequota/<name> -o yaml
oc get namespace --show-labels
oc get namespace <project> -o jsonpath='{.metadata.annotations}'
echo
```

Fix the namespace label/annotation or the quota selector. Do not raise the quota until you know that selection is correct.

---

## 9. Hands-On Exam Lab

Perform this lab without copying the solution line by line.

### Requirements

1. Create projects `lab-dev-a` and `lab-dev-b`.
2. Label both with `quota-tier=lab`.
3. Create a `ClusterResourceQuota` named `lab-tier` shared by both projects: 12 pods, 4 CPU requests, and 8 GiB memory requests.
4. In `lab-dev-a`, create a namespaced quota that permits at most 6 services and 10 secrets.
5. In both projects, install a `LimitRange` that defaults a container request to 100m CPU and 128Mi memory and a limit to 500m CPU and 512Mi memory.
6. Create a Deployment without explicit resources. Verify the admitted pod received defaults.
7. Deliberately exceed one limit and capture the exact error.
8. Generate a project-request template, add a small quota and the same defaults, install it, configure the cluster to use it, and test with a third project.
9. Restore the original project configuration and delete the three lab projects.

### Verification checklist

```bash
oc describe clusterresourcequota/lab-tier
oc get appliedclusterresourcequota -n lab-dev-a
oc describe resourcequota -n lab-dev-a
oc describe limitrange -n lab-dev-a
oc get pod -n lab-dev-a -o yaml
oc get project.config.openshift.io/cluster -o yaml
```

Pass the lab only if each command proves the requested state. “The apply command succeeded” is not verification.

---

## 10. Exam Traps and Corrections

- A `ResourceQuota` is one-project scope. A `ClusterResourceQuota` selects and aggregates several projects.
- A cluster-scoped object is not automatically a whole-cluster limit; read its selector.
- More than one `LimitRange` can exist in a project, and every matching rule is evaluated.
- There is no standard `oc create limitrange` generator. Use YAML.
- `LimitRange` does not constrain ConfigMap/Secret data size or service-account tokens.
- The EX280 “project template” is the **project-request template**, not a reusable web-application template.
- The project-request template must be stored in `openshift-config` and referenced by `project.config.openshift.io/cluster`.
- Quota usage is based on API objects and declared requests/limits; `oc top` shows measured runtime use. They answer different questions.
- Test project requests as a non-administrator.
- Configuration must persist. Do not rely on a temporary shell, port-forward, or manual post-reboot repair.

---

## 11. Quick Reference

| Task | Command |
|---|---|
| Create project quota | `oc create quota NAME --hard=requests.cpu=4,pods=20 -n PROJECT` |
| View quota usage | `oc describe quota -n PROJECT` |
| Create selected multi-project quota | `oc create clusterquota NAME --project-label-selector=key=value --hard=pods=20` |
| View multi-project quota | `oc describe clusterresourcequota/NAME` |
| View quota projected into a project | `oc describe appliedclusterresourcequota -n PROJECT` |
| Apply a LimitRange | `oc apply -f limits.yaml` |
| View limits | `oc describe limitrange -n PROJECT` |
| Set workload resources | `oc set resources deployment/NAME --requests=cpu=100m,memory=128Mi --limits=cpu=500m,memory=512Mi` |
| Generate project template | `oc adm create-bootstrap-project-template -o yaml > project-request-template.yaml` |
| Install project template | `oc apply -f project-request-template.yaml -n openshift-config` |
| Configure template reference | `oc edit project.config.openshift.io/cluster` |
| Test project permission | `oc auth can-i create projectrequests.project.openshift.io --as=USER` |

---

## 12. Review Questions

1. What is the difference between `ResourceQuota`, `ClusterResourceQuota`, and `LimitRange`?
2. Why do CPU and memory quotas often need a `LimitRange` with defaults?
3. How can a cluster quota select every project requested by one user?
4. Can a namespace contain two `LimitRange` objects?
5. Which object points OpenShift to a custom project-request template?
6. In which namespace must the referenced template exist?
7. Why should you test project creation as a normal user?
8. Why might a Deployment exist while no pods are created?
9. Does `oc top pods` show quota accounting?
10. How do you verify a cluster quota from inside a selected project?

## 13. Answers

1. `ResourceQuota` limits aggregate resources in one project. `ClusterResourceQuota` aggregates limits across projects selected by labels/annotations. `LimitRange` supplies defaults and validates individual containers, pods, images/image streams, or PVCs.
2. A quota that tracks requests or limits can reject pods that omit those fields. A `LimitRange` can supply them at admission.
3. Select the `openshift.io/requester=<user>` project annotation with `--project-annotation-selector`.
4. Yes. Every `LimitRange` is evaluated. Avoid confusing overlapping rules.
5. `project.config.openshift.io/cluster`, in `spec.projectRequestTemplate.name`.
6. `openshift-config`.
7. Cluster administration can bypass or hide permissions that affect ordinary self-provisioners.
8. The Deployment controller's ReplicaSet can be unable to create pods because quota or `LimitRange` admission rejects them. Inspect ReplicaSet events.
9. No. `oc top` shows measured runtime use from metrics. Quota status reports admission accounting based on objects and their declared resources.
10. Use `oc get` or `oc describe appliedclusterresourcequota -n <project>`.

---

## Verified References

- EX280 objectives: <https://www.redhat.com/en/services/training/red-hat-certified-openshift-administrator-exam>
- OpenShift 4.18 project creation and project-request templates: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/building_applications/projects>
- OpenShift 4.18 quota and LimitRange behavior: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/scalability_and_performance/compute-resource-quotas>
- OpenShift API reference for `ClusterResourceQuota`, `AppliedClusterResourceQuota`, and `LimitRange`: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/schedule_and_quota_apis/index>
