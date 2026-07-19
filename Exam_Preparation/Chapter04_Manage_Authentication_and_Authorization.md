# Chapter 4: Manage Authentication and Authorization

> **Exam baseline:** EX280 on OpenShift Container Platform 4.18.  
> **Reviewed:** 2026-07-19.  
> Use `oc auth can-i` for permission tests. A Kubernetes `ClusterRoleBinding` can reference only a `ClusterRole`; a namespaced `Role` can never be its `roleRef`.

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure the HTPasswd identity provider for authentication
- Create and delete users in OpenShift
- Modify user passwords
- Create and manage groups
- Modify user and group permissions using roles and role bindings
- Understand the difference between authentication and authorization
- Work with ClusterRoles, Roles, ClusterRoleBindings, and RoleBindings
- Troubleshoot authentication and authorization failures
- Apply the principle of least privilege in production environments

---

## OpenShift Concepts

### Authentication vs. Authorization

These two concepts are fundamentally different but work together:

- **Authentication** answers the question: *"Who are you?"* It verifies the identity of a user or service through credentials such as passwords, tokens, certificates, or external identity providers.
- **Authorization** answers the question: *"What are you allowed to do?"* It determines what actions an authenticated identity can perform against cluster resources.

OpenShift handles authentication through configured identity providers and the OAuth server. It handles authorization through Role-Based Access Control (RBAC), which defines roles with specific permissions and binds those roles to users, groups, or service accounts.

### Identity Providers

An identity provider (IdP) is an external system that OpenShift trusts to authenticate users. OpenShift supports multiple IdP types:

- **HTPasswd**: Simple file-based authentication with usernames and passwords stored in an HTPasswd file
- **LDAP**: Lightweight Directory Access Protocol for enterprise directory integration
- **Microsoft Active Directory**: Commonly integrated through the LDAP provider or an OpenID Connect provider; it is not a separate `type: ActiveDirectory` value
- **Keystone**: OpenStack identity service
- **GitHub / GitHub Enterprise**: OAuth-based authentication
- **Google**: Google OAuth authentication
- **OpenID Connect**: Standard OAuth 2.0 / OIDC provider integration
- **RequestHeader**: Header-based authentication (typically behind a reverse proxy)

Multiple identity providers can be configured simultaneously. The OAuth login flow lets the client select a provider; the list is not a password-authentication fallback chain. `mappingMethod` (`claim`, `lookup`, or `add`) controls how a provider identity maps to an OpenShift `User`. Use `add` carefully when different providers represent the same people.

### HTPasswd Identity Provider

HTPasswd is the simplest identity provider. It stores usernames and passwords in a plain text file with hashed passwords. Each line contains a username and a password hash separated by a colon:

```
username:$apr1$...
```

Use the `htpasswd` utility with bcrypt (`-B`) to generate entries; never store clear-text passwords in the file. HTPasswd is suitable for labs, small static user sets, and certification practice, not as a general enterprise identity lifecycle system.

### RBAC (Role-Based Access Control)

RBAC is the authorization model used by OpenShift. It consists of four resource types:

| Resource | Scope | Purpose |
|----------|-------|---------|
| `Role` | Namespace-scoped | Defines permissions within a single namespace |
| `ClusterRole` | Cluster-scoped | Defines permissions across the entire cluster |
| `RoleBinding` | Namespace-scoped | Binds a Role to a user, group, or service account within a namespace |
| `ClusterRoleBinding` | Cluster-scoped | Binds a ClusterRole to a user, group, or service account cluster-wide |

Each Role or ClusterRole contains **rules**. Each rule specifies:

- `apiGroups`: Which API groups the rule applies to (e.g., `""` for core, `"apps"` for apps)
- `resources`: Which resource types (e.g., `pods`, `deployments`, `services`)
- `verbs`: Which actions are allowed (e.g., `get`, `list`, `watch`, `create`, `update`, `delete`)

### Built-In Roles

OpenShift ships with predefined roles:

| Role | Scope | Description |
|------|-------|-------------|
| `admin` | Namespace | Full control over a namespace, cannot change resource limits |
| `edit` | Namespace | Create, update, delete most resources in a namespace |
| `view` | Namespace | Read-only access to a namespace |
| `cluster-admin` | Cluster | Unrestricted access to the entire cluster |
| `basic-user` | Cluster | Default permissions for all authenticated users |
| `self-access-review-creator` | Cluster | Create subjectaccessreviews for self |
| `system:deployer` | Namespace | Deploy applications |

### Users, Identities, and Groups

OpenShift maintains three related concepts:

- **User**: An internal OpenShift user object (`oc get user`). This is the canonical identity within the cluster.
- **Identity**: The link between a user and a specific identity provider (`oc get identity`). A single user can have multiple identities if they authenticate through multiple IdPs.
- **Group**: A collection of users (`oc get group`). Groups simplify permission management by allowing you to grant roles to multiple users at once.

With the usual `claim` mapping method, first login creates or maps the `User` and `Identity` objects. OpenShift does **not** add a user to a `Group` merely because a group name matches something. Manage OpenShift groups explicitly with `oc adm groups`, or use a supported external group-synchronization process.

---

## Architecture and Internal Operation

### Authentication Flow

```
1. User attempts to access OpenShift (console or CLI)
        |
        v
2. Request redirected to OAuth server (openshift-authentication namespace)
        |
        v
3. OAuth server checks for valid session token
        |
        ├── Token valid → Grant access
        |
        └── Token invalid/missing → Prompt for credentials
                |
                v
4. Credentials sent to configured Identity Provider(s) in order
        |
        v
5. IdP validates credentials
        |
        ├── Valid → OAuth server creates session token
        |
        └── Invalid → Try next IdP or reject
                |
                v
6. OAuth token exchanged for Kubernetes API token
        |
        v
7. User can access cluster with scoped permissions
```

### Authorization Flow

```
1. Authenticated user makes API request
        |
        v
2. API server extracts user identity from token
        |
        v
3. RBAC authorizer evaluates the request:
        |
        ├── Check ClusterRoleBindings (cluster-wide)
        |
        ├── Check RoleBindings (namespace-scoped)
        |
        ├── Match user, group, or service account
        |
        ├── Check if Role/ClusterRole grants the requested verb on the resource
        |
        ├── Granted → Allow request
        |
        └── Denied → Return 403 Forbidden
```

### HTPasswd File Management

The HTPasswd file is stored as a Secret in the `openshift-config` namespace. When you configure the HTPasswd identity provider:

1. You create an HTPasswd file locally with `htpasswd` utility
2. You create a Secret containing the file
3. You reference the Secret in the `OAuth` cluster configuration
4. The OAuth server watches the Secret for changes
5. The authentication components observe the updated Secret; allow time for reconciliation and verify a new login instead of assuming an exact delay

### Group Membership Resolution

An OpenShift `Group` explicitly lists OpenShift user names. Use `oc adm groups add-users` and `remove-users`, or a supported group-sync workflow for an external directory. Adding a user to a same-named group is not automatic. RBAC bindings that name the Group grant permissions to its current members; verify the current server decision with `oc auth can-i --as=<user> ...`.

---

## Components and Terminology

| Term | Description |
|------|-------------|
| Identity Provider (IdP) | External system that authenticates users |
| HTPasswd | File-based identity provider with hashed passwords |
| OAuth Server | OpenShift component that manages authentication sessions |
| RBAC | Role-Based Access Control authorization model |
| Role | Namespace-scoped set of permissions |
| ClusterRole | Cluster-scoped set of permissions |
| RoleBinding | Binds a Role to users/groups/service accounts in a namespace |
| ClusterRoleBinding | Binds a ClusterRole cluster-wide |
| User | Internal OpenShift user object |
| Identity | Link between a User and an Identity Provider |
| Group | Collection of users for permission management |
| `htpasswd` | Utility for creating and managing HTPasswd files |
| `cluster-admin` | Built-in role with unrestricted cluster access |
| `admin` | Built-in namespace-level administrative role |
| `edit` | Built-in namespace-level write role |
| `view` | Built-in namespace-level read-only role |
| SubjectAccessReview | API for programmatically checking permissions |

---

## Administration Tasks

### Configuring the HTPasswd Identity Provider

#### Installing the HTPasswd Utility

```bash
# On RHEL/CentOS
sudo dnf install httpd-tools

# On Ubuntu/Debian
sudo apt-get install apache2-utils
```

#### Creating an HTPasswd File

```bash
# Create file with first user; -c overwrites, -B selects bcrypt
htpasswd -c -B htpasswd.txt admin

# Add or update users; omit -c so existing entries remain
htpasswd -B htpasswd.txt developer
htpasswd -B htpasswd.txt viewer
```

The commands prompt for passwords, keeping them out of shell history and the process list. In an isolated exam lab, `-bB <file> <user> <password>` can be faster, but quote special characters and understand the exposure. The `-c` flag is only for first creation because it recreates the file.

#### Creating the HTPasswd Secret

```bash
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=htpasswd.txt \
  -n openshift-config
```

The secret must be created in the `openshift-config` namespace, and the key name must be `htpasswd`.

#### Configuring the Identity Provider

```bash
# Edit the OAuth configuration
oc edit oauth cluster
```

Add or modify the `identityProviders` section:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswd-secret
```

#### Verifying the Configuration

```bash
oc get oauth cluster -o yaml | grep -A 10 identityProviders
oc get secret htpasswd-secret -n openshift-config
```

### Creating and Managing Users

#### Automatic User Creation

With the common `mappingMethod: claim`, users and identity mappings are provisioned at first successful login. With `mappingMethod: lookup`, an administrator must pre-create the `User`, `Identity`, and `UserIdentityMapping`, for example `oc create user`, `oc create identity`, and `oc create useridentitymapping`. Read the configured mapping method before choosing a workflow.

#### Viewing Users

```bash
oc get users
oc get user <username>
oc describe user <username>
```

#### Viewing Identities

```bash
oc get identities
oc get identity <provider>:<username>
oc describe identity <provider>:<username>
```

#### Deleting a User and Its Login Identity

```bash
# First remove the user from groups and RBAC bindings that should no longer apply.
# Then delete the provider identity and the OpenShift User object.
oc delete identity <provider-name>:<provider-username>
oc delete user <username>
```

Deleting a `User` is not the same as removing the credential from the identity provider. For HTPasswd, remove the HTPasswd entry too. Clean up Group membership and RBAC bindings explicitly. If the task asks only to prevent future HTPasswd logins, removing the HTPasswd entry is the essential action; if it asks to delete the OpenShift account, remove the associated Identity and User objects as well.

#### Removing a User from HTPasswd

```bash
# Remove user from htpasswd file
htpasswd -D htpasswd.txt <username>

# Update the secret
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=htpasswd.txt \
  -n openshift-config \
  --dry-run=client -o yaml | oc replace -f -
```

### Modifying User Passwords

#### Using htpasswd Utility

```bash
# Change password for existing user
htpasswd -B htpasswd.txt <username>

# Update the secret in the cluster
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=htpasswd.txt \
  -n openshift-config \
  --dry-run=client -o yaml | oc replace -f -
```

#### Safe Update Starting from the Cluster Secret

```bash
# Retrieve the current file
oc get secret htpasswd-secret -n openshift-config \
  -o jsonpath='{.data.htpasswd}' | base64 --decode > htpasswd.txt

# Modify it with the standard Apache utility
htpasswd -B htpasswd.txt <username>

# Replace the existing Secret without deleting it first
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=htpasswd.txt \
  -n openshift-config \
  --dry-run=client -o yaml | oc replace -f -
```

There is no `htpasswdf` command included with `oc`. Use the `htpasswd` utility from `httpd-tools` (or the equivalent operating-system package).

### Creating and Managing Groups

#### Creating a Group

```bash
oc adm groups new <group-name>
```

#### Adding Users to a Group

```bash
oc adm groups add-users <group-name> <user1> <user2> <user3>
```

#### Removing Users from a Group

```bash
oc adm groups remove-users <group-name> <user1> <user2>
```

#### Viewing Groups

```bash
oc get groups
oc get group <group-name>
oc describe group <group-name>
```

#### Viewing Group Members

```bash
oc get group <group-name> -o yaml
oc get group <group-name> -o jsonpath='{.users[*].name}'
```

#### Deleting a Group

```bash
oc adm groups delete <group-name>
```

### Modifying User and Group Permissions

#### Granting a Built-In Role to a User

```bash
# Grant admin role in a specific namespace
oc adm policy add-role-to-user admin <username> -n <namespace>

# Grant edit role
oc adm policy add-role-to-user edit <username> -n <namespace>

# Grant view role
oc adm policy add-role-to-user view <username> -n <namespace>
```

#### Granting a Role to a Group

```bash
oc adm policy add-role-to-group edit <group-name> -n <namespace>
oc adm policy add-role-to-group view <group-name> -n <namespace>
```

#### Granting Cluster-Wide Permissions

```bash
# Grant cluster-admin role
oc adm policy add-cluster-role-to-user cluster-admin <username>

# Grant cluster-admin to a group
oc adm policy add-cluster-role-to-group cluster-admin <group-name>

# Grant a custom ClusterRole
oc adm policy add-cluster-role-to-user <cluster-role-name> <username>
```

#### Removing Permissions

```bash
# Remove role from user
oc adm policy remove-role-from-user admin <username> -n <namespace>

# Remove role from group
oc adm policy remove-role-from-group edit <group-name> -n <namespace>

# Remove cluster role from user
oc adm policy remove-cluster-role-from-user cluster-admin <username>

# Remove cluster role from group
oc adm policy remove-cluster-role-from-group cluster-admin <group-name>
```

#### Creating a Custom Role

```bash
oc apply -f custom-role.yaml
```

#### Binding a Custom Role

```bash
oc adm policy add-role-to-user <custom-role-name> <username> -n <namespace>
```

#### Viewing Current Permissions

```bash
# Roles assigned to a user in a namespace
oc adm policy who-can list pods -n <namespace>

# Check what a user can do
oc auth can-i list pods --as=<username> -n <namespace>
oc auth can-i create deployments.apps --as=<username> -n <namespace>

# View role bindings in a namespace
oc get rolebindings -n <namespace>
oc get clusterrolebindings
```

---

## YAML Examples

### OAuth Configuration with HTPasswd

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
identityProviders:
  - name: htpasswd
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswd-secret
```

### Custom Role (Namespace-Scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: myapp-project
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

### Custom ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-viewer
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get", "list"]
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: myapp-project
subjects:
  - kind: User
    name: developer1
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: deployment-viewer-binding
subjects:
  - kind: Group
    name: ops-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: deployment-viewer
  apiGroup: rbac.authorization.k8s.io
```

### Group Object

```yaml
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  name: developers
users:
  - developer1
  - developer2
  - developer3
nonResourceRules: null
```

### HTPasswd Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: htpasswd-secret
  namespace: openshift-config
type: Opaque
data:
  htpasswd: <base64-encoded-htpasswd-file-content>
```

---

## Explanation of YAML

### OAuth Configuration

The `OAuth` cluster configuration is a singleton resource named `cluster`. The `identityProviders` field is an array, allowing multiple providers. Each provider has:

- **`name`**: Display name shown during authentication
- **`type`**: Provider type (HTPasswd, LDAP, GitHub, etc.)
- **Provider-specific configuration**: For HTPasswd, the `htpasswd.fileData.name` references a Secret in `openshift-config`

When multiple providers exist, the login flow selects a provider. `mappingMethod` controls how that provider's identity maps to an OpenShift user; provider list order is not a credential fallback mechanism.

### Custom Role

The Role defines permissions scoped to a single namespace:

- **`apiGroups: [""]`**: The empty string refers to the core API group (pods, services, configmaps, etc.)
- **`resources: ["pods"]`**: The resource type the rule applies to
- **`verbs: ["get", "list", "watch"]`**: Allowed actions. `get` retrieves a single resource, `list` retrieves multiple resources, `watch` monitors for changes
- **`resources: ["pods/log"]`**: Sub-resources are specified with a `/` separator

Rules are additive. A user needs rules covering all the actions they must perform.

### RoleBinding

The RoleBinding connects subjects (users or groups) to a role:

- **`subjects`**: Array of users, groups, or service accounts
- **`roleRef`**: Reference to the Role being granted
- **`kind: User`**: Individual user binding
- **`kind: Group`**: Group binding (applies to all current and future group members)

Both a User and a Group can reference the same Role in a single binding.

### ClusterRoleBinding

A Kubernetes `ClusterRoleBinding` grants the referenced `ClusterRole` across the cluster. Its `roleRef.kind` must be `ClusterRole`; it cannot reference a namespaced `Role`. A namespaced `RoleBinding` can reference either a `Role` in that namespace or a `ClusterRole`, but its effect remains limited to that RoleBinding's namespace.

---

## Verification Procedures

### Verify HTPasswd Identity Provider Configuration

```bash
# Check IdP is configured
oc get oauth cluster -o yaml | grep -A 10 identityProviders

# Verify secret exists
oc get secret htpasswd-secret -n openshift-config

# Test authentication
oc login -u <username> https://api.cluster.example.com:6443
```

Expected: OAuth shows HTPasswd provider, secret exists, and login succeeds.

### Verify User Exists

```bash
oc get user <username>
oc get identity htpasswd:<username>
```

Expected: User and identity objects exist.

### Verify Group Membership

```bash
oc get group <group-name> -o yaml
oc get group <group-name> -o jsonpath='{.users[*].name}'
```

Expected: User appears in the group's user list.

### Verify Role Assignment

```bash
# Check namespace role bindings
oc get rolebindings -n <namespace>

# Check cluster role bindings
oc get clusterrolebindings

# Test specific permission
oc auth can-i create pods --as=<username> -n <namespace>
```

Expected: RoleBinding exists, and permission check returns `yes`.

### Verify Permission Denial

```bash
oc auth can-i delete deployments.apps --as=<username> -n <namespace>
```

Expected: Returns `no` for permissions not granted.

### Verify OAuth Server Health

```bash
oc get pods -n openshift-authentication
oc get pods -n openshift-oauth-apiserver
```

Expected: authentication-related pods are Ready. Also verify `oc get clusteroperator authentication` reports Available and not Degraded.

---

## Troubleshooting

### User Cannot Log In

**Symptom**: Login fails with "Invalid credentials" or authentication timeout.

**Diagnosis**:

```bash
# Check OAuth server pods
oc get pods -n openshift-authentication
oc get pods -n openshift-oauth-apiserver

# Check identity provider configuration
oc get oauth cluster -o yaml

# Check HTPasswd secret
oc get secret htpasswd-secret -n openshift-config -o yaml

# Check OAuth server logs
oc get pods -n openshift-authentication
oc logs -n openshift-authentication <oauth-pod-name> --all-containers
```

**Resolution**:
- Verify the HTPasswd secret exists and contains valid data
- Check that the username exists in the HTPasswd file
- Verify the password was hashed correctly
- Ensure OAuth server pods are running
- Check for multiple IdPs conflicting with each other

### User Cannot Perform an Action (403 Forbidden)

**Symptom**: User receives "Forbidden" errors when accessing resources.

**Diagnosis**:

```bash
# Check what the user can do
oc auth can-i <verb> <resource> --as=<username> -n <namespace>

# Check role bindings for the user
oc get rolebindings -n <namespace> -o yaml | grep -A 5 <username>

# Check cluster role bindings
oc get clusterrolebindings -o yaml | grep -A 5 <username>

# Check group membership
oc get groups -o yaml | grep -B 5 <username>

# Who has the required permission?
oc adm policy who-can <verb> <resource> -n <namespace>
```

**Resolution**:
- Grant the appropriate role: `oc adm policy add-role-to-user <role> <username> -n <namespace>`
- Verify the user is in the correct group
- Check that the Role contains the necessary rules (correct apiGroup, resource, and verb)
- Verify the RoleBinding references the correct Role name

### HTPasswd Password Change Not Taking Effect

**Symptom**: User cannot log in with new password after updating the HTPasswd file.

**Diagnosis**:

```bash
# Verify the secret was updated
oc get secret htpasswd-secret -n openshift-config -o yaml

# Check current resource version and decoded file hash
oc get secret htpasswd-secret -n openshift-config -o jsonpath='{.metadata.resourceVersion}'
oc get secret htpasswd-secret -n openshift-config \
  -o jsonpath='{.data.htpasswd}' | base64 --decode | sha256sum
```

**Resolution**:
- Retrieve the current file, modify it, and replace the Secret with `oc create ... --dry-run=client -o yaml | oc replace -f -`.
- Confirm the OAuth object references the same Secret name and that the key is named `htpasswd`.
- Wait for the Authentication Operator to reconcile and check `oc get clusteroperator authentication`. Patching a Secret with an empty object does not change its data and is not a repair.

### Group Membership Not Applied

**Symptom**: User is added to a group but does not inherit group permissions.

**Diagnosis**:

```bash
# Verify group object
oc get group <group-name> -o yaml

# Verify user is listed
oc get group <group-name> -o jsonpath='{.users[*].name}'

# Check role binding references the group
oc get rolebindings -n <namespace> -o yaml | grep -A 10 <group-name>
```

**Resolution**:
- OpenShift `Group` membership changes are stored in the cluster and should affect authorization without editing the IdP password file. Re-run `oc auth can-i ... --as=<user>` to test the server's current decision.
- Verify the RoleBinding subject `name` matches the group name exactly (case-sensitive)
- Check that the subject `kind` is `Group`, not `User`

### cluster-admin Cannot Perform an Action

**Symptom**: Even cluster-admin users receive permission denied errors.

**Diagnosis**:

```bash
# Confirm the identity and authorization result
oc whoami
oc auth can-i <verb> <resource> -n <namespace>

# Read the actual API rejection and object events
oc get events -n <namespace> --sort-by=.lastTimestamp

# Check resource quotas
oc get resourcequota -n <namespace>
```

**Resolution**:
- Separate authorization from admission and validation. A `Forbidden` message can name RBAC, SCC admission, quota, or another webhook; read the complete message.
- For pod creation, inspect the service account, requested security context, and SCC authorization instead of assuming the `cluster-admin` binding is missing.
- For object creation, verify quota, schema, namespace state, and admission webhook health.

### Multiple Identity Providers Causing Conflicts

**Symptom**: Users authenticated through one IdP cannot be found by roles assigned through another.

**Diagnosis**:

```bash
# Check all identities for a user
oc get identities

# Check user identity mappings
oc get user <username> -o yaml
```

**Resolution**:
- Each IdP creates provider-prefixed Identity objects. Decide deliberately whether identities should map to one User.
- Ensure RoleBindings reference the OpenShift user name, not an Identity object name.
- Review `mappingMethod`; `add` can attach another provider identity to an existing same-named user, while `claim` fails on a conflicting mapping and `lookup` requires manual mappings.
- Use groups for authorization policy after identity mapping is correct.

### OAuth Server Pod Crashing

**Symptom**: OAuth pods in CrashLoopBackOff.

**Diagnosis**:

```bash
oc get pods -n openshift-authentication
oc logs -n openshift-authentication <oauth-pod-name> --all-containers --previous
oc describe pod -n openshift-authentication <oauth-pod-name>
```

**Resolution**:
- Check for invalid OAuth configuration in `oc get oauth cluster -o yaml`
- Verify the HTPasswd secret format is correct
- Check for certificate expiration issues
- Review cluster operator status: `oc get clusteroperator authentication`

---

## Production Best Practices

### Authentication Best Practices

- Use enterprise identity providers (LDAP, Active Directory, OpenID Connect) in production instead of HTPasswd
- Configure only the identity providers you need and understand how users select them; provider list order is not a password fallback chain
- Set appropriate token expiration and session timeout values
- Enable audit logging for authentication events
- Rotate HTPasswd credentials regularly if HTPasswd is retained for a small lab or emergency human-login use case
- Use strong password hashing algorithms (bcrypt preferred over MD5 or SHA)

### Authorization Best Practices

- Apply the principle of least privilege: grant only the minimum permissions required
- Use groups instead of individual user bindings for easier management
- Create custom Roles for specific job functions rather than granting broad roles
- Avoid granting `cluster-admin` to individual users; use it only for break-glass access
- Regularly audit role assignments with `oc adm policy who-can`
- Use namespace-scoped Roles and RoleBindings whenever possible
- Document custom Roles with clear descriptions of their purpose

### User and Group Management

- Establish naming conventions for groups (e.g., `team-dev`, `team-ops`, `project-alpha-admins`)
- Review group membership periodically
- Remove permissions when users change roles or leave the organization
- Use service accounts for application-to-cluster communication, not user accounts
- Separate development, staging, and production access with different groups

### HTPasswd-Specific Practices

- Store the HTPasswd file securely with restricted file permissions
- Use bcrypt and let `htpasswd` prompt for the password: `htpasswd -B htpasswd.txt <user>`
- Back up the HTPasswd file before modifications
- Automate HTPasswd secret updates through CI/CD pipelines
- Limit HTPasswd to labs or a tightly controlled emergency human-login use case; applications use Kubernetes service accounts, not HTPasswd users

### RBAC Design Patterns

- **Layered roles**: Create base roles (view), then extend with additional permissions (edit, admin)
- **Namespace isolation**: Each project should have its own role bindings
- **Service account scoping**: Bind service account roles to specific namespaces only
- **Emergency access**: Maintain a small `cluster-admin` group for break-glass scenarios
- **Regular audits**: Run permission audits quarterly to remove stale bindings

---

## Certification Exam Notes

### Exam-Relevant Commands You Must Know

```bash
# HTPasswd identity provider
# First entry: -c creates the file; -B selects bcrypt and prompts securely
htpasswd -c -B htpasswd.txt <user>
htpasswd -B htpasswd.txt <user>
htpasswd -D htpasswd.txt <user>

# Secret management
oc create secret generic htpasswd-secret --from-file=htpasswd=htpasswd.txt -n openshift-config

# OAuth configuration
oc edit oauth cluster
oc get oauth cluster -o yaml

# User management
oc get users
oc get user <username>
oc delete user <username>
oc get identities
oc get identity <provider>:<username>

# Group management
oc adm groups new <group-name>
oc adm groups add-users <group-name> <user1> <user2>
oc adm groups remove-users <group-name> <user1>
oc adm groups delete <group-name>
oc get groups
oc get group <group-name> -o yaml

# Permission management
oc adm policy add-role-to-user <role> <username> -n <namespace>
oc adm policy add-role-to-group <role> <group-name> -n <namespace>
oc adm policy add-cluster-role-to-user <role> <username>
oc adm policy add-cluster-role-to-group <role> <group-name>
oc adm policy remove-role-from-user <role> <username> -n <namespace>
oc adm policy remove-role-from-group <role> <group-name> -n <namespace>
oc adm policy remove-cluster-role-from-user <role> <username>
oc adm policy remove-cluster-role-from-group <role> <group-name>

# Permission checking
oc auth can-i <verb> <resource> --as=<username> -n <namespace>
oc adm policy who-can <verb> <resource> -n <namespace>
oc get rolebindings -n <namespace>
oc get clusterrolebindings
```

### Common Exam Scenarios

1. **"Configure HTPasswd identity provider"** → Create htpasswd file, create secret in openshift-config, edit OAuth config
2. **"Add a user to the HTPasswd provider"** → `htpasswd -B htpasswd.txt user`, then replace the Secret
3. **"Change a user's password"** → `htpasswd -B htpasswd.txt user`, then replace the Secret
4. **"Create a group and add users"** → `oc adm groups new <group>` then `oc adm groups add-users <group> <users>`
5. **"Grant edit role to a user in a namespace"** → `oc adm policy add-role-to-user edit <user> -n <namespace>`
6. **"Grant cluster-admin to a user"** → `oc adm policy add-cluster-role-to-user cluster-admin <user>`
7. **"Revoke permissions from a user"** → `oc adm policy remove-role-from-user <role> <user> -n <namespace>`
8. **"Check if a user has permission"** → `oc auth can-i <verb> <resource> --as=<user> -n <namespace>`
9. **"Grant a role to a group"** → `oc adm policy add-role-to-group <role> <group> -n <namespace>`
10. **"Remove a user from a group"** → `oc adm groups remove-users <group> <user>`

### Important Exam Details

- The HTPasswd secret **must** be in the `openshift-config` namespace
- The secret key **must** be named `htpasswd`
- `oc adm policy` commands are the fastest way to manage permissions in the exam
- Built-in roles (`admin`, `edit`, `view`, `cluster-admin`) are always available
- Group membership is managed explicitly; verify its authorization effect with `oc auth can-i --as=<user>`
- `cluster-admin` is a ClusterRole, not a Role
- The `-c` flag in `htpasswd` creates a new file; only use it for the first user
- `oc auth can-i` submits an authorization review and reports `yes` or `no`; it does not perform the requested workload action

---

## Common Mistakes

### Mistake: Using `-c` Flag When Adding Subsequent Users

The `-c` flag in `htpasswd` creates (overwrites) the file. Using it when adding a second user deletes all existing users. Always omit `-c` when appending users to an existing file.

### Mistake: Creating the HTPasswd Secret in the Wrong Namespace

The HTPasswd secret must be in `openshift-config`, not in the current project or `openshift-authentication`. Creating it elsewhere causes the OAuth server to fail to find the credentials file.

### Mistake: Forgetting to Update the Secret After Modifying HTPasswd File

Modifying the local HTPasswd file does not automatically update the cluster. You must recreate or update the secret with the new file contents for changes to take effect.

### Mistake: Confusing Role with ClusterRole

Roles are namespace-scoped; ClusterRoles are cluster-scoped definitions. A RoleBinding may bind a Role or ClusterRole inside one namespace. A ClusterRoleBinding may bind only a ClusterRole and grants it cluster-wide.

### Mistake: Assuming Groups Populate Themselves

OpenShift does not add a user to a Group because names happen to match. Add/synchronize membership explicitly, inspect the Group, and test the current authorization decision.

### Mistake: Using Wrong Verb in Role Rules

Common verb mistakes include using `read` instead of `get,list,watch`, or forgetting that `*` (wildcard) grants all verbs. Be precise with verbs to follow least privilege.

### Mistake: Forgetting apiGroups in Role Rules

The `apiGroups` field determines which API the rule applies to. Core resources (pods, services) use `""`, while apps resources (deployments, replicasets) use `"apps"`. Omitting or mis-specifying the apiGroup causes the rule to not match.

### Mistake: Granting cluster-admin Instead of Namespace admin

`cluster-admin` provides unrestricted cluster access. For namespace-level administration, use the `admin` role with a RoleBinding. Granting `cluster-admin` when `admin` suffices violates least privilege.

---

## Chapter Summary

This chapter covered authentication and authorization in OpenShift Container Platform. You configured an HTPasswd identity provider, managed users, identities, and explicit group membership, and granted least-privilege access with Roles, ClusterRoles, RoleBindings, and ClusterRoleBindings. You used `oc auth can-i` for permission verification and learned to distinguish RBAC denials from SCC, quota, validation, and webhook admission failures.

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Create HTPasswd file (first user) | `htpasswd -c -B htpasswd.txt user` (enter password at prompt) |
| Add/update HTPasswd user | `htpasswd -B htpasswd.txt user` |
| Remove user from HTPasswd | `htpasswd -D htpasswd.txt user` |
| Create HTPasswd secret | `oc create secret generic htpasswd-secret --from-file=htpasswd=htpasswd.txt -n openshift-config` |
| Configure IdP | `oc edit oauth cluster` |
| View IdP config | `oc get oauth cluster -o yaml` |
| List users | `oc get users` |
| View user details | `oc get user <name>` |
| Delete user | `oc delete user <name>` |
| List identities | `oc get identities` |
| Create group | `oc adm groups new <group>` |
| Add users to group | `oc adm groups add-users <group> <user1> <user2>` |
| Remove users from group | `oc adm groups remove-users <group> <user>` |
| Delete group | `oc adm groups delete <group>` |
| Grant role to user | `oc adm policy add-role-to-user <role> <user> -n <ns>` |
| Grant role to group | `oc adm policy add-role-to-group <role> <group> -n <ns>` |
| Grant cluster role to user | `oc adm policy add-cluster-role-to-user <role> <user>` |
| Grant cluster role to group | `oc adm policy add-cluster-role-to-group <role> <group>` |
| Remove role from user | `oc adm policy remove-role-from-user <role> <user> -n <ns>` |
| Remove role from group | `oc adm policy remove-role-from-group <role> <group> -n <ns>` |
| Remove cluster role from user | `oc adm policy remove-cluster-role-from-user <role> <user>` |
| Check permission | `oc auth can-i <verb> <resource> --as=<user> -n <ns>` |
| Who can do action | `oc adm policy who-can <verb> <resource> -n <ns>` |
| View role bindings | `oc get rolebindings -n <ns>` |
| View cluster role bindings | `oc get clusterrolebindings` |

### Built-In Roles

| Role | Scope | Description |
|------|-------|-------------|
| `cluster-admin` | Cluster | Full unrestricted access |
| `admin` | Namespace | Full namespace control (except quotas) |
| `edit` | Namespace | Create/update/delete most resources |
| `view` | Namespace | Read-only access |
| `basic-user` | Cluster | Default for all authenticated users |

### Common RBAC Verbs

| Verb | Action |
|------|--------|
| `get` | Retrieve a single resource |
| `list` | Retrieve multiple resources |
| `watch` | Monitor for changes |
| `create` | Create a new resource |
| `update` | Update an existing resource |
| `patch` | Partially update a resource |
| `delete` | Delete a resource |
| `*` | All verbs (wildcard) |

### Common API Groups

| apiGroups Value | Resources |
|----------------|-----------|
| `""` (empty string) | pods, services, configmaps, secrets, namespaces |
| `"apps"` | deployments, replicasets, daemonsets, statefulsets |
| `"batch"` | jobs, cronjobs |
| `"rbac.authorization.k8s.io"` | roles, rolebindings, clusterroles, clusterrolebindings |
| `"route.openshift.io"` | routes |
| `"image.openshift.io"` | imagestreams, imagetags |
| `"*" ` | All API groups (wildcard) |

### HTPasswd Hashing Algorithms

| Flag | Algorithm | Security Level |
|------|-----------|---------------|
| (default on some utilities) | MD5/APR variant | Legacy; avoid |
| `-s` | SHA-1 | Legacy; avoid |
| `-B` | bcrypt | High (recommended) |

---

## Review Questions

1. What is the difference between authentication and authorization in OpenShift?

2. Why must the HTPasswd secret be created in the `openshift-config` namespace?

3. What happens when a user authenticates through HTPasswd for the first time?

4. How do you grant the `edit` role to a user named `developer1` in the `myapp-project` namespace?

5. What is the difference between a Role and a ClusterRole?

6. How do you check if a user has permission to create pods in a namespace?

7. How do you prove that new group membership grants the intended permission?

8. What command removes the `cluster-admin` role from a user?

9. What is the difference between `oc adm policy add-role-to-user` and `oc adm policy add-cluster-role-to-user`?

10. How do you find out which users have permission to delete deployments in a namespace?

11. What does the `apiGroups: [""]` field mean in a Role rule?

12. Can a ClusterRoleBinding reference a namespace-scoped Role?

13. What happens if you use the `-c` flag with `htpasswd` when adding a second user?

14. How do you add a user to an existing group?

15. What are the four RBAC resource types in OpenShift?

---

## Answers

1. **Authentication** verifies who a user is (identity verification through credentials). **Authorization** determines what an authenticated user is allowed to do (permission enforcement through RBAC roles and bindings).

2. The OAuth server in OpenShift is configured to look for HTPasswd secrets specifically in the `openshift-config` namespace. This is a fixed location defined by the OpenShift authentication operator. Placing the secret elsewhere means the OAuth server cannot find the credentials.

3. With `mappingMethod: claim`, OpenShift creates or maps an `Identity` named from the provider and login identity to a `User`. Group membership is separate and must already be managed explicitly.

4. `oc adm policy add-role-to-user edit developer1 -n myapp-project`

5. A **Role** is namespace-scoped and defines permissions within a single namespace. A **ClusterRole** is cluster-scoped and can define permissions across the entire cluster, including cluster-level resources like nodes and namespaces. ClusterRoles can be bound at either the namespace level (via RoleBinding) or cluster level (via ClusterRoleBinding).

6. `oc auth can-i create pods --as=<username> -n <namespace>`. This creates an authorization review and returns `yes` or `no` without creating a pod.

7. Inspect `oc get group <group> -o yaml`, then run `oc auth can-i <verb> <resource> --as=<username> -n <namespace>` and confirm it returns `yes` for the intended rule.

8. `oc adm policy remove-cluster-role-from-user cluster-admin <username>`

9. `add-role-to-user` grants a namespace-scoped Role and creates a RoleBinding in the specified namespace. `add-cluster-role-to-user` grants a ClusterRole and creates a ClusterRoleBinding with cluster-wide effect. The former is limited to one namespace; the latter applies across the entire cluster.

10. `oc adm policy who-can delete deployments -n <namespace>`. This lists all users, groups, and service accounts that have the specified permission.

11. The empty string `""` refers to the **core API group**, which includes fundamental resources like pods, services, configmaps, secrets, and namespaces. No prefix is needed for these resources.

12. No. A Kubernetes ClusterRoleBinding's `roleRef.kind` must be `ClusterRole`. Bind a namespaced Role with a RoleBinding, or bind a ClusterRole with either a namespaced RoleBinding or a ClusterRoleBinding depending on the required scope.

13. The `-c` flag creates (overwrites) the file. All previously existing users are deleted, leaving only the newly added user. This is a data-destructive operation.

14. `oc adm groups add-users <group-name> <username>`

15. **Role** (namespace-scoped permissions), **ClusterRole** (cluster-scoped permissions), **RoleBinding** (binds roles to subjects in a namespace), and **ClusterRoleBinding** (binds roles to subjects cluster-wide).

---

## Verified References

- EX280 objectives: <https://www.redhat.com/en/services/training/red-hat-certified-openshift-administrator-exam>
- OpenShift 4.18 authentication, identity providers, users, groups, and RBAC: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/index>
- OpenShift 4.18 Kubernetes RBAC API behavior: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/rbac_apis/index>
