# Valkey Operator

## Main Components

### Valkey Operator

**Role:** Main orchestrator

**Functionalities:**
- watch for new CRs
- act on new CRs:
  - creates new clusters
  - upgrades clusters
  - scales clusters (plus resharding)
- communicates with **Sidecar**
- maintains certificates for Valkey TLS and for commmunication with **Sidecar** (mTLS)
  - internal 
  - *(optinal)* external, using Cert manager
- *(optional)* injects Valkey exporter for Prometheus

### Valkey Sidecar

**Role:** Valkey node coordinator

**Functionalities:**
- listen for commands from **Operator**
- reports statuses back to **Operator**
- responsible with pod readiness and liveness

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
  clusterType: cluster   # Options: sentinel, cluster
  masterCount: 3
  replicasPerMaster: 1

  tls:
    enabled: true

  podAntiAffinity:
    enabled: true

  tolerations:
  - key: "node-role.kubernetes.io/infra"
    operator: "Exists"
    effect: "NoSchedule"

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

  exporter:
    enabled: true

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
    metricsEndpoint: http://valkey-exporter:9121/metrics

  conditions:
  - type: Ready
    status: True
    lastTransitionTime: "2025-09-02T17:00:00Z"
  - type: TLSConfigured
    status: True
  - type: ACLSynced
    status: True
  - type: 
```

### Cluster Backup

TODO

### Cluster Restore

TODO

