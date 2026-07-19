# Chapter 2: Work with Resource Manifests

> **Edition note:** Commands and resource behavior in this chapter are reviewed for OpenShift Container Platform 4.18, the version currently named for EX280. Always read the exam environment instructions before beginning a task.

## Learning Objectives

By the end of this chapter, you will be able to:

- Deploy applications from YAML resource manifests using `oc apply` and `oc create`
- Update application deployments through manifest edits and `oc set` commands
- Deploy applications using Kustomize base configurations
- Work with Kustomize overlays to customize deployments across environments
- Create and use Secrets for sensitive data management
- Create and use ConfigMaps for application configuration
- Understand the differences between imperative and declarative resource management
- Manage resource lifecycle through manifest-driven workflows

---

## OpenShift Concepts

### Declarative vs. Imperative Management

OpenShift supports two approaches to resource management:

- **Imperative**: Direct commands that perform an action immediately, such as `oc run nginx --image=nginx`. The resulting API object can persist, but the command does not leave you with a version-controlled desired-state file unless you generate and save one.
- **Declarative**: YAML manifests that describe the desired state of resources. Applying these manifests with `oc apply` tells OpenShift what the cluster should look like, and the control plane works to reconcile reality to that desired state.

Declarative management is the foundation of GitOps, infrastructure-as-code, and reproducible deployments. The exam emphasizes declarative workflows.

### Resource Manifests

A resource manifest is a YAML (or JSON) file that describes a Kubernetes or OpenShift resource. Every manifest contains three top-level fields:

- `apiVersion`: Which API group and version the resource belongs to
- `kind`: The type of resource (Deployment, Service, ConfigMap, Secret, etc.)
- `metadata`: Identifying information including name and namespace

Manifests can contain a single resource or multiple resources separated by `---`.

### Client-Side Apply and Server-Side Apply

Plain `oc apply -f manifest.yaml` normally uses **client-side apply**. The client calculates a three-way merge from the live object, the new manifest, and the `kubectl.kubernetes.io/last-applied-configuration` annotation, then sends a patch to the API server. The API server still performs authentication, authorization, schema validation, admission, and persistence.

`oc apply --server-side -f manifest.yaml` uses **server-side apply**. The API server performs field management and records managers in `metadata.managedFields`. Server-side apply can report field conflicts when two managers own the same field. Do not assume ordinary `oc apply` prevents every overwrite: preview the diff and use one clear source of truth.

### Kustomize

Kustomize is a manifest customization tool built into the `oc` CLI. It allows you to maintain a base set of manifests and then apply environment-specific patches without duplicating YAML files. Kustomize uses a `kustomization.yaml` file to define:

- Which base resources to include
- Patches to apply
- Common labels and annotations
- Name prefixes and suffixes
- Replacements and generated ConfigMaps or Secrets

Kustomize eliminates the need for third-party templating engines for most use cases.

### Secrets

Secrets store sensitive data such as passwords, tokens, and keys. Values in the `data` field are base64-encoded; **base64 is encoding, not encryption**. Access must be restricted with RBAC, and encryption at rest should be configured where required. Secrets can be mounted as volumes or exposed as environment variables in pods. OpenShift supports multiple Secret types:

- `Opaque`: Generic secret (default)
- `kubernetes.io/tls`: TLS certificate and key pairs
- `kubernetes.io/docker-configjson`: Container registry authentication
- `kubernetes.io/basic-auth`: HTTP basic authentication credentials

### ConfigMaps

ConfigMaps store non-sensitive configuration data as key-value pairs or entire files. Like Secrets, they can be mounted as volumes or exposed as environment variables. ConfigMaps separate configuration from image content, enabling the same container image to run across environments with different configuration.

---

## Architecture and Internal Operation

### Manifest Processing Pipeline

When a manifest is applied, the following sequence occurs:

```
User runs `oc apply -f manifest.yaml`
        |
        v
oc parses the input and builds an API request
        |
        v
Request sent to kube-apiserver over HTTPS
        |
        v
API server validates against OpenAPI schema
        |
        v
Admission controllers evaluate the request
  (SCC validation, quota enforcement, resource limits)
        |
        v
Object stored in etcd
        |
        v
Controller manager detects change via informer
        |
        v
Relevant controller reconciles actual state to desired state
```

Admission controllers are a critical checkpoint. They enforce Security Context Constraints, resource quotas, limit ranges, and other cluster policies before the object is persisted.

### ConfigMap and Secret Distribution

When a ConfigMap or Secret is mounted into a pod, the following occurs:

1. The kubelet on the node detects the pod specification references a ConfigMap or Secret
2. The kubelet retrieves the object from the API server
3. The content is written to a temporary directory on the node
4. The directory is mounted into the container at the specified path
5. The kubelet periodically refreshes projected ConfigMap and Secret volumes; propagation is eventually consistent and can take time
6. An environment variable is fixed when the container starts, and a `subPath` volume mount does not receive automatic updates

This means ConfigMaps and Secrets must exist in the **same namespace** as the pod that references them.

### Kustomize Execution Model

Kustomize operates entirely client-side. When you run `oc apply -k <directory>`:

1. Kustomize reads the `kustomization.yaml` file
2. It loads all referenced base resources
3. It applies patches, labels, annotations, and name transformations
4. It produces a merged YAML stream
5. The merged output is sent to `oc apply` as if it were a single manifest file

No Kustomize-specific data is stored in the cluster. The API server only sees standard Kubernetes resources.

---

## Components and Terminology

| Term | Description |
|------|-------------|
| Manifest | A YAML or JSON file describing a desired cluster resource |
| `oc apply` | Declaratively creates or updates resources from a manifest |
| `oc create` | Imperatively creates a new resource (fails if it already exists) |
| `oc set` | Imperatively modifies specific fields of a resource |
| Kustomize | Client-side manifest customization tool |
| `kustomization.yaml` | Kustomize configuration file defining bases, patches, and transforms |
| Base | Unmodified resource manifests representing a default configuration |
| Overlay | Environment-specific modifications applied on top of a base |
| Patch | A partial resource definition used to modify a base resource |
| Strategic Merge Patch | Patch type that merges maps recursively and replaces lists by matching keys |
| JSON Patch | Patch type using RFC 6902 operations (add, remove, replace) |
| Secret | Kubernetes resource for storing sensitive data |
| ConfigMap | Kubernetes resource for storing non-sensitive configuration data |
| Environment Variable injection | Populating container env vars from ConfigMaps or Secrets |
| Volume mount | Mounting ConfigMaps or Secrets as files inside containers |

---

## Administration Tasks

### Deploying Applications from YAML Manifests

#### Single-Resource Manifest

```bash
oc apply -f deployment.yaml
```

#### Multi-Resource Manifest

```bash
oc apply -f app-stack.yaml
```

A multi-resource manifest contains multiple documents separated by `---`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
```

#### Applying from a URL

```bash
oc apply -f https://raw.githubusercontent.com/example/app/main/deployment.yaml
```

#### Applying from a Directory

```bash
oc apply -f /path/to/manifests/
```

This applies manifest files in that directory. Add `--recursive` (or `-R`) when manifests are also stored in subdirectories:

```bash
oc apply -R -f /path/to/manifests/
```

#### Dry Run

```bash
# Validate without applying
oc apply -f deployment.yaml --dry-run=client -o yaml

# Server-side validation without persisting
oc apply -f deployment.yaml --dry-run=server
```

### Creating Resources Imperatively

#### Creating a Deployment from a Command

```bash
oc create deployment nginx-app \
  --image=registry.redhat.io/ubi9/nginx-124
```

#### Creating a ConfigMap from Literal Values

```bash
oc create configmap app-config \
  --from-literal=db_host=db.example.com \
  --from-literal=db_port=5432 \
  --from-literal=log_level=info
```

#### Creating a ConfigMap from a File

```bash
oc create configmap nginx-config \
  --from-file=nginx.conf=/etc/nginx/nginx.conf
```

#### Creating a Secret from Literal Values

```bash
oc create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='S3cureP@ss!'
```

This quoting prevents shell interpretation, but a literal can still appear in shell history and the process list. For real credentials, prefer a protected input file, an approved secret-management workflow, or another method that does not place the value in command arguments.

#### Creating a Secret from a File

```bash
oc create secret generic tls-cert \
  --from-file=tls.crt=/path/to/cert.pem \
  --from-file=tls.key=/path/to/key.pem
```

#### Creating a Secret with Specific Type

```bash
oc create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

### Updating Application Deployments

#### Updating via Manifest Edit

```bash
# Edit the deployment interactively in your default editor
oc edit deployment/myapp
```

This opens the full resource YAML. Modify the desired fields and save. The change is applied immediately.

#### Updating Replicas

```bash
oc scale deployment/myapp --replicas=5
```

#### Updating Container Image

```bash
oc set image deployment/myapp nginx=registry.redhat.io/ubi9/nginx-124:1.24-9
```

The format is `<container-name>=<new-image>`. The container name must match the name defined in the deployment's pod template.

#### Updating Resource Limits

```bash
oc set resources deployment/myapp \
  --containers=nginx \
  --requests=cpu=200m,memory=256Mi \
  --limits=cpu=500m,memory=512Mi
```

#### Rolling Back a Deployment

```bash
# View revision history
oc rollout history deployment/myapp

# Roll back to previous revision
oc rollout undo deployment/myapp

# Roll back to specific revision
oc rollout undo deployment/myapp --to-revision=3
```

#### Monitoring a Rollout

```bash
oc rollout status deployment/myapp
oc rollout status deployment/myapp --timeout=120s
```

### Deploying Applications Using Kustomize

#### Project Structure

```
app-manifests/
├── kustomization.yaml
├── deployment.yaml
├── service.yaml
└── configmap.yaml
```

#### Base `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

labels:
  - pairs:
      app: myapp
      managed-by: kustomize
    includeSelectors: true
```

#### Applying Kustomize

```bash
oc apply -k ./app-manifests/
```

The `-k` flag tells `oc` to process the directory through Kustomize before applying.

### Working with Kustomize Overlays

#### Directory Structure

```
app-manifests/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   └── replica-patch.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── replica-patch.yaml
│       └── resource-limits-patch.yaml
```

#### Base `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

#### Development Overlay `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: dev-

replicas:
  - name: myapp
    count: 1

patches:
  - path: replica-patch.yaml
```

#### Production Overlay `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-

replicas:
  - name: myapp
    count: 5

patches:
  - path: replica-patch.yaml
  - path: resource-limits-patch.yaml
```

#### Strategic Merge Patch Example (`replica-patch.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: nginx
          env:
            - name: ENV_NAME
              value: "production"
```

#### Applying Specific Overlay

```bash
# Deploy development configuration
oc apply -k ./app-manifests/overlays/development/

# Deploy production configuration
oc apply -k ./app-manifests/overlays/production/
```

#### Viewing Kustomize Output Without Applying

```bash
oc kustomize ./app-manifests/overlays/production/
```

This renders the final merged YAML to stdout without applying it to the cluster.

### Creating and Managing Secrets

#### Encoding Values for Manual Secret Creation

```bash
echo -n 'admin' | base64
echo -n 'S3cureP@ss!' | base64
```

#### Creating a Secret via YAML

```bash
oc apply -f secret.yaml
```

#### Inspecting a Secret

```bash
oc get secret db-credentials -o yaml
oc get secret db-credentials -o jsonpath='{.data.username}' | base64 -d
```

#### Rotating a Secret

```bash
# Update the secret
oc apply -f secret-updated.yaml

# Restart pods to pick up the new secret
oc rollout restart deployment/myapp
```

### Creating and Managing ConfigMaps

#### Inspecting a ConfigMap

```bash
oc get configmap app-config -o yaml
oc get configmap app-config -o jsonpath='{.data.db_host}'
```

#### Updating a ConfigMap

```bash
oc edit configmap/app-config
```

Or reapply with updated values:

```bash
oc create configmap app-config --from-literal=db_host=newdb.example.com --from-literal=log_level=debug --dry-run=client -o yaml | oc apply -f -
```

---

## YAML Examples

### Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp-project
  labels:
    app: myapp
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
    spec:
      containers:
        - name: nginx
          image: registry.redhat.io/ubi9/nginx-124:latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
          volumeMounts:
            - name: config-volume
              mountPath: /etc/nginx/conf.d
              readOnly: true
      volumes:
        - name: config-volume
          configMap:
            name: nginx-configmap
```

### ConfigMap Manifest

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp-project
data:
  db_host: "db.example.com"
  db_port: "5432"
  log_level: "info"
  app_env: "production"
```

### ConfigMap with File Content

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configmap
  namespace: myapp-project
data:
  default.conf: |
    server {
        listen 8080;
        server_name _;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        location /health {
            return 200 'healthy';
            add_header Content-Type text/plain;
        }
    }
```

### Secret Manifest

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: myapp-project
type: Opaque
data:
  db_username: YWRtaW4=
  db_password: U0NlY3JlUEBzczE=
```

### Multi-Resource Manifest

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp-project
data:
  db_host: "db.example.com"
  log_level: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: myapp-project
type: Opaque
data:
  db_username: YWRtaW4=
  db_password: U0NlY3JlUEBzczE=
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp-project
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: registry.redhat.io/ubi9/nginx-124:latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp-project
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

### Kustomize Base Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: registry.redhat.io/ubi9/nginx-124:latest
          ports:
            - containerPort: 8080
```

### Kustomize Strategic Merge Patch

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  template:
    spec:
      containers:
        - name: nginx
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          env:
            - name: ENV_NAME
              value: "production"
```

### Kustomize JSON Patch

```yaml
op: replace
path: /spec/template/spec/containers/0/image
value: registry.redhat.io/ubi9/nginx-124:1.24-9
```

---

## Explanation of YAML

### Deployment with ConfigMap and Secret Integration

This deployment demonstrates production-grade patterns:

- **`spec.strategy`**: Configures rolling update behavior. `maxSurge: 1` allows one extra pod during updates. `maxUnavailable: 0` ensures zero downtime during rollouts.
- **`envFrom`**: Injects all key-value pairs from a ConfigMap or Secret as environment variables. This is cleaner than listing each variable individually when many configuration values exist.
- **`volumeMounts`**: Mounts the `nginx-configmap` ConfigMap as files under `/etc/nginx/conf.d`, allowing nginx configuration to be managed externally from the container image.
- **`volumes`**: Defines the source of the mounted volume. The `configMap.name` field references the ConfigMap that must exist in the same namespace.

### ConfigMap with File Content

The `default.conf` key contains a multi-line string using the YAML `|` (literal block) indicator. When mounted as a volume, each key becomes a file, and the value becomes the file content. This enables managing entire configuration files through ConfigMaps.

### Secret Manifest

The `data` field contains base64-encoded values. `YWRtaW4=` decodes to `admin` and `U0NlY3JlUEBzczE=` decodes to `S3cureP@ss!`. For plaintext values during development, you can use the `stringData` field instead, which accepts unencoded strings:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  db_username: admin
  db_password: S3cureP@ss!
```

The API server automatically encodes `stringData` values when storing the Secret.

### Kustomize Patch

The strategic merge patch only includes the fields that differ from the base. Kustomize matches resources by `apiVersion`, `kind`, and `metadata.name`, then recursively merges the provided fields. This means you only specify what changes, not the entire resource.

---

## Verification Procedures

### Verify Deployment from Manifest

```bash
oc get deployment myapp
oc get pods -l app=myapp
oc get svc myapp-service
```

Expected: Deployment shows correct replica count, pods are Running, and service has a ClusterIP.

### Verify ConfigMap Injection

```bash
oc exec <pod-name> -- env | grep db_host
oc exec <pod-name> -- cat /etc/nginx/conf.d/default.conf
```

Expected: Environment variables reflect ConfigMap values, and the nginx configuration file is present.

### Verify Secret Injection

```bash
oc exec <pod-name> -- env | grep db_username
```

Expected: Secret values appear as environment variables. Note: Secret values are never displayed by `oc describe` for security.

### Verify Kustomize Deployment

```bash
# Check applied resources have expected labels
oc get all -l app=myapp

# Verify name prefix was applied
oc get pods -l app=myapp

# Verify replica count matches overlay
oc get deployment -l app=myapp
```

### Verify Rolling Update Completed

```bash
oc rollout status deployment/myapp
oc rollout history deployment/myapp
```

Expected: Rollout completes successfully, and revision history shows the new revision.

### Verify Secret Encoding

```bash
oc get secret app-secrets -o jsonpath='{.data.db_password}' | base64 -d && echo
```

Expected: Outputs the decoded password value.

---

## Troubleshooting

### Manifest Fails to Apply

**Symptom**: `oc apply -f manifest.yaml` returns an error.

**Diagnosis**:

```bash
# Apply with verbose output
oc apply -f manifest.yaml -v=9

# Check for validation errors
oc apply -f manifest.yaml --dry-run=server -v=8
```

**Common Causes**:
- Invalid YAML syntax (indentation errors, missing fields)
- Resource already exists with a different owner (use `oc replace` or delete first)
- Insufficient permissions for the current user
- SCC denial preventing the pod from running
- Resource quota exceeded in the project

### ConfigMap or Secret Not Found in Pod

**Symptom**: Pod stuck in `CreateContainerConfigError`.

**Diagnosis**:

```bash
oc describe pod <pod-name>
```

Look for events like `Error: configmaps "app-config" not found`.

**Resolution**:
- Verify the ConfigMap/Secret exists in the same namespace: `oc get configmap app-config`
- Verify the name matches exactly (case-sensitive)
- Check for typos in the manifest reference

### Secret Base64 Decode Errors

**Symptom**: `oc apply` fails with `invalid character` or base64 decoding errors.

**Diagnosis**:

```bash
# Verify encoding
echo "YWRtaW4=" | base64 -d

# Check for newlines in encoded values
oc get secret app-secrets -o jsonpath='{.data.db_password}' | base64 -d
```

**Resolution**:
- Use `stringData` instead of `data` to avoid manual encoding
- Ensure base64 values do not contain trailing newlines
- Use `echo -n 'value' | base64` (the `-n` prevents newline inclusion)

### Kustomize Fails to Render

**Symptom**: `oc apply -k` returns errors about missing resources or invalid patches.

**Diagnosis**:

```bash
# Render without applying to isolate the issue
oc kustomize ./path/to/overlay/

# Inspect the exact file and resource paths used by the overlay
oc kustomize ./path/to/overlay/
```

**Common Causes**:
- Relative path errors in overlay `kustomization.yaml`
- Patch references a resource name that does not exist in the base
- Strategic merge patch has incorrect `apiVersion` or `kind`
- JSON patch has invalid RFC 6902 syntax

### Deployment Rollout Fails

**Symptom**: `oc rollout status` reports a failure or times out.

**Diagnosis**:

```bash
oc rollout status deployment/myapp --timeout=300s
oc describe deployment/myapp
oc get pods -l app=myapp
```

**Resolution**:
- Check if new pods are failing to start (image pull errors, SCC denial)
- Verify the old pods can terminate gracefully (check `terminationGracePeriodSeconds`)
- Check if `maxUnavailable: 0` combined with resource constraints prevents new pods from scheduling
- Undo the rollout: `oc rollout undo deployment/myapp`

### Env Variable Not Appearing in Container

**Symptom**: `oc exec` shows the environment variable is missing.

**Diagnosis**:

```bash
oc describe pod <pod-name> | grep -A 20 "Environment Variables from"
oc get configmap app-config -o yaml
```

**Resolution**:
- Verify the ConfigMap key name matches exactly
- Check that `envFrom` references the correct ConfigMap name
- If using individual `env` entries, verify `valueFrom.configMapKeyRef.name` and `key` fields
- Restart the workload if the value is consumed through an environment variable. A normal ConfigMap or Secret volume updates eventually; an environment variable and a `subPath` mount do not.

### Multi-Resource Manifest Partial Failure

**Symptom**: Only some resources in a multi-resource manifest are created.

**Diagnosis**:

```bash
# Apply with --validate flag
oc apply -f multi-resource.yaml --validate=true

# Check each resource individually
oc apply -f multi-resource.yaml --dry-run=server -o name
```

**Resolution**:
- Fix the failing resource and reapply the entire manifest
- Consider splitting large multi-resource manifests into smaller files
- Remember that a multi-document apply is not a transaction: already-created objects are not rolled back when a later object fails

---

## Production Best Practices

### Manifest Organization

- Store manifests in version control (Git) with clear directory structure
- Separate base configurations from environment-specific overlays
- Use one logical resource per file for readability, or group related resources
- Name files after the resource they contain: `deployment.yaml`, `service.yaml`, `configmap.yaml`
- Avoid committing Secrets to version control; use sealed secrets, external vaults, or generate them during CI/CD

### Secret Management

- Never store plaintext passwords in manifests checked into Git
- Use `stringData` in manifests for local development only; rotate before production deployment
- Apply the principle of least privilege: create minimal Secrets scoped to specific applications
- Rotate Secrets regularly and restart affected pods
- Consider using an external secrets operator (HashiCorp Vault, AWS Secrets Manager) for production
- Use the supported OpenShift API-server encryption-at-rest procedure when policy requires it; first check the cluster's current encryption configuration and operational requirements

### ConfigMap Best Practices

- Keep ConfigMaps small; excessively large ConfigMaps consume API server memory
- Use ConfigMaps for configuration, not for large data files
- Validate ConfigMap keys — they must be valid file names if mounted as volumes
- Use `envFrom` for bulk injection and individual `env` entries for selective injection
- Set up ConfigMap update monitoring to detect configuration drift

### Kustomize Best Practices

- Keep base manifests minimal and generic
- Use overlays for environment differences (replicas, resource limits, image tags)
- Use `namePrefix` for environment identification rather than duplicating names
- Use the `labels` transformer to tag resources by environment or team; decide deliberately whether selectors and pod templates should also change
- Test Kustomize output with `oc kustomize` before applying to production
- Avoid deep nesting of overlay directories; two levels (base/overlay) is sufficient for most cases

### Update Strategies

- Use rolling updates, readiness probes, enough capacity, and an appropriate `maxUnavailable` value to reduce interruption; no single field can guarantee zero downtime
- Set appropriate `terminationGracePeriodSeconds` for graceful shutdown
- Test manifests in a non-production environment before applying to production
- Use `--dry-run=server` to validate manifests before actual application
- Maintain rollback capability by tracking manifest versions in Git

### Resource Quotas and Limits

- Define realistic resource requests for scheduling and capacity planning. Set limits when the workload and platform policy require them; test CPU limits carefully because unnecessary throttling can hurt performance.
- Set requests slightly above baseline usage and limits at maximum expected usage
- Monitor actual resource consumption and adjust requests/limits accordingly
- Use LimitRange objects to set defaults for containers that omit resource specifications

---

## Certification Exam Notes

### Exam-Relevant Commands You Must Know

```bash
# Deploy from manifest
oc apply -f <file.yaml>
oc apply -f <directory>/

# Deploy from Kustomize
oc apply -k <directory>/

# Create resources imperatively
oc create configmap <name> --from-literal=key=value
oc create secret generic <name> --from-literal=key=value
oc create secret tls <name> --cert=<file> --key=<file>

# Update deployments
oc edit deployment/<name>
oc set image deployment/<name> <container>=<image>
oc set resources deployment/<name> --containers=<name> --requests=... --limits=...
oc scale deployment/<name> --replicas=<count>

# Rollout management
oc rollout status deployment/<name>
oc rollout history deployment/<name>
oc rollout undo deployment/<name>

# Inspect ConfigMaps and Secrets
oc get configmap <name> -o yaml
oc get secret <name> -o yaml
oc get secret <name> -o jsonpath='{.data.<key>}' | base64 -d

# Kustomize rendering
oc kustomize <directory>/

# Dry run validation
oc apply -f <file> --dry-run=client -o yaml
oc apply -f <file> --dry-run=server
```

### Common Exam Scenarios

1. **"Deploy an application from a YAML file"** → `oc apply -f /path/to/manifest.yaml`
2. **"Update the image in a deployment"** → `oc set image deployment/<name> <container>=<new-image>`
3. **"Create a ConfigMap from literal values"** → `oc create configmap <name> --from-literal=key1=val1 --from-literal=key2=val2`
4. **"Create a Secret and use it in a deployment"** → Create the Secret, then reference it in the deployment via `env` or `envFrom`
5. **"Scale a deployment to N replicas"** → `oc scale deployment/<name> --replicas=N`
6. **"Roll back a failed deployment"** → `oc rollout undo deployment/<name>`
7. **"Use Kustomize to deploy with an overlay"** → `oc apply -k ./overlays/<environment>/`
8. **"View the decoded value of a Secret"** → `oc get secret <name> -o jsonpath='{.data.<key>}' | base64 -d`
9. **"Check rollout status"** → `oc rollout status deployment/<name>`
10. **"Add an environment variable to a deployment"** → `oc set env deployment/<name> KEY=value`

### Important Exam Details

- Secrets are **base64-encoded**, not encrypted, by default. Know how to encode and decode.
- ConfigMaps and Secrets must be in the **same namespace** as the consuming pod.
- `oc apply` is **idempotent** — running it multiple times produces the same result.
- `oc create` will **fail** if the resource already exists.
- Kustomize patches match resources by `apiVersion`, `kind`, and `metadata.name`.
- `oc edit` opens the resource in your terminal editor (usually `vi`). Know basic `vi` commands.
- Rolling updates create a new ReplicaSet and gradually replace old pods.

---

## Common Mistakes

### Mistake: Using `oc create` After Resource Exists

`oc create -f manifest.yaml` fails with `already exists` if the resource is already deployed. Use `oc apply` instead, which handles both creation and updates. Reserve `oc create` for one-time resource creation where you are certain the resource does not exist.

### Mistake: Forgetting Namespace in Manifest

If a manifest does not specify `metadata.namespace`, the resource is created in the **current default project**. This causes confusion when the same manifest is applied from different contexts. Always include the namespace explicitly in production manifests.

### Mistake: Base64 Encoding with Trailing Newlines

Running `echo 'password' | base64` includes a trailing newline in the encoded value. Use `echo -n 'password' | base64` to avoid this. The trailing newline causes authentication failures when the Secret is consumed.

### Mistake: ConfigMap Key Names with Invalid Characters

ConfigMap keys used as volume mounts become file names. Keys containing `/`, `..`, or other filesystem-invalid characters will cause pod startup failures. Use alphanumeric characters, hyphens, and underscores only.

### Mistake: Not Restarting Pods After Secret Update

Updating a Secret does not automatically propagate to running pods when mounted as environment variables. Pods must be restarted to pick up new Secret values. Use `oc rollout restart deployment/<name>` after updating a Secret.

### Mistake: Kustomize Path Errors

Kustomize overlays reference base resources using relative paths. An incorrect relative path causes `unable to find one of 'fn' or 'behavior'` or resource-not-found errors. Always verify paths from the overlay's `kustomization.yaml` location, not from the project root.

### Mistake: Strategic Merge Patch List Behavior

Strategic merge behavior depends on the Kubernetes schema's merge metadata. For example, the container list is normally merged by `name`, while lists without a merge key may be replaced. There is no `patchStrategicMerge` key inside a resource. Put patch references in `kustomization.yaml` and always inspect the result with `oc kustomize`.

### Mistake: Confusing `env` with `envFrom`

- `env` defines individual environment variables with explicit names and values
- `envFrom` injects all key-value pairs from a ConfigMap or Secret as environment variables
- Mixing both can cause conflicts if keys overlap — `env` takes precedence

---

## Chapter Summary

This chapter covered the complete workflow of working with resource manifests in OpenShift. You learned the distinction between declarative and imperative management and why `oc apply` is the preferred method for production deployments. You deployed applications from single and multi-resource YAML manifests, updated deployments using `oc set`, `oc edit`, and `oc scale`, and managed rolling updates and rollbacks. You mastered Kustomize for environment-specific deployments, creating base configurations and overlays with strategic merge and JSON patches. You created and managed Secrets for sensitive data and ConfigMaps for application configuration, including both imperative creation and YAML-based deployment. You learned how ConfigMaps and Secrets are injected into containers as environment variables and volume mounts, and how to troubleshoot common issues including manifest application failures, missing configuration, base64 encoding errors, and Kustomize rendering problems.

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Apply manifest | `oc apply -f <file.yaml>` |
| Apply directory | `oc apply -f <directory>/` |
| Apply Kustomize | `oc apply -k <directory>/` |
| Render Kustomize | `oc kustomize <directory>/` |
| Dry run (client) | `oc apply -f <file> --dry-run=client -o yaml` |
| Dry run (server) | `oc apply -f <file> --dry-run=server` |
| Edit resource | `oc edit deployment/<name>` |
| Set image | `oc set image deployment/<name> <container>=<image>` |
| Set env var | `oc set env deployment/<name> KEY=value` |
| Set resources | `oc set resources deployment/<name> --containers=<c> --requests=cpu=200m,memory=256Mi` |
| Scale deployment | `oc scale deployment/<name> --replicas=N` |
| Rollout status | `oc rollout status deployment/<name>` |
| Rollout history | `oc rollout history deployment/<name>` |
| Rollback | `oc rollout undo deployment/<name>` |
| Create ConfigMap (literal) | `oc create configmap <name> --from-literal=key=value` |
| Create ConfigMap (file) | `oc create configmap <name> --from-file=key=/path/to/file` |
| Create Secret (literal) | `oc create secret generic <name> --from-literal=key=value` |
| Create TLS Secret | `oc create secret tls <name> --cert=file.crt --key=file.key` |
| View ConfigMap | `oc get configmap <name> -o yaml` |
| View Secret | `oc get secret <name> -o yaml` |
| Decode Secret value | `oc get secret <name> -o jsonpath='{.data.<key>}' \| base64 -d` |
| Base64 encode | `echo -n 'value' \| base64` |
| Base64 decode | `echo 'encoded' \| base64 -d` |
| Restart pods | `oc rollout restart deployment/<name>` |

### Kustomize Key Fields

| Field | Purpose |
|-------|---------|
| `resources` | List of base manifest files or directories to include |
| `namePrefix` | Prefix added to all resource names |
| `nameSuffix` | Suffix added to all resource names |
| `labels` | Label transformer; `includeSelectors` and `includeTemplates` control where labels propagate |
| `commonAnnotations` | Annotations applied to all resources |
| `replicas` | Override replica counts for Deployments |
| `images` | Override container images |
| `patches` | List of patch files (strategic merge or JSON) |
| `patchesStrategicMerge` | Inline strategic merge patches (deprecated, use `patches`) |
| `patchesJson6902` | JSON patches with target specification |

### ConfigMap and Secret Injection Patterns

| Pattern | YAML Snippet |
|---------|-------------|
| Single env from ConfigMap | `env: - name: KEY\n  valueFrom:\n    configMapKeyRef:\n      name: my-config\n      key: mykey` |
| All env from ConfigMap | `envFrom:\n  - configMapRef:\n      name: my-config` |
| Single env from Secret | `env: - name: KEY\n  valueFrom:\n    secretKeyRef:\n      name: my-secret\n      key: mykey` |
| All env from Secret | `envFrom:\n  - secretRef:\n      name: my-secret` |
| ConfigMap as volume | `volumes:\n  - name: cfg\n    configMap:\n      name: my-config` |
| Secret as volume | `volumes:\n  - name: sec\n    secret:\n      secretName: my-secret` |

---

## Review Questions

1. What is the difference between `oc apply` and `oc create`, and when should each be used?

2. How do you update the container image in an existing deployment without editing the full manifest?

3. What happens when a ConfigMap referenced by a pod does not exist?

4. Explain the difference between `env` and `envFrom` in a container specification.

5. How do you decode a base64-encoded Secret value from the command line?

6. What is the purpose of `oc kustomize` versus `oc apply -k`?

7. What are strategic merge patches and how do they differ from JSON patches in Kustomize?

8. How do you perform a zero-downtime deployment update?

9. What command would you use to roll back a deployment to its previous revision?

10. Why must ConfigMaps and Secrets exist in the same namespace as the pod that references them?

11. What is the difference between the `data` and `stringData` fields in a Secret manifest?

12. How do you validate a manifest without actually applying it to the cluster?

13. What does the `oc rollout status` command do, and when should you use it?

14. How do you add a common label to all resources in a Kustomize configuration?

15. After updating a Secret, why might running pods still show the old value, and how do you fix it?

---

## Answers

1. `oc apply` is declarative and idempotent — it creates resources if they do not exist and updates them if they do. `oc create` is imperative and fails if the resource already exists. Use `oc apply` for production manifests and `oc create` for one-time resource creation or imperative commands like `oc create configmap --from-literal`.

2. Use `oc set image deployment/<name> <container-name>=<new-image>`. This updates only the image field without modifying other deployment settings.

3. The pod enters `CreateContainerConfigError` state and will not start. The kubelet cannot resolve the ConfigMap reference, and the error appears in `oc describe pod` events.

4. `env` defines individual environment variables with explicit `name` and `value` (or `valueFrom`) fields. `envFrom` injects all key-value pairs from an entire ConfigMap or Secret as environment variables, where each key becomes the variable name and each value becomes the variable value.

5. `oc get secret <name> -o jsonpath='{.data.<key>}' | base64 -d`. The `jsonpath` extracts the encoded value, and `base64 -d` decodes it.

6. `oc kustomize <dir>` renders the final merged YAML to stdout without applying it. `oc apply -k <dir>` renders and applies the resources to the cluster. Use `oc kustomize` for preview and debugging.

7. Strategic merge patches merge YAML structures recursively, matching maps by key and lists by defined merge keys (e.g., containers by `name`). JSON patches use RFC 6902 operations (`add`, `remove`, `replace`) with explicit JSON paths. Strategic merge is more intuitive for Kubernetes resources; JSON patches offer precise path-based control.

8. Configure the deployment with `strategy.type: RollingUpdate`, set `maxUnavailable: 0` to ensure existing pods remain running until new pods are ready, and set `maxSurge` to allow extra pods during the transition. Then update the image and monitor with `oc rollout status`.

9. `oc rollout undo deployment/<name>` rolls back to the previous revision. To roll back to a specific revision: `oc rollout undo deployment/<name> --to-revision=<number>`.

10. ConfigMaps and Secrets are namespace-scoped resources. The kubelet resolves references using the pod's namespace context. A ConfigMap in a different namespace is invisible to the pod's kubelet.

11. The `data` field requires base64-encoded values. The write-only convenience field `stringData` accepts plain strings, which the API server converts into `data`. If the same key appears in both fields, the `stringData` value takes precedence. Do not commit either form of a real password to Git.

12. Use `oc apply -f <file> --dry-run=client -o yaml` for client-side validation (syntax and structure) or `oc apply -f <file> --dry-run=server` for server-side validation (including admission control checks) without persisting changes.

13. `oc rollout status` monitors the progress of a deployment rollout and reports when all replicas have been successfully updated. Use it after any deployment change (image update, scale, configuration change) to confirm the rollout completed.

14. Add a `labels` transformer to `kustomization.yaml`. Use `includeSelectors: true` only when the label should also become part of selectors:
```yaml
labels:
  - pairs:
      environment: production
      team: platform
    includeSelectors: true
```

15. Secrets mounted as environment variables are baked into the container at startup and do not update automatically. Mounted volume Secrets do update, but with a delay. To immediately apply new Secret values, restart the pods: `oc rollout restart deployment/<name>`.

---

## Verified References

- OpenShift Container Platform 4.18, *Building applications* (projects, manifests, ConfigMaps, Secrets, and deployment workflows): <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/building_applications/index>
- Kubernetes, *Declarative Management of Kubernetes Objects Using Configuration Files*: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/>
- Kubernetes, *Kustomize*: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/>
- Kubernetes, *Secrets*: <https://kubernetes.io/docs/concepts/configuration/secret/>
- Kubernetes, *ConfigMaps*: <https://kubernetes.io/docs/concepts/configuration/configmap/>
