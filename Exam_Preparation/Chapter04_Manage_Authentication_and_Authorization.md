# Chapter 4: Manage Authentication and Authorization

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
- **Active Directory**: Microsoft directory service integration
- **Keystone**: OpenStack identity service
- **GitHub / GitHub Enterprise**: OAuth-based authentication
- **Google**: Google OAuth authentication
- **OpenID Connect**: Standard OAuth 2.0 / OIDC provider integration
- **Token**: Static token-based authentication
- **RequestHeader**: Header-based authentication (typically behind a reverse proxy)

Multiple identity providers can be configured simultaneously. OpenShift tries them in order until one successfully authenticates the user.

### HTPasswd Identity Provider

HTPasswd is the simplest identity provider. It stores usernames and passwords in a plain text file with hashed passwords. Each line contains a username and a password hash separated by a colon:

```
username:$apr1$...
```

The passwords must be hashed using one of the supported algorithms: MD5, SHA, bcrypt, or APR1. HTPasswd is ideal for small teams, development environments, and certification exams.

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

When a user authenticates through an IdP for the first time, OpenShift automatically creates the User and Identity objects. If a group with a matching name exists, the user is added to that group automatically.

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
5. When the Secret is updated, new credentials take effect automatically (typically within seconds)

### Group Membership Resolution

Group membership is resolved at authentication time:

1. User authenticates through an IdP
2. OpenShift checks if the IdP returns group attributes
3. OpenShift also checks static Group objects configured in the cluster
4. The user's effective groups are the union of both sources
5. RoleBindings and ClusterRoleBindings evaluate against the effective group list

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
| `htpasswdf` | OpenShift CLI plugin for managing HTPasswd users |
| `cluster-admin` | Built-in role with unrestricted cluster access |
| `admin` | Built-in namespace-level administrative role |
| `edit` | Built-in namespace-level write role |
| `view` | Built-in namespace-level read-only role |
| SubjectAccessReview | API for programmatically checking permissions |
| SAPassword | HTPasswd authentication plugin for `oc` CLI |

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
# Create file with first user (-c creates the file, -b for batch mode)
htpasswd --bbc htpasswd.txt admin AdminPassword1!

# Add additional users (-b for batch, omit -c to append)
htpasswd -b htpasswd.txt developer DevPassword1!
htpasswd -b htpasswd.txt viewer ViewPassword1!
```

The `-b` flag allows providing the password on the command line. The `-c` flag should only be used for the first user, as it recreates the file.

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
identityProviders:
  - name: htpasswd
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

Users are created automatically when they authenticate through a configured identity provider for the first time. No manual user creation command is needed.

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

#### Deleting a User

```bash
oc delete user <username>
```

Note: Deleting a user does not remove their identities or group memberships automatically. You may need to clean up associated identities and group memberships manually.

#### Removing a User from HTPasswd

```bash
# Remove user from htpasswd file
htpasswd -D htpasswd.txt <username>

# Update the secret
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=htpasswdd.txt \
  -n openshift-config \
  --dry-run=client -o yaml | oc apply -f -
```

### Modifying User Passwords

#### Using htpasswd Utility

```bash
# Change password for existing user
htpasswd -b htpasswd.txt <username> NewPassword1!

# Update the secret in the cluster
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=htpasswd.txt \
  -n openshift-config \
  --dry-run=client -o yaml | oc apply -f -
```

#### Using the htpasswdf Plugin

```bash
# Install the plugin (if not already available)
# The htpasswdf plugin is included with the oc CLI in OpenShift 4

# Add or update a user password
htpasswdf modify htpasswd.txt <username> <new-password>

# Remove a user
htpasswdf delete htpasswd.txt <username>
```

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
oc adm policy can-I --as=<username> list pods -n <namespace>
oc adm policy can-I --as=<username> create deployments -n <namespace>

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

When multiple providers exist, OpenShift attempts authentication in the order listed.

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

Identical in structure to RoleBinding, but grants cluster-wide permissions. ClusterRoleBindings can reference either ClusterRoles or Roles. When a Role is referenced in a ClusterRoleBinding, the permissions are still namespace-scoped but apply across all namespaces.

---

## Verification Procedures

### Verify HTPasswd Identity Provider Configuration

```bash
# Check IdP is configured
oc get oauth cluster -o yaml | grep -A 10 identityProviders

# Verify secret exists
oc get secret htpasswd-secret -n openshift-config

# Test authentication
oc login -u <username> -p <password> https://api.cluster.example.com:6443
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
oc adm policy can-I --as=<username> create pods -n <namespace>
```

Expected: RoleBinding exists, and permission check returns `yes`.

### Verify Permission Denial

```bash
oc adm policy can-I --as=<username> delete deployments -n <namespace>
```

Expected: Returns `no` for permissions not granted.

### Verify OAuth Server Health

```bash
oc get pods -n openshift-authentication
oc get pods -n openshift-oauth-apiserver
```

Expected: All authentication pods are Running.

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
oc logs -n openshift-authentication deploy/oauth-apiserver
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
oc adm policy can-I --as=<username> <verb> <resource> -n <namespace>

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

# Check secret modification time
oc get secret htpasswd-secret -n openshift-config -o jsonpath='{.metadata.creationTimestamp}'
oc get secret htpasswd-secret -n openshift-config -o jsonpath='{.metadata.resourceVersion}'
```

**Resolution**:
- Ensure the updated secret was applied with `oc apply` or `oc create --dry-run=client -o yaml | oc apply -f -`
- The OAuth server watches for secret changes; it may take 30-60 seconds to propagate
- Force a refresh by patching the secret: `oc patch secret htpasswd-secret -n openshift-config -p='{}'`

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
- Group membership is evaluated at authentication time; the user must re-authenticate
- Verify the RoleBinding subject `name` matches the group name exactly (case-sensitive)
- Check that the subject `kind` is `Group`, not `User`

### cluster-admin Cannot Perform an Action

**Symptom**: Even cluster-admin users receive permission denied errors.

**Diagnosis**:

```bash
# Check if another admission controller is blocking the action
oc get pods -n openshift-apiserver
oc logs -n openshift-apiserver cluster-version-operator

# Check SCC constraints
oc get scc

# Check resource quotas
oc get resourcequota -n <namespace>
```

**Resolution**:
- cluster-admin bypasses RBAC but not SCC, resource quotas, or admission controllers
- Check if the action violates a Security Context Constraint
- Verify resource quotas are not exceeded
- Check for custom admission controllers blocking the action

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
- Each IdP creates separate Identity objects for the same user
- Ensure RoleBindings reference the correct user name (not the identity)
- Use groups to abstract away IdP-specific identity differences
- Order IdPs carefully in the OAuth configuration to avoid unintended authentication

### OAuth Server Pod Crashing

**Symptom**: OAuth pods in CrashLoopBackOff.

**Diagnosis**:

```bash
oc get pods -n openshift-authentication
oc logs -n openshift-authentication deploy/oauth-apiserver --previous
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
- Configure multiple identity providers with fallback ordering
- Set appropriate token expiration and session timeout values
- Enable audit logging for authentication events
- Rotate HTPasswd passwords regularly if used as a fallback provider
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
- Use bcrypt hashing: `htpasswd -nbB <user> <password>`
- Back up the HTPasswd file before modifications
- Automate HTPasswd secret updates through CI/CD pipelines
- Limit HTPasswd to emergency and service accounts in production; use enterprise IdPs for regular users

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
htpasswd -bbc htpasswd.txt <user> <password>
htpasswd -b htpasswd.txt <user> <password>
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
oc adm policy can-I --as=<username> <verb> <resource> -n <namespace>
oc adm policy who-can <verb> <resource> -n <namespace>
oc get rolebindings -n <namespace>
oc get clusterrolebindings
```

### Common Exam Scenarios

1. **"Configure HTPasswd identity provider"** → Create htpasswd file, create secret in openshift-config, edit OAuth config
2. **"Add a user to the HTPasswd provider"** → `htpasswd -b htpasswd.txt user pass`, update secret
3. **"Change a user's password"** → `htpasswd -b htpasswd.txt user newpass`, update secret
4. **"Create a group and add users"** → `oc adm groups new <group>` then `oc adm groups add-users <group> <users>`
5. **"Grant edit role to a user in a namespace"** → `oc adm policy add-role-to-user edit <user> -n <namespace>`
6. **"Grant cluster-admin to a user"** → `oc adm policy add-cluster-role-to-user cluster-admin <user>`
7. **"Revoke permissions from a user"** → `oc adm policy remove-role-from-user <role> <user> -n <namespace>`
8. **"Check if a user has permission"** → `oc adm policy can-I --as=<user> <verb> <resource> -n <namespace>`
9. **"Grant a role to a group"** → `oc adm policy add-role-to-group <role> <group> -n <namespace>`
10. **"Remove a user from a group"** → `oc adm groups remove-users <group> <user>`

### Important Exam Details

- The HTPasswd secret **must** be in the `openshift-config` namespace
- The secret key **must** be named `htpasswd`
- `oc adm policy` commands are the fastest way to manage permissions in the exam
- Built-in roles (`admin`, `edit`, `view`, `cluster-admin`) are always available
- Group membership is evaluated at **authentication time** — users must re-login after group changes
- `cluster-admin` is a ClusterRole, not a Role
- The `-c` flag in `htpasswd` creates a new file; only use it for the first user
- `oc adm policy can-I` tests permissions without making actual API calls

---

## Common Mistakes

### Mistake: Using `-c` Flag When Adding Subsequent Users

The `-c` flag in `htpasswd` creates (overwrites) the file. Using it when adding a second user deletes all existing users. Always omit `-c` when appending users to an existing file.

### Mistake: Creating the HTPasswd Secret in the Wrong Namespace

The HTPasswd secret must be in `openshift-config`, not in the current project or `openshift-authentication`. Creating it elsewhere causes the OAuth server to fail to find the credentials file.

### Mistake: Forgetting to Update the Secret After Modifying HTPasswd File

Modifying the local HTPasswd file does not automatically update the cluster. You must recreate or update the secret with the new file contents for changes to take effect.

### Mistake: Confusing Role with ClusterRole

Roles are namespace-scoped; ClusterRoles are cluster-scoped. Binding a Role with a ClusterRoleBinding applies the namespace-scoped permissions across all namespaces, which can be confusing. Use ClusterRoles for truly cluster-wide permissions.

### Mistake: Not Re-Authenticating After Group Changes

Group membership is evaluated at authentication time. Adding a user to a group does not immediately grant permissions to existing sessions. The user must log out and log back in.

### Mistake: Using Wrong Verb in Role Rules

Common verb mistakes include using `read` instead of `get,list,watch`, or forgetting that `*` (wildcard) grants all verbs. Be precise with verbs to follow least privilege.

### Mistake: Forgetting apiGroups in Role Rules

The `apiGroups` field determines which API the rule applies to. Core resources (pods, services) use `""`, while apps resources (deployments, replicasets) use `"apps"`. Omitting or mis-specifying the apiGroup causes the rule to not match.

### Mistake: Granting cluster-admin Instead of Namespace admin

`cluster-admin` provides unrestricted cluster access. For namespace-level administration, use the `admin` role with a RoleBinding. Granting `cluster-admin` when `admin` suffices violates least privilege.

---

## Chapter Summary

This chapter covered authentication and authorization in OpenShift Container Platform. You learned the distinction between authentication (identity verification) and authorization (permission enforcement). You configured the HTPasswd identity provider, including creating the password file, storing it as a secret, and referencing it in the OAuth configuration. You managed users, identities, and groups — understanding how they relate and how group membership is resolved at authentication time. You worked extensively with RBAC, granting and revoking permissions using built-in roles and custom roles through RoleBindings and ClusterRoleBindings. You used `oc adm policy` commands for efficient permission management and `oc adm policy can-I` for permission verification. Troubleshooting covered login failures, permission denials, secret update propagation, and OAuth server issues. Production best practices emphasized least privilege, group-based management, regular audits, and enterprise identity provider integration.

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Create HTPasswd file (first user) | `htpasswd -bbc htpasswd.txt user pass` |
| Add user to HTPasswd | `htpasswd -b htpasswd.txt user pass` |
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
| Check permission | `oc adm policy can-I --as=<user> <verb> <resource> -n <ns>` |
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
| (default) | MD5 | Low (avoid in production) |
| `-s` | SHA | Medium |
| `-B` | bcrypt | High (recommended) |
| `-2b` | bcrypt (explicit) | High (recommended) |

---

## Review Questions

1. What is the difference between authentication and authorization in OpenShift?

2. Why must the HTPasswd secret be created in the `openshift-config` namespace?

3. What happens when a user authenticates through HTPasswd for the first time?

4. How do you grant the `edit` role to a user named `developer1` in the `myapp-project` namespace?

5. What is the difference between a Role and a ClusterRole?

6. How do you check if a user has permission to create pods in a namespace?

7. Why must a user re-authenticate after being added to a group?

8. What command removes the `cluster-admin` role from a user?

9. What is the difference between `oc adm policy add-role-to-user` and `oc adm policy add-cluster-role-to-user`?

10. How do you find out which users have permission to delete deployments in a namespace?

11. What does the `apiGroups: [""]` field mean in a Role rule?

12. Can a ClusterRoleBinding reference a namespace-scoped Role? If so, what is the effect?

13. What happens if you use the `-c` flag with `htpasswd` when adding a second user?

14. How do you add a user to an existing group?

15. What are the four RBAC resource types in OpenShift?

---

## Answers

1. **Authentication** verifies who a user is (identity verification through credentials). **Authorization** determines what an authenticated user is allowed to do (permission enforcement through RBAC roles and bindings).

2. The OAuth server in OpenShift is configured to look for HTPasswd secrets specifically in the `openshift-config` namespace. This is a fixed location defined by the OpenShift authentication operator. Placing the secret elsewhere means the OAuth server cannot find the credentials.

3. OpenShift automatically creates a `User` object and an `Identity` object linking the user to the HTPasswd provider. If any Group objects contain the username in their member list, the user is added to those groups. The user then receives permissions from any RoleBindings or ClusterRoleBindings that reference the user or their groups.

4. `oc adm policy add-role-to-user edit developer1 -n myapp-project`

5. A **Role** is namespace-scoped and defines permissions within a single namespace. A **ClusterRole** is cluster-scoped and can define permissions across the entire cluster, including cluster-level resources like nodes and namespaces. ClusterRoles can be bound at either the namespace level (via RoleBinding) or cluster level (via ClusterRoleBinding).

6. `oc adm policy can-I --as=<username> create pods -n <namespace>`. This returns `yes` or `no` without making an actual API call.

7. Group membership is evaluated at authentication time by the OAuth server. The list of effective groups is attached to the user's session token. Until the user re-authenticates and receives a new token, the old group list remains in effect.

8. `oc adm policy remove-cluster-role-from-user cluster-admin <username>`

9. `add-role-to-user` grants a namespace-scoped Role and creates a RoleBinding in the specified namespace. `add-cluster-role-to-user` grants a ClusterRole and creates a ClusterRoleBinding with cluster-wide effect. The former is limited to one namespace; the latter applies across the entire cluster.

10. `oc adm policy who-can delete deployments -n <namespace>`. This lists all users, groups, and service accounts that have the specified permission.

11. The empty string `""` refers to the **core API group**, which includes fundamental resources like pods, services, configmaps, secrets, and namespaces. No prefix is needed for these resources.

12. Yes. When a ClusterRoleBinding references a namespace-scoped Role, the Role's permissions are applied across all namespaces. This effectively elevates the namespace-scoped permissions to cluster-wide scope, though the Role itself still only defines namespace-level resource permissions.

13. The `-c` flag creates (overwrites) the file. All previously existing users are deleted, leaving only the newly added user. This is a data-destructive operation.

14. `oc adm groups add-users <group-name> <username>`

15. **Role** (namespace-scoped permissions), **ClusterRole** (cluster-scoped permissions), **RoleBinding** (binds roles to subjects in a namespace), and **ClusterRoleBinding** (binds roles to subjects cluster-wide).
