# Vars

## Operator SDK Capabilities
The Operator SDK (Go-based) scaffolds a controller that can:
- Watch and reconcile custom and native Kubernetes resources.
- Create, update, and delete resources declaratively.
- React to events (add/update/delete) and status changes.
- Manage finalizers to delay deletion.
- Patch resources dynamically.
- Integrate webhooks to validate or mutate resources.
- Handle leader election, metrics, and health checks.

## Core Principles for Operator SDK Development
1. Declarative First
- Treat the Custom Resource (CR) as the source of truth.
- Reconciliation logic should always aim to bring the system to the desired state described in the CR.
- Avoid imperative hacks - let Kubernetes do the heavy lifting.
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

## Architecture - Valkey Operator

### Core Components

| Component    | Role |
|--------------|------|
| Operator     | Control plane for orchestration, resharding, failover, upgrades      |
| Sidecar      | Agent co-located with each Valkey node, handles lifecycle and health |
| SPIRE Agent  | Injects SPIFFE identity into each workload                           |
| SPIRE Server | Issues and rotates mTLS certs with SPIFFE IDs                        |

### Identity & Security
- **mTLS with SPIFFE:** all gRPC communication uses mutual TLS with SPIFFE IDs
- **Operator validates SPIFFE SANs:** ensures sidecar identity matches expected namespace/tenant
- **Namespace-scoped RBAC:** **operator** runs in single/multi-namespace or cluster-wide mode, with RBAC patterns enforced at startup
- **Trust Domain Reinforcement:** SPIFFE trust domains mirror namespace boundaries for extra isolation
- **Cerificates rotation:** the certificates are issued with a lifespan of 2-6h (TBD decide duration). They will be rotated when 66% of valability passed

### Communication Model

| Flow                  | Direction            | Notes |
|-----------------------|----------------------|-------|
| Heartbeat             | Sidecar -> Operator  | Unary RPC every minute with jittered backoff |
| Orchestration Trigger | Operator -> Sidecar  | Operator replies with "stay connected" flag  |
| Orchestration Session | Bidirectional stream | Temporary stream for multi-step workflows (eg resharding) |

Rules
- **Protocol:** gRPC
- **Hybrid gRPC model:** stateless polling + sticky orchestration sessions
- **Backoff strategy:** logarithmic with jitter to prevent thundering herd
- **Stream lifecycle:** short-lived, closed after orchestration completes
- **no mid stream TLS change:** there is no need for it. If mTLS certs are rotated during orchestration, the change will be postponed

### Orchestration State
- **CRD Status Field:** tracks orchestration progress, step versioning, and sidecar responses
- **Durable across restarts:** **operator** can resume orchestration mid-flow
- **Partial failure handling:** case-dependent logic with retry, rollback, or quorum enforcement
- **Queued operations:** if another orchestration is required during an onging one or if multiple orchestrations are required, they will be queued and executed sequentially

### Observability
- **Traces:** per-orchestration flow, correlated across operator and sidecars
- **Metrics:** heartbeat latency, orchestration duration, retry counts
- **Events:** _(optional)_ human-readable audit trail, not used for logic

### Unbound Mode
- **Sidecars operate independently:** no operator communication
- **Best-effort maintenance:** health checks, restarts, local failover
- **User responsibility:** no orchestration, no centralized control

## Valkey ACL

### Some considerations related to ACL in Valkey

ACL is the most crucial setting in good functionning of a cluster, maybe same level as networking and maybe above TLS.

The most critical moment is when the pod is starting. If aclfile `/etc/valkey-acl/users.acl` is provided in valkey.conf and the file is corrupt, the startup of the cluster is in a bad shape.
1. primaries will start, but no ACL. This means no user, no pass, no nothing. They are bare open
2. replicas will start the cycle of restarts, as they are not able to connect to primaries (if primaries still have ACL)

### ACL 'way-of-life'

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
# for replication
ACL SETUSER replicator on >replica-password +@replication
```
3. Persist ACLs Securely
Valkey stores ACLs in memory. To persist them, use the aclfile directive in `valkey.conf`:
```ini
aclfile /etc/valkey-acl/users.acl
# important: we also need the info for replication
masteruser replicator
masterauth replica-password
```
Export your ACL rules
```resp
# ensures your ACLs survive restarts and are version-controlled if needed.
ACL SAVE
```

Regarding password rotation, Valkey allows multiple passwords per user, so both will be valid.
```resp
ACL SETUSER replicator on >old-password >new-password +replication
```

## ACL File Format

Valkey ACL files are plain text files containing one or more ACL SETUSER commands. Each line defines a user and their permissions using a domain-specific language (DSL).

```resp
ACL SETUSER <username> [rules...]
```
Each rule modifies the user's state, authentication, or access rights.

### Parameter Types

| Param Type | Description                                                            |
|------------|------------------------------------------------------------------------|
| username   | unique identifier for the user                                         |
| flags      | keywords that toggle user state (e.g., `on`, `off`, `nopass`)          |
| passwords  | plaintext (`>password`) or SHA256 hash (`#hash`)                       |
| commands   | command categories (`+@read`, `+@write`) or specific commands (`+GET`) |
| keys       | Key patterns using glob-style matching (`~reports:*`)                  |
| channels   | Pub/Sub channel patterns (`&news:*`)                                   |
| selectors  | Advanced permission blocks (optional, for scoped access control)       |

### Rule Constraints

1. User Flags  
**Format:**  
`on` / `off`: Enables or disables the user  
`nopass`: Allows login without a password  
`sanitize-payload`: Hides sensitive command payloads in logs  
    > Must include either on or off. nopass and sanitize-payload are optional

2. Passwords  
**Format:** `>plaintext` or `#SHA256hash`  
    > Multiple passwords allowed. Must be unique per user  
    > Plaintext must be prefixed with `>`; hash must be valid SHA256 and prefixed with `#`
    > Only printable chars are accepted, except _space_. Regex: `^[\x21-\x7E]{16,128}$`

3. Command Permissions  
**Categories:** `+@read`, `+@write`, `+@admin`  
**Specific commands:** `+GET`, `-FLUSHALL`  
    > Negation: Use - to revoke access  
    > Must start with `+` or `-`. Categories must be valid ACL groups. Commands must be known Valkey commands

4. Key Patterns  
**Format:** `~pattern`  
**example** `reports:my[cp]ut?er*` will match _mycutters_, _mtputiereles_  
    > Supports glob-style matching (`*`, `?`, `[]`). example `reports:my[cp]ut?er* will match  
    > Must start with ~. Pattern must be syntactically valid

5. Channel Patterns  
**Format:** `&pattern``  
    > Same matching rules as keys  
    > Must start with `&`. Pattern must be valid

6. Selectors (Advanced)  
**Format:** `selectorname (+command ~keypattern &channelpattern)`
    > `selectorname` is optional. the rest are as above  
    > They are grouped inside `(` and `)`

More on [Valkey ACL page](https://valkey.io/topics/acl/).

## TLS

| TLS Setting        | Purpose                      | Example                      |
|--------------------|------------------------------|------------------------------|
| port 0             | Disables non-TLS port        | port 0                       |
| tls-port           | Enables TLS port             | 6379                         |
| tls-cert-file      | Server certificate           | /etc/valkey/tls/tls.crt      |
| tls-key-file       | Server private key           | /etc/valkey/tls/tls.key      |
| tls-ca-cert-file   | CA bundle for trust          | /etc/valkey/tls/ca.pem       |
| tls-auth-clients   | Require client certs (mTLS)  | yes                          |
| tls-cluster        | Encrypt cluster bus traffic  | yes                          |
| tls-replication    | Encrypt replication traffic  | yes                          |
| tls-ciphers        | Specify cipher suites        | TLS13-AES-256-GCM-SHA384:... |
| tls-protocols      | Restrict TLS versions        | TLSv1.2 TLSv1.3              |
| tls-dh-params-file | Custom DH params (if needed) | /etc/valkey/tls/dhparams.pem |

Considerations:
- always use full certificate chains in tls-ca-cert-file
- bundle old + new CA certs during rotation to avoid trust gaps
- Valkey does not reload certs dynamically - pods must be restarted
- sse read-only volume mounts and non-root users for security
- validate certs with sidecar before Valkey restarts.

Sample config
```ini
tls-port 6379
port 0
tls-cert-file /etc/valkey-cluster/tls/server.crt
tls-key-file /etc/valkey-cluster/tls/server.key
tls-ca-cert-file /etc/valkey-cluster/tls/ca_bundle.pem
tls-auth-clients yes
tls-cluster yes
tls-replication yes
```

The `valkey-cli` must be run with full trust
```sh
valkey-cli --tls --cert /etc/client/client.crt --key /etc/client/client.key --cacert /etc/client/ca_bundle.pem
```

## Ways of blocking/delaying pod delete

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

## How a pod can be deleted

### Voluntary Deletion Scenarios (Intentional)

| Trigger                           | Who Initiates         | Can Be Blocked? |
|-----------------------------------|-----------------------|-----------------|
| `kubectl delete pod`              | User                  | Fin, PDB, VH    |
| Rolling update (ss)               | Controller            | PDB, Upd, VH    |
| `kubectl rollout restart`         | User                  | PDB, Upd, VH    |
| `kubectl drain`                   | User                  | PDB, VH         |
| Cluster autoscaler                | Kubernetes controller | PDB, VH         |
| Eviction API (policy/v1/Eviction) | Kubelet or controller | PDB, VH         |
| Preemption (PriorityClass)        | Scheduler             | No              |

### Involuntary Deletion Scenarios (Automatic or Forced)
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

## CSI Secrets

Secrets can be managed in a cluster also from an external provider, like a Vault. In order to use them, the following steps are to be followed (example for Azure KeyVault):

1. Install the drivers

```sh
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system --set enableSecretRotation=true
```

2. Create the CR. It will "expose" the objects from the external provider

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kv-example
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    keyvaultName: "my-keyvault-name"
    cloudName: "AzurePublicCloud"
    objects: |
      array:
      - objectName: my-secret
        objectType: secret
      - objectName: my-cert
        objectType: cert
    tenantId: "<your-tenant-id>"
```

3. Mount the secrets in a pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-secrets
spec:
  containers:
  - name: busybox
    image: busybox
    command: [ "sleep", "3600" ]
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets-store"
      readOnly: true
  volumes:
  - name: secrets-store-inline
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azure-kv-example"
```

4. (optional) Mount them in a `Secret`

```yaml
spec:
  ...
  secretObjects:
  - secretName: synced-k8s-secret
    type: Opaque
    data:
    - objectName: my-secret
      key: password
```

## `csi secrets` vs `secret` Considerations

A `secret` can be consumed and loaded by the **operator**, while `csi secrtes` can only be mounted in a pod. Because of this, and because it is desired to have a common approach for both worlds, the **operator** will not manage external provided secrets (by both means).

The only secrets creted by it are intermediate and client/server certificates used internally by mTLS. Main two reasons:
-  it is extremely complicated for an external party to generate all needed certificates, with proper SPIFFE-style SANs for authorization
- it is imperative that the communication between **operator** and **sidecars** is not hampered by some misconfigured certificates

The only exception is when the cluster is created directly unbound, when there is no mTLS, so no need for related certificates.

## oliver006/redis_exporter

sample-pwd-file.json
```json
{
  "redis://localhost:16379": "",
  "redis://exporter@localhost:16390": "exporter-password",
  "redis://localhost:16380": "redis-password",
  "rediss://localhost:6379": "tls-enabled-redis-password"
}
```

Hot reload of pwd file
```sh
curl -X POST http://localhost:9121/-/reload
```

ACL for cluster
```resp
ACL SETUSER <<<USERNAME>>> -@all +@connection +memory -readonly +strlen +config|get +xinfo +pfcount -quit +zcard +type +xlen -readwrite -command +client -wait +scard +llen +hlen +get +eval +slowlog +cluster|info +cluster|slots +cluster|nodes -hello -echo +info +latency +scan -reset -auth -asking ><<<PASSWORD>>>
```

