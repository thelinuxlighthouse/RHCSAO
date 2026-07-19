# Chapter 9: Configure Application Security

> **Exam baseline:** EX280 on OpenShift Container Platform 4.18.  
> **Reviewed:** 2026-07-19.  
> Security Context Constraints (SCCs) and RBAC work together. SCCs do **not** replace RBAC.

## What You Must Be Able to Do

By the end of this chapter, you should be able to:

- create and select service accounts;
- grant a service account minimum API permissions and test them;
- create a short-lived service-account token for a human/tooling use case;
- link an image pull Secret to a service account;
- explain SCC authorization, selection, defaulting, and validation;
- run a privileged workload only when explicitly required;
- create and consume Secrets without confusing base64 with encryption;
- create, verify, and troubleshoot Jobs; and
- create, verify, manually test, suspend, and troubleshoot CronJobs.

---

## 1. Service Accounts: Workload Identities

A service account (SA) is a namespaced Kubernetes identity for workloads and automation. Its full user name is:

```text
system:serviceaccount:<namespace>:<service-account-name>
```

Related groups include all service accounts and the service accounts in one namespace:

```text
system:serviceaccounts
system:serviceaccounts:<namespace>
```

Every namespace has a `default` service account. A pod uses it when `spec.serviceAccountName` is omitted. Create a dedicated account when a workload needs API access, special SCC permission, or distinct image-pull credentials.

```bash
oc create serviceaccount report-reader -n reports
oc get serviceaccount/report-reader -n reports -o yaml
```

Select it in a pod template:

```yaml
spec:
  serviceAccountName: report-reader
  containers:
  - name: report
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
```

Changing a Deployment's service account changes its pod template and triggers a rollout.

### Modern service-account tokens

OpenShift uses bound, projected service-account tokens for pods. They are time-limited, audience-bound, and rotated by the kubelet. Older automatically generated long-lived token Secrets are not the normal model.

The default projected files are usually under:

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
/var/run/secrets/kubernetes.io/serviceaccount/namespace
```

Disable automatic mounting when the application does not call the API:

```yaml
spec:
  serviceAccountName: no-api-client
  automountServiceAccountToken: false
```

Create a token for a temporary CLI/test use case:

```bash
oc create token report-reader -n reports
oc create token report-reader -n reports --duration=30m
```

The server can enforce or adjust duration. Do not solve long-running application authentication by generating a huge-duration static token. Let pods use rotating projected tokens, and make the application reload the token when it changes.

Never print or copy a token into a book, Git repository, chat, or command log.

---

## 2. Grant API Access with RBAC

Service accounts start with very little application API permission. Grant the smallest role in the smallest namespace.

### Bind a built-in role

```bash
oc adm policy add-role-to-user view \
  -z report-reader \
  -n reports
```

`-z` means a service account in the namespace supplied by `-n`. The equivalent full subject is `system:serviceaccount:reports:report-reader`.

Avoid examples that grant `admin` or `cluster-admin` to an application merely to make an error disappear.

### Custom namespaced Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
  namespace: reports
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["report-config"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: report-reader-config
  namespace: reports
subjects:
- kind: ServiceAccount
  name: report-reader
  namespace: reports
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: config-reader
```

```bash
oc apply -f report-rbac.yaml
```

### Prove the permission

```bash
SA=system:serviceaccount:reports:report-reader

oc auth can-i get configmap/report-config --as="$SA" -n reports
oc auth can-i list secrets --as="$SA" -n reports
```

Expected: the exact ConfigMap get is `yes`; broad Secret listing should remain `no` unless explicitly required.

---

## 3. Image Pull Secrets

Create a registry Secret without exposing the password in shared shell history when possible. This direct form is common in isolated labs:

```bash
oc create secret docker-registry private-registry \
  --docker-server=registry.example.com \
  --docker-username='<user>' \
  --docker-password='<password>' \
  --docker-email='<email>' \
  -n reports
```

Link it for image pulls:

```bash
oc secrets link report-reader private-registry --for=pull -n reports
oc get serviceaccount/report-reader -n reports \
  -o jsonpath='{.imagePullSecrets[*].name}{"\n"}'
```

Do not use an invented `imagepullsecrets` annotation. The SA field is `imagePullSecrets`, and `oc secrets link` edits it correctly.

The Secret must be in the same namespace as the service account/pod. Linking registry credentials for pull does not grant Kubernetes API permission.

---

## 4. Security Context Constraints

An SCC is an OpenShift admission policy for pod security. It can control:

- privileged containers and privilege escalation;
- allowed Linux capabilities;
- host network, PID, IPC, ports, paths, and devices;
- allowed volume types;
- UID, SELinux, FSGroup, and supplemental group strategies;
- seccomp profiles; and
- read-only root-file-system requirements.

### How SCC admission works

```text
pod create request
      |
identify requesting user and pod service account
      |
find SCCs they are authorized by RBAC to "use"
      |
sort/select an SCC that can default and validate the pod
      |
admit pod and record chosen SCC, or reject with all validation reasons
```

The service account does not have one permanent SCC “assigned” to every pod. It is authorized to use one or more SCCs; admission selects a suitable SCC for each pod. The admitted pod records the result in the `openshift.io/scc` annotation.

### Inspect built-in SCCs

```bash
oc get scc
oc describe scc/restricted-v2
oc describe scc/anyuid
oc describe scc/privileged
```

`restricted-v2` is the normal restrictive baseline in modern OpenShift. `anyuid` permits arbitrary UID choices. `privileged` allows highly dangerous host-level behavior. Never edit built-in SCC objects; updates can reset them and the change can weaken unrelated workloads.

### Test SCC authorization

```bash
SA=system:serviceaccount:reports:report-reader
oc auth can-i use scc/restricted-v2 --as="$SA"
oc auth can-i use scc/privileged --as="$SA"
```

### Grant and remove SCC use

Only when the application requirement proves it is necessary:

```bash
oc adm policy add-scc-to-user anyuid \
  -z legacy-app \
  -n legacy

oc adm policy remove-scc-from-user anyuid \
  -z legacy-app \
  -n legacy
```

These commands create/remove RBAC authorization for the SA to use the SCC. Verify with `oc auth can-i` and with an admitted pod annotation.

### Diagnose selected SCC

```bash
oc get pod/<pod> -n <namespace> \
  -o jsonpath='{.metadata.annotations.openshift\.io/scc}{"\n"}'
```

---

## 5. Run a Privileged Application Only When Required

A privileged container has near-host-level power. Prefer changing the application to run under `restricted-v2`, then consider a narrow custom SCC or `anyuid`. Use `privileged` only when the task explicitly requires privileged operation.

### Controlled lab sequence

```bash
oc create namespace privileged-lab
oc create serviceaccount device-agent -n privileged-lab
oc adm policy add-scc-to-user privileged \
  -z device-agent \
  -n privileged-lab
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: device-agent
  namespace: privileged-lab
spec:
  serviceAccountName: device-agent
  restartPolicy: Never
  containers:
  - name: agent
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["/bin/sh", "-c", "id; sleep 3600"]
    securityContext:
      privileged: true
```

```bash
oc apply -f privileged-pod.yaml
oc get pod/device-agent -n privileged-lab \
  -o jsonpath='{.metadata.annotations.openshift\.io/scc}{"\n"}'
oc exec -n privileged-lab device-agent -- id
```

Cleanup:

```bash
oc delete project privileged-lab
```

The container-level `securityContext.privileged` field belongs under a container, not the pod-level security context. Do not add host networking, host PID, hostPath, every capability, and root UID unless each is separately required.

### Custom SCC guidance

Creating an SCC is a cluster-level security design task. If a task requires a custom SCC:

1. copy only the fields needed from a restrictive baseline;
2. give it a unique name;
3. avoid `volumes: ['*']`, `RunAsAny`, all host namespaces, and broad capabilities unless required;
4. do not embed broad users/groups when RBAC can grant `use` cleanly;
5. grant it only to the intended service account; and
6. test that the required pod succeeds and an excessive pod still fails.

---

## 6. Secrets

A Secret stores byte values in the Kubernetes API. YAML `data` values are base64-encoded; base64 is reversible encoding, not encryption. Protection also depends on RBAC, etcd encryption-at-rest configuration, backups, audit/log handling, and how the application uses the value.

### Create Secrets

```bash
oc create secret generic database-credentials \
  --from-literal=username='<user>' \
  --from-literal=password='<password>' \
  -n reports

oc create secret tls report-tls \
  --cert=server.crt \
  --key=server.key \
  -n reports
```

For a declarative example, `stringData` accepts clear text on input and the API converts it to `data`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: reports
type: Opaque
stringData:
  username: example-user
  password: replace-before-apply
```

Do not commit real `stringData` or decoded `data` to Git.

### Consume as environment variables

```yaml
env:
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: database-credentials
      key: username
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: database-credentials
      key: password
```

Environment values do not update inside an already running container when the Secret changes. Roll out new pods.

### Consume as a volume

```yaml
volumes:
- name: db-credentials
  secret:
    secretName: database-credentials
containers:
- name: report
  volumeMounts:
  - name: db-credentials
    mountPath: /var/run/secrets/report-db
    readOnly: true
```

Projected Secret volumes are updated eventually, but the application must reread the file. A `subPath` mount does not receive automatic updates.

### Inspect without leaking

```bash
oc get secret/database-credentials -n reports
oc get secret/database-credentials -n reports \
  -o jsonpath='{.data}'
echo
```

Decode only when necessary and avoid putting the output in screenshots/logs:

```bash
oc get secret/database-credentials -n reports \
  -o jsonpath='{.data.username}' | base64 --decode
echo
```

---

## 7. Jobs

A Job creates pods until the required number complete successfully.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: report-once
  namespace: reports
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      serviceAccountName: report-reader
      restartPolicy: Never
      containers:
      - name: report
        image: registry.access.redhat.com/ubi9/ubi-minimal:latest
        command: ["/bin/sh", "-c", "date; echo report complete"]
```

`restartPolicy` for Job pods must be `Never` or `OnFailure`; either can be valid. With `Never`, a failed container produces a failed pod and the Job controller can create another pod. With `OnFailure`, kubelet can restart the container in the same pod.

```bash
oc apply -f job.yaml
oc get job/report-once -n reports
oc describe job/report-once -n reports
oc get pod -n reports -l batch.kubernetes.io/job-name=report-once
oc logs job/report-once -n reports
oc wait --for=condition=complete job/report-once -n reports --timeout=5m
```

Key fields:

- `completions`: successful completions required;
- `parallelism`: maximum pods running at once;
- `backoffLimit`: failed attempts allowed before Job failure;
- `activeDeadlineSeconds`: total execution deadline; and
- `ttlSecondsAfterFinished`: optional cleanup after completion/failure.

---

## 8. CronJobs

A CronJob creates Jobs according to a schedule. It does not run application containers directly.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
  namespace: reports
spec:
  schedule: "0 2 * * *"
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          serviceAccountName: report-reader
          restartPolicy: Never
          containers:
          - name: report
            image: registry.access.redhat.com/ubi9/ubi-minimal:latest
            command: ["/bin/sh", "-c", "date; echo scheduled report"]
```

Cron fields are minute, hour, day of month, month, and day of week. `0 2 * * *` is 02:00 each day in the configured time zone.

Concurrency policies:

- `Allow`: overlapping Jobs are allowed;
- `Forbid`: skip a new run while the previous one is still active; and
- `Replace`: replace the active run with the new run.

Verify and manually test:

```bash
oc apply -f cronjob.yaml
oc get cronjob/nightly-report -n reports
oc describe cronjob/nightly-report -n reports

oc create job manual-report \
  --from=cronjob/nightly-report \
  -n reports
oc logs job/manual-report -n reports
```

Suspend/resume:

```bash
oc patch cronjob/nightly-report -n reports \
  --type=merge -p '{"spec":{"suspend":true}}'

oc patch cronjob/nightly-report -n reports \
  --type=merge -p '{"spec":{"suspend":false}}'
```

---

## 9. Troubleshooting

### Pod rejected by SCC

The API error usually lists every SCC considered and why each failed. Read the complete error.

```bash
SA=system:serviceaccount:<namespace>:<service-account>
oc auth can-i use scc/<scc> --as="$SA"
oc get pod/<pod> -n <namespace> -o yaml
oc describe scc/<scc>
```

Fix unnecessary pod privileges first. Grant a broader SCC only if the workload requirement justifies it.

### Application receives HTTP 401

401 means authentication failed: missing/invalid/expired token or wrong audience. Confirm projected token mounting and application reload behavior. Do not print the token.

```bash
oc get pod/<pod> -n <namespace> \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}{.spec.automountServiceAccountToken}{"\n"}'
```

### Application receives HTTP 403

Authentication succeeded but authorization/admission denied the request:

```bash
SA=system:serviceaccount:<namespace>:<service-account>
oc auth can-i <verb> <resource> --as="$SA" -n <namespace>
oc get rolebinding -n <namespace> -o yaml
```

Match API group, resource/subresource, verb, namespace, and optional `resourceNames`.

### Image pull fails

```bash
oc describe pod/<pod> -n <namespace>
oc get serviceaccount/<sa> -n <namespace> -o yaml
oc get secret/<pull-secret> -n <namespace>
```

Check registry hostname, credentials, SA link, image pull specification, and registry trust/network access.

### Secret not found or wrong key

```bash
oc get secret -n <namespace>
oc get secret/<name> -n <namespace> -o jsonpath='{.data}'
echo
oc describe pod/<pod> -n <namespace>
```

Secrets are namespaced. The pod and referenced Secret must be in the same namespace, and key names are case-sensitive.

### Job never completes

```bash
oc describe job/<job> -n <namespace>
oc get pod -n <namespace> -l batch.kubernetes.io/job-name=<job>
oc logs job/<job> -n <namespace>
oc get events -n <namespace> --sort-by=.lastTimestamp
```

Check command exit code, retries, deadline, image pulls, SCC, quota, and whether the process actually exits.

### CronJob creates no Jobs

```bash
oc get cronjob/<name> -n <namespace> -o yaml
oc describe cronjob/<name> -n <namespace>
```

Check `spec.suspend`, schedule, time zone, last schedule time, starting deadline, controller events, and whether `Forbid` sees a previous active Job.

---

## 10. Hands-On Exam Lab

1. Create project `security-lab` and service account `auditor`.
2. Create a Role allowing only `get` and `list` on ConfigMaps; bind it to `auditor`.
3. Prove allowed and denied actions with `oc auth can-i --as=system:serviceaccount:...`.
4. Create a Secret and mount it read-only into a pod without printing its value.
5. Link a registry pull Secret to the SA and verify `imagePullSecrets`.
6. Create a one-time Job using the SA and prove completion/logs.
7. Create a CronJob, manually instantiate it as a Job, then suspend the schedule.
8. In a disposable namespace, authorize a dedicated SA to use `privileged`, run the minimal privileged pod, prove the selected SCC annotation, and delete the project.

---

## 11. Exam Traps

- SCC does not replace RBAC; RBAC grants the `use` verb on an SCC.
- A service account can use several SCCs; admission selects one per pod.
- The selected SCC is in the pod annotation `openshift.io/scc`.
- Do not modify built-in SCCs.
- Use `-z` when `oc adm policy` should target a service account.
- Base64 is not encryption.
- Modern pod tokens are projected and rotated; do not create huge-duration static tokens for applications.
- Link image pull Secrets with `oc secrets link ... --for=pull`.
- Job pod `restartPolicy` can be `Never` or `OnFailure`, not `Always`.
- A CronJob creates Jobs, and `concurrencyPolicy: Forbid` skips overlap rather than queuing unlimited work.
- Real Secrets and pull credentials never belong in source control.

---

## 12. Quick Reference

| Task | Command |
|---|---|
| Create SA | `oc create serviceaccount NAME -n NS` |
| Create temporary token | `oc create token NAME -n NS --duration=30m` |
| Grant namespaced role to SA | `oc adm policy add-role-to-user ROLE -z SA -n NS` |
| Test SA permission | `oc auth can-i VERB RESOURCE --as=system:serviceaccount:NS:SA -n NS` |
| Link pull Secret | `oc secrets link SA SECRET --for=pull -n NS` |
| List SCCs | `oc get scc` |
| Grant SCC use | `oc adm policy add-scc-to-user SCC -z SA -n NS` |
| Show chosen SCC | `oc get pod/POD -n NS -o jsonpath='{.metadata.annotations.openshift\.io/scc}'` |
| Create generic Secret | `oc create secret generic NAME --from-literal=key=value -n NS` |
| Job logs | `oc logs job/NAME -n NS` |
| Manual Job from CronJob | `oc create job NAME --from=cronjob/CRONJOB -n NS` |

---

## 13. Review Questions and Answers

1. **What is a service account's full identity?** `system:serviceaccount:<namespace>:<name>`.
2. **What is the difference between a 401 and 403 from the API?** 401 is failed authentication; 403 is an authenticated request denied by authorization or admission.
3. **Does an SCC replace a Role?** No. They control different stages, and RBAC authorizes SCC use.
4. **How do you prove which SCC admitted a pod?** Read its `openshift.io/scc` annotation.
5. **Why is Secret `data` not encrypted merely because it looks unreadable?** It is base64-encoded and can be decoded without a key.
6. **Which Job restart policies are allowed?** `Never` and `OnFailure`.
7. **How do you test a CronJob immediately?** Create a Job with `--from=cronjob/<name>`.
8. **What should a long-running pod use for API authentication?** Its rotating projected bound service-account token, not a manually copied long-lived token.

---

## Verified References

- EX280 objectives: <https://www.redhat.com/en/services/training/red-hat-certified-openshift-administrator-exam>
- OpenShift 4.18 service accounts and bound tokens: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/using-service-accounts>
- OpenShift 4.18 SCC administration: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/managing-pod-security-policies>
- OpenShift 4.18 security and compliance: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/security_and_compliance/index>
- Kubernetes Jobs and CronJobs used by OCP 4.18: <https://kubernetes.io/docs/concepts/workloads/controllers/job/> and <https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/>
