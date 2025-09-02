# Valkey Operator - Requirements

<small>*draft*</small>

## Main Components

### Valkey Operator

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
  - *(optinal)* external, using Cert manager
- *(optional)* injects Valkey exporter for Prometheus
- can be cluster level or namespaced (different RBACs)

### Valkey Sidecar

**Role:** Valkey node coordinator

**Functionalities:**
- listen for commands from **operator**
- reports statuses back to **operator**
- promote/demote Valkey cluster nodes
- trap SIGTERM and acts on pod deletion
- responsible with pod readiness and liveness
- heartbeat back to the **operator**

### Valkey Operator Web UI

**Role:** UI for Operator

## Custom Resource Definitions

### Cluster Definition

```yaml
apiVersion: valkey.io/v1alpha1
kind: ValkeyCluster
metadata:
  name: valkey-poc
spec:
  valkeyImage:
    name: valkey/valkey-bundle
    tag: 8.1
    imagePullSecrets: []

  clusterType: cluster   # Options: sentinel, cluster
  masterCount: 3
  replicasPerMaster: 1

  tls:
    enabled: true

  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - valkey-poc
      topologyKey: "kubernetes.io/hostname"
  
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
  
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
  
  tolerations:
  - key: "node-role.kubernetes.io/infra"
    operator: "Exists"
    effect: "NoSchedule"

  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "500m"
  
  labels:
    myLabel: myValue

  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000

  acl:
    enabled: true
    users:
    - name: "valkey-user"
      password: "user-secret-pass"

  clusterPassword: "cluster-secret-pass"

  sidecarImage:
    name: ghcr.io/valkeyoperatorsidecar
    tag: v1.0
    imagePullSecrets: []
  
  exporter:
    enabled: true
    image:
      name: name
      tag: tag
      imagePullSecrets: []
  
    
```

### Valkey Operator Defaults

```yaml
apiVersion: valkey.io/v1alpha1
kind: ValkeyOperatioDefaults
metadata:
  name: valkey-operator-defaults
spec:
  # all from ValkeyCluster
  # generated at install, in operator namespace, from built in values
  $ can also exist in namespaces
  # precedence will be (from least to last):
  # - built in values
  # - the one in operator ns, with name valkey-operator-defaults
  # - the one in ns where ValkeyCluster CR is created
```

### Cluster Status

```yaml
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
  phase: Running           # Pending, Running, Failed, Updating, Backup, Restore
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

### Cluster Backup

TBD

### Cluster Restore

TBD

## Features (unorganized)

**Finalizers** - on pod delete, on CR delete (cleanup Valkey resources)  
**Status in CRs** - to reflect sidecar executions/Valkey cluster states  
**CRDs versioning** - as much as possible, do not change anything, add (with defaults)

**Observability** - metrics (Prometheus), logging (structured)

### mTLS

**Operator** generates a root CA on install, saved in ValkeyCA. For each Valkey cluster, it generates an intermediate CA, saved in ValkeyCertificateBundle. The mTLS and TLS certificates are mounted from secrets in containers as volumes. must mount CA in containers in ca-bundle. On cert change, **Sidecar** will reload and use it. **Operator** will accept a period of time (1m? 1h?) the older cert. After all sidecars are switched, will unload it from memory and *(TBD)* will clean ValkeyCertificateBinding of deprecated.

**AuthN/AuthZ:** use SPIFFE-style SANs to authorize requests (e.g. only sidecars with CN `valkey-poc-sidecar` can respond).

Protobuf contracts with versioning.

**Valkey /data** - accesible also from sidecar
