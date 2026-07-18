# Chapter 9: Configure Application Security

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure and manage service accounts for applications
- Understand and run privileged applications when necessary
- Manage and apply Security Context Constraints (SCCs) for pod security
- Create and apply secrets to manage sensitive information
- Configure application access to Kubernetes APIs using ClusterRoles
- Deploy Jobs to perform one-time batch tasks
- Configure CronJobs for scheduled batch processing
- Understand the relationship between SCCs, Roles, and service accounts
- Troubleshoot common application security issues
- Apply security best practices in production environments

---

## OpenShift Concepts

### Service Accounts

A service account is an identity used by applications within a cluster. Unlike user accounts, service accounts are used for automated processes and applications that need to interact with the Kubernetes API.

**Key Differences:**
- **Users**: Human administrators who authenticate via passwords, tokens, or identity providers
- **Service Accounts**: Application identities that authenticate using tokens

Service accounts can be bound to Roles/ClusterRoles to grant API access permissions.

### Security Context Constraints (SCCs)

SCCs are OpenShift's security model for controlling what a pod can do. They replace Kubernetes RBAC for pod-level security by defining:

- **Allowed capabilities**: Linux capabilities like NET_ADMIN, SYS_ADMIN
- **Allowed volumes**: Which volume types (hostPath, secret, configMap, etc.)
- **Allowed users/groups**: RunAsUser, RunAsGroup constraints
- **SELinux contexts**: Required security contexts
- **RunAsNonRoot**: Require pods to run as non-root user
- **FSGroup**: File system group for volume permissions

SCCs grant permissions to Service Accounts. A pod's service account determines which SCC applies.

### Privileged Applications

Privileged applications require elevated privileges to perform administrative tasks or access sensitive resources. OpenShift restricts privileged operations by default to maintain security.

**When Privileged Access is Needed:**
- Database operations requiring SYS_ADMIN
- Network operations requiring NET_ADMIN
- Volume mounting requiring specific capabilities
- Custom runtime operations

**Security Implications:**
- Privileged pods bypass many security controls
- Increased attack surface
- Potential for cluster compromise
- Should be minimized and carefully controlled

### Secrets

Secrets store sensitive data such as passwords, API keys, certificates, and tokens. They are base64-encoded and stored in etcd.

**Secret Types:**
- **Opaque**: Generic key-value pairs (default)
- **kubernetes.io/tls**: TLS certificate and key pairs
- **kubernetes.io/dockerconfigjson**: Container registry authentication
- **kubernetes.io/basic-auth**: HTTP basic authentication
- **kubernetes.io/ssh-auth**: SSH private keys

### Jobs and CronJobs

**Jobs**: Execute a fixed number of successful pod completions. Ideal for batch processing tasks.

**CronJobs**: Schedule Jobs to run at specified intervals using Cron syntax. Ideal for recurring batch tasks.

### API Access Control

Applications need API access to perform operations on cluster resources. Access is controlled through:

- **Service Accounts**: Identity for the application
- **Roles/ClusterRoles**: Permission definitions
- **RoleBindings/ClusterRoleBindings**: Bind permissions to service accounts

---

## Architecture and Internal Operation

### Service Account Token Flow

```
Pod starts with Service Account
        │
        v
Service Account Token (automatically mounted at /var/run/secrets/kubernetes.io/serviceaccount/)
        │
        v
Application reads token for API authentication
        │
        v
API Server validates token and grants access based on RoleBindings
```

### SCC Enforcement Flow

```
Pod Creation Request
        │
        v
API Server checks Service Account's SCC
        │
        ├── SCC allows operation → Create pod
        │
        └── SCC denies operation → Reject with SCC violation error
```

### Secrets Mounting Flow

```
Pod spec references Secret
        │
        v
Kubelet retrieves Secret from API server
        │
        v
Secret content written to temporary directory on node
        │
        v
Directory mounted into container at specified path
        │
        v
Container reads secret as file or environment variable
```

### Job Execution Flow

```
Job created
        │
        v
Job controller creates pod
        │
        v
Pod runs to completion
        │
        v
Pod succeeds or fails
        │
        v
Job controller checks completion count
        │
        ├── Target reached → Mark Job complete
        │
        └── Not reached → Restart pod
```

### CronJob Scheduling Flow

```
CronJob created with Cron expression
        │
        v
CronJob controller checks schedule
        │
        v
Scheduled time reached
        │
        v
CronJob creates Job
        │
        v
Job executes (same flow as Job)
        │
        v
Job completes
        │
        v
CronJob waits for next scheduled time
```

---

## Components and Terminology

| Term | Description |
|------|-------------|
| Service Account | Kubernetes identity for applications |
| Service Account Token | API authentication token for service accounts |
| SCC | Security Context Constraint - OpenShift pod security policy |
| RunAsUser | User ID constraint for pod containers |
| RunAsGroup | Group ID constraint for pod containers |
| RunAsNonRoot | Require non-root user |
| AllowPrivilegeEscalation | Allow container to gain more privileges |
| HostNetwork | Use host network namespace |
| HostPID | Use host PID namespace |
| HostIPC | Use host IPC namespace |
| HostPath Volume | Mount host directory into container |
| Secret | Kubernetes resource for sensitive data |
| Job | Kubernetes resource for batch processing |
| CronJob | Kubernetes resource for scheduled batch processing |
| CompletionMode | How Job determines completion (OnComplete or NonIndexed) |
| BackoffLimit | Number of retry attempts for failed Job |
| ActiveDeadlineSeconds | Maximum job execution time |
| TTLSecondsAfterFinished | Time to keep completed Job |

---

## Administration Tasks

### Configuring and Managing Service Accounts

#### Creating a Service Account

```bash
oc create serviceaccount myapp-sa -n myapp-project
```

#### Viewing Service Accounts

```bash
oc get serviceaccounts
oc get sa myapp-sa -n myapp-project
oc describe sa myapp-sa -n myapp-project
```

#### Adding a Token to a Service Account

```bash
oc create token myapp-sa -n myapp-project
oc create token myapp-sa --duration=24h -n myapp-project
```

#### Adding ImagePullSecrets to a Service Account

```bash
oc create serviceaccount myapp-sa -n myapp-project
oc create secret docker-registry myapp-registry \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com \
  -n myapp-project
oc annotate serviceaccount myapp-sa \
  imagepullsecrets="myapp-registry" -n myapp-project
```

#### Binding Service Account to Role

```bash
oc adm policy add-role-to-user admin myapp-sa -n myapp-project
oc adm policy add-cluster-role-to-user cluster-admin myapp-sa
```

#### Deleting a Service Account

```bash
oc delete serviceaccount myapp-sa -n myapp-project
```

### Running Privileged Applications

#### Creating a Pod with Privileged Access

```bash
oc apply -f privileged-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: myapp-project
  annotations:
    security.alpha.kubernetes.io/seccompProfile: 'localhost/unconfined'
spec:
  serviceAccountName: privileged-sa
  containers:
    - name: privileged
      image: registry.redhat.io/ubi9/ubi-minimal:latest
      securityContext:
        privileged: true
        runAsUser: 0
        runAsGroup: 0
        fsGroup: 0
      volumeMounts:
        - name: host-path
          mountPath: /host/path
        - name: host-device
          mountPath: /dev
          readOnly: true
  volumes:
    - name: host-path
      hostPath:
        path: /var/log
    - name: host-device
      hostPath:
        path: /dev
        type: Directory
```

#### Creating Pod with HostNetwork

```yaml
spec:
  serviceAccountName: privileged-sa
  hostNetwork: true
  containers:
    - name: app
      image: registry.redhat.io/ubi9/ubi-minimal:latest
```

#### Creating Pod with HostPID

```yaml
spec:
  serviceAccountName: privileged-sa
  hostPID: true
  containers:
    - name: app
      image: registry.redhat.io/ubi9/ubi-minimal:latest
```

### Managing and Applying SCCs

#### Listing SCCs

```bash
oc get scc
oc get scc -o wide
```

#### Viewing SCC Details

```bash
oc describe scc privileged
oc describe scc restricted
```

#### Creating Custom SCC

```bash
oc apply -f custom-scc.yaml
```

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: custom-scc
  annotations:
    description: "Custom SCC for specific use case"
allowPrivilegedContainer: true
allowHostDirVolumePlugin: true
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowedCapabilities:
  - NET_ADMIN
  - SYS_ADMIN
allowedUnsafeSysctls:
  - net.ipv4.ip_forward
defaultAddCapabilities: []
defaultAllowPrivilegeEscalation: false
fsGroup:
  type: RunAsAny
groups: []
readOnlyRootFilesystem: false
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
users:
  - system:serviceaccount:myapp-project:myapp-sa
volumes:
  - '*'
```

#### Assigning SCC to Service Account

```bash
oc adm policy add-scc-to-user privileged -z myapp-sa -n myapp-project
oc adm policy add-scc-to-user restricted -z myapp-sa -n myapp-project
```

#### Removing SCC from Service Account

```bash
oc adm policy remove-scc-from-user privileged -z myapp-sa -n myapp-project
```

#### Creating SCC via YAML

```bash
oc apply -f scc.yaml
```

### Creating and Applying Secrets

#### Creating Opaque Secret from Literal Values

```bash
oc create secret generic myapp-secret \
  --from-literal=username=admin \
  --from-literal=password=S3cureP@ss! \
  --from-literal=api-key=abc123 \
  -n myapp-project
```

#### Creating Secret from File

```bash
oc create secret generic tls-secret \
  --from-file=tls.crt=/path/to/cert.pem \
  --from-file=tls.key=/path/to/key.pem \
  -n myapp-project
```

#### Creating Docker Registry Secret

```bash
oc create secret docker-registry myapp-registry \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com \
  -n myapp-project
```

#### Creating TLS Secret

```bash
oc create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n myapp-project
```

#### Creating Secret from Literal Values (stringData)

```bash
oc create secret generic myapp-secret \
  --from-literal=username=admin \
  --from-literal=password=S3cureP@ss! \
  -n myapp-project
```

#### Viewing Secret Content

```bash
oc get secret myapp-secret -o yaml
oc get secret myapp-secret -o jsonpath='{.data.password}' | base64 -d && echo
```

#### Updating Secret

```bash
oc create secret generic myapp-secret \
  --from-literal=password=NewP@ss! \
  --dry-run=client -o yaml | oc apply -f -
```

#### Deleting Secret

```bash
oc delete secret myapp-secret -n myapp-project
```

### Configuring Application Access to Kubernetes APIs

#### Creating a Custom ClusterRole

```bash
oc apply -f custom-clusterrole.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/status"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get"]
```

#### Creating a Custom Role

```bash
oc apply -f custom-role.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
  namespace: myapp-project
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]
```

#### Creating RoleBinding

```bash
oc apply -f rolebinding.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: myapp-project
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: myapp-project
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### Creating ClusterRoleBinding

```bash
oc apply -f clusterrolebinding.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-reader-binding
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: myapp-project
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### Testing API Access

```bash
oc adm policy can-I --as=system:serviceaccount:myapp-project:myapp-sa \
  get pods -n myapp-project
```

### Deploying Jobs

#### Creating a Basic Job

```bash
oc apply -f job.yaml
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myapp-job
  namespace: myapp-project
spec:
  template:
    metadata:
      name: myapp-job-pod
    spec:
      restartPolicy: OnFailure
      containers:
        - name: myapp-job
          image: registry.redhat.io/ubi9/bash:latest
          command:
            - /bin/sh
            - -c
            - 'echo "Starting job"; sleep 10; echo "Job complete"'
      backoffLimit: 3
  backoffLimit: 3
```

#### Creating Job with Completion Count

```yaml
spec:
  template:
    spec:
      containers:
        - name: processor
          image: registry.redhat.io/ubi9/bash:latest
          command:
            - /bin/sh
            - -c
            - 'echo "Processing"; sleep 5; echo "Done"'
      restartPolicy: OnFailure
  completions: 5
  parallelism: 2
```

#### Creating Job with ActiveDeadline

```yaml
spec:
  template:
    spec:
      containers:
        - name: processor
          image: registry.redhat.io/ubi9/bash:latest
          command:
            - /bin/sh
            - -c
            - 'echo "Processing"; sleep 100; echo "Done"'
      restartPolicy: OnFailure
  activeDeadlineSeconds: 300
```

#### Creating Job with TTL

```yaml
spec:
  template:
    spec:
      containers:
        - name: processor
          image: registry.redhat.io/ubi9/bash:latest
          command:
            - /bin/sh
            - -c
            - 'echo "Processing"; sleep 5; echo "Done"'
      restartPolicy: OnFailure
  ttlSecondsAfterFinished: 3600
```

#### Viewing Job Status

```bash
oc get jobs
oc get job myapp-job -n myapp-project
oc describe job myapp-job -n myapp-project
oc logs job/myapp-job -n myapp-project
```

### Configuring CronJobs

#### Creating a CronJob

```bash
oc apply -f cronjob.yaml
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: myapp-cronjob
  namespace: myapp-project
spec:
  schedule: "*/5 * * * *"  # Every 5 minutes
  jobTemplate:
    metadata:
      name: myapp-job
    spec:
      template:
        metadata:
          name: myapp-job-pod
        spec:
          restartPolicy: OnFailure
          containers:
            - name: myapp-job
              image: registry.redhat.io/ubi9/bash:latest
              command:
                - /bin/sh
                - -c
                - 'echo "Running at $(date)"; sleep 10; echo "Complete"'
          backoffLimit: 3
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 60
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

#### Creating CronJob with Concurrency

```yaml
spec:
  schedule: "0 * * * *"  # Hourly
  concurrencyPolicy: Allow
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: processor
              image: registry.redhat.io/ubi9/bash:latest
              command:
                - /bin/sh
                - -c
                - 'echo "Hourly task"; sleep 30; echo "Done"'
          restartPolicy: OnFailure
```

#### Creating CronJob with Timezone

```yaml
spec:
  schedule: "0 9 * * *"  # 9 AM UTC
  startingDeadlineSeconds: 120
```

#### Viewing CronJob Status

```bash
oc get cronjobs
oc get cronjob myapp-cronjob -n myapp-project
oc describe cronjob myapp-cronjob -n myapp-project
oc get pods -l batch.kubernetes.io/job-name=myapp-job -n myapp-project
```

---

## YAML Examples

### Service Account with Token

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: myapp-project
  annotations:
    description: "Service account for myapp"
```

### Pod with Privileged Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-app
  namespace: myapp-project
  labels:
    app: privileged
spec:
  serviceAccountName: privileged-sa
  securityContext:
    runAsUser: 0
    runAsGroup: 0
    fsGroup: 0
    privileged: true
    allowPrivilegeEscalation: true
  containers:
    - name: privileged-app
      image: registry.redhat.io/ubi9/ubi-minimal:latest
      command:
        - /bin/sh
        - -c
        - 'id; ls -la /proc/1/status | grep Cap'
      securityContext:
        privileged: true
        capabilities:
          add:
            - NET_ADMIN
            - SYS_ADMIN
  volumes:
    - name: host-logs
      hostPath:
        path: /var/log
        type: Directory
  hostPID: true
  hostNetwork: true
```

### SCC for Privileged Access

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: myapp-scc
  annotations:
    description: "SCC for myapp privileged access"
allowHostDirVolumePlugin: true
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: true
allowedCapabilities:
  - NET_ADMIN
  - SYS_ADMIN
defaultAddCapabilities: []
defaultAllowPrivilegeEscalation: false
fsGroup:
  type: RunAsAny
groups: []
readOnlyRootFilesystem: false
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
users:
  - system:serviceaccount:myapp-project:myapp-sa
volumes:
  - '*'
```

### Secret with Multiple Keys

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
  namespace: myapp-project
type: Opaque
stringData:
  database-url: "postgres://user:password@db.example.com:5432/myapp"
  api-key: "sk-1234567890abcdef"
  jwt-secret: "jwt-secret-key-12345"
  tls-cert: |
    -----BEGIN CERTIFICATE-----
    MIID...
    -----END CERTIFICATE-----
  tls-key: |
    -----BEGIN PRIVATE KEY-----
    MIIE...
    -----END PRIVATE KEY-----
```

### Custom ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: app-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
```

### RoleBinding for Service Account

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-binding
  namespace: myapp-project
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: myapp-project
roleRef:
  kind: ClusterRole
  name: app-reader
  apiGroup: rbac.authorization.k8s.io
```

### Job with Multiple Completions

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
  namespace: myapp-project
spec:
  completions: 5
  parallelism: 2
  template:
    metadata:
      labels:
        app: data-processor
    spec:
      restartPolicy: OnFailure
      containers:
        - name: processor
          image: registry.redhat.io/ubi9/bash:latest
          command:
            - /bin/sh
            - -c
            - |
              echo "Processing batch $(date)"
              sleep 30
              echo "Batch complete"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
      backoffLimit: 3
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 3600
```

### CronJob for Scheduled Processing

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
  namespace: myapp-project
  labels:
    app: backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 60
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    metadata:
      labels:
        app: backup
    spec:
      template:
        metadata:
          labels:
            app: backup
        spec:
          restartPolicy: OnFailure
          serviceAccountName: backup-sa
          containers:
            - name: backup
              image: registry.redhat.io/ubi9/bash:latest
              command:
                - /bin/sh
                - -c
                - |
                  echo "Starting backup at $(date)"
                  # Backup logic here
                  sleep 60
                  echo "Backup complete"
              resources:
                requests:
                  memory: "256Mi"
                  cpu: "200m"
              volumeMounts:
                - name: backup-storage
                  mountPath: /backup
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: backup-pvc
          backoffLimit: 3
```

---

## Explanation of YAML

### SCC Structure

The SCC defines pod security constraints:

- **`allowPrivilegedContainer`**: Allow privileged containers
- **`allowHost*`**: Allow host namespace/plugins
- **`allowedCapabilities`**: Linux capabilities to add
- **`runAsUser`**: User ID constraints (RunAsAny, MustRunAs, MustRunAsNonRoot)
- **`users`**: Service accounts allowed to use this SCC
- **`volumes`**: Allowed volume types

### Secret Types

- **Opaque**: Generic key-value pairs
- **kubernetes.io/tls**: TLS certificate and key
- **kubernetes.io/dockerconfigjson**: Docker registry auth
- **kubernetes.io/basic-auth**: HTTP basic auth
- **kubernetes.io/ssh-auth**: SSH authentication

### Job Fields

- **`completions`**: Number of successful pods required
- **`parallelism`**: Max concurrent pods
- **`backoffLimit`**: Number of retry attempts
- **`activeDeadlineSeconds`**: Maximum execution time
- **`ttlSecondsAfterFinished`**: Cleanup time after completion

### CronJob Fields

- **`schedule`**: Cron expression (minute hour day month week)
- **`concurrencyPolicy`**: Forbid, Replace, or Allow
- **`startingDeadlineSeconds`**: Deadline for first scheduled run
- **`successfulJobsHistoryLimit`**: Keep successful jobs
- **`failedJobsHistoryLimit`**: Keep failed jobs

---

## Verification Procedures

### Verify Service Account

```bash
oc get serviceaccount myapp-sa -n myapp-project
oc describe sa myapp-sa -n myapp-project
```

Expected: Service account exists with correct annotations.

### Verify SCC Assignment

```bash
oc describe pod -l app=myapp -n myapp-project | grep SCC
oc adm policy who-can -z myapp-sa -n myapp-project
```

Expected: Pod shows correct SCC, and permissions are granted.

### Verify Secret Creation

```bash
oc get secret myapp-secret -o yaml
oc get secret myapp-secret -o jsonpath='{.data.password}' | base64 -d && echo
```

Expected: Secret exists with correct data.

### Verify Job Completion

```bash
oc get jobs myapp-job -n myapp-project
oc describe job myapp-job -n myapp-project
```

Expected: Job shows completed status with correct completion count.

### Verify CronJob Schedule

```bash
oc get cronjob myapp-cronjob -n myapp-project
oc describe cronjob myapp-cronjob -n myapp-project
```

Expected: CronJob shows correct schedule and history.

### Verify API Access

```bash
oc adm policy can-I --as=system:serviceaccount:myapp-project:myapp-sa \
  get pods -n myapp-project
```

Expected: Returns `yes` for allowed actions.

---

## Troubleshooting

### Pod Creation Fails with SCC Violation

**Symptom**: Pod creation fails with "forbidden: security context constraint violation"

**Diagnosis**:

```bash
oc describe pod <pod-name> -n <namespace>
oc get scc
oc describe scc <scc-name>
```

**Resolution**:
- Check pod security context against SCC requirements
- Assign appropriate SCC to service account
- Modify pod security context to comply with SCC

### Service Account Token Not Mounted

**Symptom**: Application cannot access API using service account token.

**Diagnosis**:

```bash
oc exec <pod-name> -- ls -la /var/run/secrets/kubernetes.io/serviceaccount/
oc exec <pod-name> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

**Resolution**:
- Verify service account is correctly specified in pod spec
- Check token annotation is set
- Verify pod is running

### Secret Not Accessible in Pod

**Symptom**: Pod cannot read secret files or environment variables.

**Diagnosis**:

```bash
oc exec <pod-name> -- ls -la /var/run/secrets/
oc exec <pod-name> -- env | grep SECRET
```

**Resolution**:
- Verify secret name and namespace match
- Check secret key names match expected values
- Ensure secret is mounted correctly

### Job Not Completing

**Symptom**: Job stuck in Active state.

**Diagnosis**:

```bash
oc describe job <job-name> -n <namespace>
oc logs <pod-name> -n <namespace>
```

**Resolution**:
- Check pod logs for errors
- Verify command completes successfully
- Check resource constraints
- Review backoffLimit settings

### CronJob Not Scheduling

**Symptom**: CronJob never creates jobs.

**Diagnosis**:

```bash
oc describe cronjob <cronjob-name> -n <namespace>
oc get events -n <namespace>
```

**Resolution**:
- Verify Cron expression is correct
- Check startingDeadlineSeconds
- Review timezone configuration
- Check for scheduling conflicts

### Privileged Pod Fails

**Symptom**: Privileged pod creation fails.

**Diagnosis**:

```bash
oc describe pod <pod-name>
oc get scc
oc describe scc privileged
```

**Resolution**:
- Verify SCC allows required operations
- Check service account has SCC assignment
- Review host namespace permissions
- Verify security context matches SCC

---

## Production Best Practices

### Service Account Best Practices

- **Use dedicated service accounts**: One service account per application
- **Apply least privilege**: Grant only required API permissions
- **Rotate tokens**: Set appropriate token durations
- **Monitor usage**: Track service account API calls
- **Document permissions**: Maintain documentation of service account permissions

### SCC Best Practices

- **Use restricted SCC**: Default to restricted profile
- **Minimize privileged access**: Only grant when absolutely necessary
- **Audit SCC usage**: Regularly review SCC assignments
- **Document SCCs**: Maintain SCC documentation
- **Plan SCC removal**: Prepare for SCC deprecation

### Secret Management Best Practices

- **Use external secrets**: Consider HashiCorp Vault or AWS Secrets Manager
- **Rotate secrets**: Implement regular secret rotation
- **Mask secrets**: Don't log or display secrets
- **Use specific secret types**: Use kubernetes.io/tls for TLS, etc.
- **Avoid plaintext**: Never store plaintext passwords in secrets

### Job and CronJob Best Practices

- **Set timeouts**: Use activeDeadlineSeconds to prevent hanging
- **Monitor completion**: Track job success/failure rates
- **Clean up completed jobs**: Use TTLSecondsAfterFinished
- **Handle failures**: Implement retry logic with backoffLimit
- **Log output**: Ensure job output is captured and monitored

---

## Certification Exam Notes

### Exam-Relevant Commands You Must Know

```bash
# Service Account commands
oc create serviceaccount <name> -n <namespace>
oc get serviceaccounts
oc describe sa <name> -n <namespace>
oc create token <name> -n <namespace>

# SCC commands
oc get scc
oc describe scc <name>
oc adm policy add-scc-to-user <scc> -z <sa> -n <namespace>
oc adm policy remove-scc-from-user <scc> -z <sa> -n <namespace>

# Secret commands
oc create secret generic <name> --from-literal=key=value
oc create secret docker-registry <name> --docker-server=...
oc get secret <name> -o yaml
oc delete secret <name> -n <namespace>

# Role commands
oc apply -f clusterrole.yaml
oc apply -f role.yaml
oc apply -f rolebinding.yaml

# Job commands
oc get jobs
oc get job <name> -n <namespace>
oc describe job <name> -n <namespace>

# CronJob commands
oc get cronjobs
oc get cronjob <name> -n <namespace>
oc describe cronjob <name> -n <namespace>

# API access testing
oc adm policy can-I --as=<user> <verb> <resource> -n <namespace>
```

### Common Exam Scenarios

1. **"Create a service account"** → `oc create serviceaccount <name> -n <namespace>`
2. **"Generate a token for a service account"** → `oc create token <name> -n <namespace>`
3. **"Assign SCC to service account"** → `oc adm policy add-scc-to-user <scc> -z <sa> -n <namespace>`
4. **"Create a secret from literal values"** → `oc create secret generic <name> --from-literal=key=value`
5. **"Create a Job"** → `oc apply -f job.yaml`
6. **"Create a CronJob"** → `oc apply -f cronjob.yaml`
7. **"Check job status"** → `oc get jobs` or `oc describe job <name>`
8. **"Create a custom ClusterRole"** → `oc apply -f clusterrole.yaml`
9. **"Bind service account to role"** → `oc adm policy add-role-to-user <role> -z <sa>`
10. **"View SCC details"** → `oc describe scc <name>`

### Important Exam Details

- Service accounts are created with `oc create serviceaccount <name>`
- SCCs are assigned with `oc adm policy add-scc-to-user`
- Secrets are stored in etcd but displayed as base64-encoded
- Jobs require `restartPolicy: OnFailure`
- CronJobs use Cron syntax: `* * * * *` (minute hour day month week)
- `completions` specifies how many pods must succeed
- `parallelism` specifies max concurrent pods
- `backoffLimit` specifies retry attempts
- Token duration defaults to 1 hour

---

## Common Mistakes

### Mistake: Forgetting to Specify Service Account

Not specifying a service account in pod spec means the default service account is used. This may not have the required permissions.

### Mistake: Using Wrong SCC Name

SCC names are case-sensitive. Using the wrong SCC name causes pod creation to fail.

### Mistake: Not Setting RestartPolicy for Jobs

Jobs require `restartPolicy: OnFailure`. Without it, pods restart indefinitely.

### Mistake: Incorrect Cron Syntax

Cron syntax is `* * * * *` (minute, hour, day, month, week). Incorrect syntax causes scheduling failures.

### Mistake: Hard-Coding Secrets in Pod Spec

Secrets should be mounted as volumes or referenced via envFrom, not hard-coded in pod spec.

### Mistake: Not Setting Token Duration

Service account tokens default to 1 hour. For long-running applications, set `--duration` when creating tokens.

### Mistake: Forgetting to Create Secret in Same Namespace

Secrets must exist in the same namespace as the pod that references them.

### Mistake: Using Restricted SCC for Privileged Operations

The restricted SCC doesn't allow privileged containers. Use privileged SCC or custom SCC for privileged operations.

---

## Chapter Summary

This chapter covered configuring application security in OpenShift Container Platform. You learned to create and manage service accounts for application identities and API access. You understood Security Context Constraints (SCCs) and how they control pod-level security permissions. You configured privileged applications with appropriate SCC assignments. You created and applied secrets for sensitive data management, including opaque secrets, TLS secrets, and Docker registry secrets. You configured application access to Kubernetes APIs using Roles, ClusterRoles, and RoleBindings. You deployed Jobs for one-time batch processing tasks and CronJobs for scheduled recurring tasks. You diagnosed common security issues including SCC violations, token mounting failures, secret access problems, and job scheduling issues. Production best practices emphasized least privilege, regular audits, secret rotation, and proper resource management.

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Create service account | `oc create serviceaccount <name> -n <namespace>` |
| List service accounts | `oc get serviceaccounts` |
| Describe service account | `oc describe sa <name> -n <namespace>` |
| Create token | `oc create token <name> -n <namespace>` |
| Create secret (literal) | `oc create secret generic <name> --from-literal=key=value` |
| Create secret (file) | `oc create secret generic <name> --from-file=key=/path/to/file` |
| Create TLS secret | `oc create secret tls <name> --cert=file --key=file` |
| Create Docker secret | `oc create secret docker-registry <name> --docker-server=...` |
| List SCCs | `oc get scc` |
| Describe SCC | `oc describe scc <name>` |
| Assign SCC to user | `oc adm policy add-scc-to-user <scc> -z <sa>` |
| Remove SCC from user | `oc adm policy remove-scc-from-user <scc> -z <sa>` |
| Create Job | `oc apply -f job.yaml` |
| Create CronJob | `oc apply -f cronjob.yaml` |
| List jobs | `oc get jobs` |
| Describe job | `oc describe job <name>` |
| List cronjobs | `oc get cronjobs` |
| Describe cronjob | `oc describe cronjob <name>` |
| Test API access | `oc adm policy can-I --as=<user> <verb> <resource>` |
| Create ClusterRole | `oc apply -f clusterrole.yaml` |
| Create RoleBinding | `oc apply -f rolebinding.yaml` |

### SCC Types

| Type | Description |
|------|-------------|
| privileged | Full access to all resources |
| restricted | Default secure profile |
| runasany | Can run as any user |
| hostaccess | Full host access |
| custom | Custom SCC definition |

### Job Parameters

| Parameter | Description |
|-----------|-------------|
| `completions` | Number of successful pods required |
| `parallelism` | Max concurrent pods |
| `backoffLimit` | Number of retry attempts |
| `activeDeadlineSeconds` | Maximum execution time |
| `ttlSecondsAfterFinished` | Cleanup time after completion |
| `restartPolicy` | OnFailure (required for Jobs) |

### CronJob Schedule Syntax

| Field | Values |
|-------|--------|
| Minute | 0-59 |
| Hour | 0-23 |
| Day of month | 1-31 |
| Month | 1-12 |
| Day of week | 0-6 (Sunday=0) |

### Cron Examples

| Expression | Schedule |
|------------|----------|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour |
| `0 0 * * *` | Daily at midnight |
| `0 9 * * 1-5` | Weekdays at 9 AM |
| `0 0 1 * *` | Monthly on 1st |

---

## Review Questions

1. What is the difference between a user and a service account?

2. How do you create a service account and generate a token for it?

3. What is a Security Context Constraint (SCC)?

4. How do you assign an SCC to a service account?

5. What is the purpose of a Job in Kubernetes?

6. How does a CronJob differ from a Job?

7. What is the Cron syntax for running a task every 5 minutes?

8. How do you create a secret with multiple key-value pairs?

9. What is the difference between a Role and a ClusterRole?

10. How do you test if a service account has permission to perform an action?

11. What is the `backoffLimit` parameter in a Job?

12. How do you specify that a Job should run 3 pods in parallel?

13. What is the purpose of `activeDeadlineSeconds` in a Job?

14. What happens if a CronJob's `concurrencyPolicy` is set to `Forbid`?

15. How do you run a privileged container in OpenShift?

---

## Answers

1. A **user** is a human administrator who authenticates with credentials (password, token, identity provider). A **service account** is an application identity that uses tokens for API authentication and doesn't require user login.

2. Create with `oc create serviceaccount <name> -n <namespace>`, then generate a token with `oc create token <name> -n <namespace>`.

3. A Security Context Constraint (SCC) defines what a pod can do in OpenShift, including allowed capabilities, volumes, users, and security contexts. It replaces Kubernetes RBAC for pod-level security.

4. Use `oc adm policy add-scc-to-user <scc-name> -z <service-account-name> -n <namespace>`.

5. A Job runs a fixed number of successful pod completions. It's used for batch processing tasks that must complete successfully.

6. A Job runs once to completion. A CronJob schedules Jobs to run at specified intervals using Cron syntax.

7. `*/5 * * * *` - The `*/5` in the minute field means every 5 minutes.

8. `oc create secret generic <name> --from-literal=key1=value1 --from-literal=key2=value2`

9. A **Role** is namespace-scoped. A **ClusterRole** is cluster-scoped and can apply across all namespaces.

10. Use `oc adm policy can-I --as=system:serviceaccount:<namespace>:<service-account> <verb> <resource>`.

11. `backoffLimit` specifies how many times the Job controller will retry a failed pod before giving up.

12. Set `parallelism: 3` in the Job spec to allow up to 3 concurrent pods.

13. `activeDeadlineSeconds` specifies the maximum time (in seconds) a Job can run before being terminated.

14. With `Forbid`, the CronJob will not start a new job if another job is already running. It waits for the existing job to complete.

15. Use `oc adm policy add-scc-to-user privileged -z <service-account> -n <namespace>` to assign the privileged SCC, then create a pod with `privileged: true` in the security context.
