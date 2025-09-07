# Valkey Operator - Requirements

*draft*

## Main Components

### 1. Valkey Operator

**Role:** Main orchestrator

**Functionalities:**
- watch and act on new CRs:
  - creates new clusters
  - upgrades clusters
  - scales clusters (plus resharding)
- detects Valkey cluster drifts (statefulsets, config maps, secrets etc)
- communicates with **sidecar**
- maintains certificates for Valkey TLS and for commmunication with **sidecar** (mTLS)
  - internal 
  - *(optional)* external, using Cert manager, for TLS (and future mTLS in Valkey)
- *(optional)* injects Valkey exporter for Prometheus
- can be cluster level or namespaced (different RBACs)

### 2. Valkey Sidecar

**Role:** Valkey node coordinator

**Functionalities:**
- listen for commands from **operator**
- reports statuses back to **operator**
- promote/demote Valkey cluster nodes
- trap SIGTERM and acts on pod deletion
- responsible with pod readiness and liveness
- heartbeat back to the **operator**

### 3. Valkey Operator Web UI

**Role:** UI for Operator

## Custom Resource Definitions

### Cluster Definition

```yaml
apiVersion: valkey.io/v1alpha1
kind: ValkeyCluster
metadata:
  name: valkey-poc
spec:
  # mutable after creation
  valkeyImage:
    name: valkey/valkey-bundle
    tag: 8.1
    imagePullSecrets: []
  
  # imutable after creation
  clusterType: cluster   # Options: sentinel, cluster
  # mutable after creation
  masterCount: 3
  replicasPerMaster: 1
  
  # mutable after creation
  tls:
    enabled: true
  
  # mutable after creation
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - valkey-poc
      topologyKey: "kubernetes.io/hostname"
  
  # mutable after creation
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - valkey-poc
        topologyKey: "kubernetes.io/hostname"
  
  # mutable after creation
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
  
  # mutable after creation
  tolerations:
  - key: "node-role.kubernetes.io/infra"
    operator: "Exists"
    effect: "NoSchedule"

  # mutable after creation
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "500m"
  
  # mutable after creation
  labels:
    myLabel: myValue
  
  # (i)mutable after creation
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000

  # if nothing is provided, no acl, no pass, no nothing
  acl: valkey-poc-acl

  # mutable after creation
  sidecarImage:
    name: ghcr.io/valkeyoperatorsidecar
    tag: v1.0
    imagePullSecrets: []
  
  # mutable after creation
  exporter:
    enabled: true
    image:
      name: name
      tag: tag
      imagePullSecrets: []
```

```yaml
apiVersion: valkey.io/v1alpha1
kind: ValkeyClusterAcl
metadata:
  name: valkey-poc-acl
spec:
  # the generated secret that will be mounted in valkey nodes
  secretName: valkey-poc-acl
  admin:
    name: valkey-admin              # if empty, legacy mode used (with "default" user)
    # password will be from secretName, with key admin.name. if empty, name is "default"
  # imutable after creation (?)
  clusterPassword: "cluster-secret-pass"
    valueFrom:
      secretKeyRef:
        key: cluster-password
  users:
  - name: "valkey-user"
    acl: "+GET +SET ~app:*"         # admin is +@all ~*
    # password will be from secretName, with key user[i].name
  secrets:
    secretName: valkey-poc-acl
status:
  phase: Applied                    # MissingSecret, IncompleteSecret, Applying, Applied, Error
  nodes:
  - name: node1
    status: Ok
  - name: node2
    status: Error
    message: "ACL LOAD failed: invalid syntax"
```

```yaml
# corresponding secret for acl
apiVersion: v1
kind: Secret
metadata:
  name: valkey-poc-acl-defs
type: Opaque
stringData:
  valkey-admin-acl: "+@all ~*"
  valkey-admin-pass: "myPass"
  valkey-user-acl: "+GET +SET ~app:*"
  valkey-user-pass: "otherPass"
```

### Valkey Operator Defaults

```yaml
apiVersion: valkey.io/v1alpha1
kind: ValkeyClusterDefaults
metadata:
  name: valkey-operator-defaults
spec:
  # all from ValkeyCluster
  # generated at install, in operator namespace, from built in values
  # can also exist in namespaces
  # precedence will be (from least to last):
  # - built in values
  # - the one in operator ns, with name valkey-operator-defaults
  # - the one in ns where ValkeyCluster CR is created
```

### Cluster Status

```yaml
# there are a lot of things to be clarified here
# what to have, details etc
# maybe (more than 50%) will be merged with ValkeyCluster
# if merge, watch out for size, as etcd might reject it
apiVersion: valkey.io/v1alpha1
kind: ValkeyClusterStatus
metadata:
  name: valkey-poc-status
  ownerReferences:
  - apiVersion: valkey.io/v1alpha1
    kind: ValkeyCluster
    name: my-app
    uid: a1b2c3d4-5678-90ef-ghij-klmnopqrstuv
    controller: true
    blockOwnerDeletion: true

spec:
  clusterRef: valkey-poc   # Reference to the ValkeyCluster resource

status:
  phase: Running           # Pending, Running, Failed, Updating, Backup, Restore, Unbound
  clusterType: cluster
  masterCount: 3
  replicasPerMaster: 1

  statefulSet:
    name: valkey-poc

  tls:
    enabled: true
    certificateStatus: Valid

  acl:
    enabled: true
    synced: true
    users:
    - name: valkey-user
      status: Active

  exporter:
    enabled: true
    metricsEndpoint: http://Valkey-exporter:9121/metrics

  conditions:
  - type: Ready
    status: True
    lastTransitionTime: "2025-09-02T17:00:00Z"
  - type: TLSConfigured
    status: True
  - type: ACLSynced
    status: True
```

### Certificates

```yaml
# this is created upon valkey operator install
apiVersion: valkey.io/v1alpha1
kind: ValkeyCA
metadata:
  name: valkey-root-ca
spec:
  description: "Root CA for Valkey TLS/mTLS"
  caCertSecretRef: valkey-root-ca-cert     # Secret containing the root CA certificate
  caKeySecretRef: valkey-root-ca-key       # Secret containing the root CA private key
  validForDays: 3650                       # Optional: validity period for issued certs
```

```yaml
# this is created upon valkey cluster creation (note ownerReferences)
# it is the intermediate CA for mTLS and TLS
apiVersion: valkey.io/v1alpha1
kind: ValkeyCertificateBundle
metadata:
  name: valkey-poc-cert-bundle
  ownerReferences:
  - apiVersion: valkey.io/v1alpha1
    kind: ValkeyCluster
    name: valkey-poc
    uid: a1b2c3d4-5678-90ef-ghij-klmnopqrstuv
    controller: true
    blockOwnerDeletion: true
spec:
  caRef: valkey-root-ca                    # Reference to ValkeyCA
  purpose: mtls                            # Options: mtls, tls
  certSecretRef: valkey-cert-2025-09       # Secret containing the signed certificate
  keySecretRef: valkey-key-2025-09         # Secret containing the private key
  issuedAt: "2025-09-01T00:00:00Z"
  expiresAt: "2026-09-01T00:00:00Z"
  rotationPolicy:
    autoRotate: true
    renewBeforeDays: 30
```

```yaml
# certificates used for mTLS and TLS. current ones
apiVersion: valkey.io/v1alpha1
kind: ValkeyCertificateBinding
metadata:
  name: valkey-poc-cert-binding
  ownerReferences:
  - apiVersion: valkey.io/v1alpha1
    kind: ValkeyCertificateBundle
    name: valkey-poc-cert-bundle
    uid: a1b2c3d4-5678-90ef-ghij-klmnopqrstuv
    controller: true
    blockOwnerDeletion: true
spec:
  clusterRef: valkey-poc

  active:
  - purpose: mtls
    bundleRef: valkey-poc-cert-bundle
    secretName: valkey-mtls-secret
    thumbprint: "a1b2c3d4e5f67890abcd1234567890efabcd1234"
    issuedAt: "2025-09-01T00:00:00Z"
    expiresAt: "2026-09-01T00:00:00Z"

  - purpose: tls
    bundleRef: valkey-poc-cert-bundle-tls
    secretName: valkey-poc-tls-secret
    thumbprint: "beadfacefeed1234567890abcdef1234567890ab"
    issuedAt: "2025-08-15T00:00:00Z"
    expiresAt: "2026-08-15T00:00:00Z"
    active: true
  
  # maybe we won't need this
  deprecated:
  - purpose: mtls
    thumbprint: "deadbeefcafebabe1234567890abcdef12345678"
    issuedAt: "2024-09-01T00:00:00Z"
    expiresAt: "2025-09-01T00:00:00Z"
```

> **Important:** Valkey 9 comes with client mTLS. To be studied.

### Cluster Backup

TBD

### Cluster Restore

TBD

## Features (unorganized)

### CR Changes
TBD

### **Operator** recovery and resources reconciliation from restart/scale 0
TBD

### Resources identification

Mark all created kubernetes resources (statefulsets, configmaps, secrets, pvcs, svcs, pdbs etc) with an annotation, to clearly link them to the `ValkeyCluster`. Like
```yaml
metadata:
  annotations:
    valkey.io/cr: valkey-poc    # name of the ValkeyCluster
    valkey.io/locked: "true"    # maybe...
  ownerReferences:
    # etc
```

All created CRs must have set correct `ownerReferences`.

### Unbound Valkey cluster from **operator**

If the user wishes to take control over Valkey cluster, it will be allowed to remove `valkey.io/cr: valkey-poc` from `statefulset`. After doing this, the operator will perform the following:
- clear all other annotations from resources
- will clear all statuses from `ValkeyClusterStatus` and will change to
```yaml
status:
  phase: Unbound
```
**TBD:** what to do and how, if user wants it back

### Validation Webhooks

In order to distinguish changes self vs others, **operator** generates a unique token per update. The webhook maintains a short-lived cache of valid tokens. Once used, the token is invalidated.

```yaml
metadata:
  annotations:
    valkey.io/update-token: "abc123"
```

All other changes are rejected. Exceptions are pods, for `delete`.

### Operator - Sidecar communication

**Operator** is mostly idle, the **sidecar’s** heartbeat is low-frequency and lightweight. The communication will be
- **Sidecar** initiated - Heartbits or major events: HTTP/1.1. Major events can be Valkey failed, someone deleted the pod
- **Operator** initiated - rest of communication: gRPC - only **orchestrator** open channels to needed **sidecars** when orchestration is triggered. Even use short-lived gRPC streams if needed.

### mTLS

**Operator** generates a root CA on install, saved in ValkeyCA. For each Valkey cluster, it generates an intermediate CA, saved in ValkeyCertificateBundle. The mTLS and TLS certificates are mounted from secrets in containers as volumes. must mount CA in containers in ca-bundle. On cert change, **Sidecar** will reload and use it. **Operator** will accept a period of time (1m? 1h?) the older cert. After all sidecars are switched, will unload it from memory and *(TBD)* will clean ValkeyCertificateBinding of deprecated.

**AuthN/AuthZ:** use SPIFFE-style SANs to authorize requests (e.g. only sidecars with CN `valkey-poc-sidecar` can respond).

Protobuf contracts with versioning.

### Pods deletion blocking

When a pod needs to be recreated, the following steps will be taken:
1. Patch PDB to minAvailable: N-1. This temporarily allows exactly one Pod to be disrupted
2. Delete or restart one Pod
3. Start a validation webhook and deny changes on rest of the pods (from same statefulset)
4. Wait for the Pod to become healthy
5. Patch PDB back to minAvailable: 100%
6. Stop validation webhook

Ways of blocking/delaying pod delete

**Finalizers** - on pod delete, on CR delete (cleanup Valkey resources). On pods, they must be injected (PATCH) after creation
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  finalizers:
  - valkey.io/finalizer
```

**Statefulset Update Strategy - on delete:** this will not update any pods if something changes. it will create a pod with new spec only on delete
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: valkey
spec:
  serviceName: "valkey"
  replicas: 6
  selector:
    matchLabels:
      app: valkey
  updateStrategy:
    type: OnDelete
  template:
    metadata:
      labels:
        app: valkey
    spec:
      containers:
      - name: nginx
        image: valkey
```

**Pod Lifecycle** - `preStop` is called and waited before SIGTERM is sent
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

**Pod Disruption Budget** - used with `minAvailable: 100%`. Only `--force --grace-period=0` will work
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: block-all-disruptions
spec:
  minAvailable: 100%
  selector:
    matchLabels:
      app: my-stateful-app
```

**Intercept deletion:** When a Pod is marked for deletion, Kubernetes sets `metadata.deletionTimestamp`.

### ACL creation and rotation

#### Creation
Store pregenerated ACL file as a Kubernetes Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: valkey-poc-acl
type: Opaque
stringData:
  users.acl: |
    user admin on >StrongPassword +@all ~*
```

Mount it as a read-only volume:
```yaml
# container spec
volumeMounts:
- name: acl-volume
  mountPath: /etc/valkey/acl
  readOnly: true
volumes:
- name: acl-volume
  secret:
    secretName: valkey-acl
# ...
# pod spec
volumes:
- name: users-acl-volume
  secret:
    secretName: valkey-poc-acl
    items:
    - key: users.acl
      path: users.acl
```
Reference it in `valkey.conf`:
```ini
aclfile /etc/valkey/acl/users.acl
```

#### Secret rotation
When a change in `ValkeyClusterAcl` or in secret is detected and reconciled, the following happens:
1. **Operator** creates the new secret with the rendered users.acl
2. **Operator** opens grpc with all sidecars
3. **Operator** confirms with all sidecars that ACL is ok (do we need to test k8s here?)
4. **Operator** asks for acl reapply
5. **Sidecar** checks if its password changed
6. **Sidecar** connects to Valkey and executes
```resp
ACL LOAD
```
7. **Sidecar** (if password changed, reconnects and) checks ACL
```resp
ACL LIST
```
8. **Sidecar** reports back status
9. **Operator** change status in `ValkeyClusterAcl`

### Deferred executions

There are cases when one object depends on another. Use Indexing for Reverse Lookup: with [controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime), index the CRs by the Secrets they reference:

```go
mgr.GetFieldIndexer().IndexField(&ValkeyClusterAcl{}, "spec.secrets.secretName", func(obj client.Object) []string {
    return []string{obj.(*ValkeyClusterAcl).Spec.Secrets.SecretName}
})
```
Then, when a Secret changes, list all CRs that reference it:

```go
var aclList ValkeyClusterAclList
r.List(ctx, &aclList, client.MatchingFields{"spec.secrets.secretName": secret.Name})
```

The reconcile loop should:
- check if the Secret exists and is complete
- if not, set `status.phase`: `WaitingForSecret` and requeue
- if yes, proceed with work and update `status.phase`: `Applied`

Practicly, we have the following rules
1. When a CR changes, reconcile it
2. When a Secret changes, reconcile all CRs that reference it
3. Use Status to Track Progress
```yaml
status:
  phase: WaitingForSecret
  lastChecked: 2025-09-07T19:17:00Z
  missingKeys:
    - valkey-user-pass
```

Actual Known Deps:
| Who | DependsOn | Why/What |
|-----|-----------|----------|
| ValkeyCluster | ValkeyCertificateBundle | Intermediate CA for mTLS (and TLS) |
| ValkeyCluster | ValkeyCertificateBinding | Certs for mTLS (and TLS) |
| ValkeyCluster | ValkeyClusterAcl | Certs for mTLS (and TLS) |
| ValkeyCertificateBundle | ValkeyCertificateCA | Operator CA |
| ValkeyCertificateBundle | Secret | the secret that holds the intermediate certs |
| ValkeyCertificateBinding | ValkeyCertificateBundle | the issuer for certs |
| ValkeyCertificateBinding | Secrets | the secrets that hold the certs for mTLS (and TLS) |
| ValkeyClusterAcl | Secret | the secret that holds the ACL definitions |

### Others

**Status in CRs** - to reflect sidecar executions/Valkey cluster states  
**CRDs versioning** - as much as possible, do not change anything, add (with defaults)  
**Observability** - metrics (Prometheus), logging (structured)  
**Valkey /data** - accesible also from sidecar  
**Valkey /tmp** - common between containers

## Vars

### Operator SDK Capabilities
The Operator SDK (Go-based) scaffolds a controller that can:
- Watch and reconcile custom and native Kubernetes resources.
- Create, update, and delete resources declaratively.
- React to events (add/update/delete) and status changes.
- Manage finalizers to delay deletion.
- Patch resources dynamically.
- Integrate webhooks to validate or mutate resources.
- Handle leader election, metrics, and health checks.

### Core Principles for Operator SDK Development
1. Declarative First
- Treat the Custom Resource (CR) as the source of truth.
- Reconciliation logic should always aim to bring the system to the desired state described in the CR.
- Avoid imperative hacks—let Kubernetes do the heavy lifting.
2. Idempotency
- Every reconciliation loop should be safe to repeat.
- No side effects from duplicate executions.
- This ensures stability, especially during retries or restarts.
3. Event-Driven, Not Time-Driven
- Watch relevant resources (CRs, Secrets, ConfigMaps, etc.)
- Avoid polling or time-based triggers unless absolutely necessary.
- Use indexing and dependency tracking to respond intelligently to changes.
4. Status Is a First-Class Citizen
- Use the status field to reflect runtime state, errors, and progress.
- This improves observability, debugging, and auditability.
- Think of status as your operator’s dashboard.
5. Graceful Error Handling
- Don’t crash or panic—report errors in status, log clearly, and retry safely.
- Use exponential backoff or rate limiting if needed.
6. Validation and Admission Control
- Validate CRs before acting on them.
- Use OpenAPI schemas, webhooks, or preflight checks to catch misconfigurations early.
7. Security by Design
- Minimize RBAC permissions—follow least privilege.
- Avoid mounting Secrets unless necessary.
- Sanitize inputs and avoid leaking sensitive data in logs or status.
8. Modular and Extensible
- Keep your reconciliation logic clean and composable.
- Use helper functions, services, or sub-controllers for complex workflows.
- Make it easy to extend without rewriting everything.
9. Resilience Across Restarts
- Avoid in-memory state that doesn’t survive Pod restarts.
- Use Kubernetes-native constructs (labels, annotations, status) to track progress.
10. GitOps Compatibility
- Design your CRs and workflows to be declarative and version-controlled.
- Avoid side effects that aren’t reflected in the CR spec.
- Make your operator predictable in CI/CD pipelines.
11. Observability & Metrics
- Emit Prometheus metrics for reconciliation success/failure, duration, and resource health.
- Use structured logging with correlation IDs or resource names.
- Integrate with tracing if your operator spans multiple systems.

### Valkey ACL
Valkey ACL is built to be backward-compatible. That means, by default, Valkey ships with a built-in user called `default`, which has full access to all commands and keys unless explicitly restricted. It is the infered user for legacy mode, where no user is provided.
```sh
# legacy mode, "default" is infered
AUTH myPass
# ACL mode
AUTH myUser myPass
```

Bootstrap strategy for ACL implementation:
1. Lock Down the Default User Immediately. In `valkey.conf`, do the following:
```ini
# Set a strong password for the default user
requirepass "myPass"

# Disable dangerous commands if needed (TBD)
rename-command FLUSHALL ""
rename-command CONFIG ""
```
2. Use the `default` user to create a named superuser
```sh
AUTH myPass
# create your named superuser:
ACL SETUSER valkey-admin on >myValkeyAdminPass +@all ~*
# disable or Restrict the Default User
ACL SETUSER default off
# or restrict it severely
ACL SETUSER default on >someOtherPassword -@all ~none
```
3. Persist ACLs Securely
Valkey stores ACLs in memory. To persist them, use the aclfile directive in `valkey.conf`:
```ini
aclfile /etc/valkey-acl/users.acl
```
Export your ACL rules
```sh
# ensures your ACLs survive restarts and are version-controlled if needed.
ACL SAVE
```

### How a pod can be deleted

#### Voluntary Deletion Scenarios (Intentional)

| Trigger                           | Who Initiates         | Can Be Blocked? |
|-----------------------------------|-----------------------|-----------------|
| `kubectl delete pod`              | User                  | Fin, PDB, VH    |
| Rolling update (ss)               | Controller            | PDB, Upd, VH    |
| `kubectl rollout restart`         | User                  | PDB, Upd, VH    |
| `kubectl drain`                   | User                  | PDB, VH         |
| Cluster autoscaler                | Kubernetes controller | PDB, VH         |
| Eviction API (policy/v1/Eviction) | Kubelet or controller | PDB, VH         |
| Preemption (PriorityClass)        | Scheduler             | No              |

#### Involuntary Deletion Scenarios (Automatic or Forced)
*These are triggered by system conditions or failures*

| Trigger                                 | Cause                       | Can Be Blocked? |
|-----------------------------------------|-----------------------------|-----------------|
| Node failure                            | Node becomes unreachable    | No              |
| Resource pressure (memory, disk)        | Kubelet eviction            | No              |
| Pod TTL (ttlSecondsAfterFinished)       | TTL controller              | Fin             |
| Pod owner deletion (e.g. Job)           | OwnerReference with cascade | Fin             |
| Force delete `--force --grace-period=0` | User override               | Fin, VH         |

Legend:

| Code | Name                                               |
|------|----------------------------------------------------|
| Fin  | Finalizers - technically it won't block, but delay |
| PDB  | Pod Disruption Budget                              |
| Upd  | Statefulset - Update Strategy - On Delete          |
| VH   | Api Validation Hook                                |