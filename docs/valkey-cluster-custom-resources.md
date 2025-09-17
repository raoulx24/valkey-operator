# Custom Resource Definitions

## Cluster Definition

There will be several main Valkey Deployment Falvours., ValkeyStandalone, , ValkeyCluster

| CR                 | Description                            |
|--------------------|----------------------------------------|
| ValkeyStandalone   | Single Valkey instance                 |
| ValkeyCluster      | Classic Valkey Cluster                 |
| ValkeySentinel     | Sentinel Valkey Cluster                |
| ValkeyMultiCluster | Multi Datacenter/Region Valkey Cluster |

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
  
  # mutable after creation
  masterCount: 3
  replicasPerMaster: 1
  
  # if nothing is provided: no tls.
  # mtls between operator and side car is always on
  tls: valkey-poc-tls

  # if nothing is provided: no acl, no pass, no nothing
  acl: valkey-poc-acl

  operatorMtls:
    clientCertValidity: 90d        # 1m, 1h, 1d, 1mo, 1y
    rotateBeforePercent: 66        # between 66 and 80
  
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
  
  # imutable after creation
  pvc:
    storageSize: 30Gi
    storageClassName: my-storage-class
  
  backup:
    database:
      enabled: true
      save: "60 100"
    aof:
      enabled: true
      appendfsync: everysec              # always, everysec, or no
      autoAofRewritePercentage: 100
      autoAofRewriteMinSize: 64mb
  
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
kind: ValkeyClusterTls
metadata:
  name: valkey-poc-tls
spec:
  # if provided, someone else is generationg the certs
  # they will also be responsible with rotation
  # it must have the three keys, tls.crt, tls.key, ca.crt
  externalCerts: valkey-poc-tls-external
  
  # ignored if externalCerts is not nil
  internalCerts:
    clientCertValidity: 90d        # 1m, 1h, 1d, 1mo, 1y
    rotateBeforePercent: 66        # between 66 and 80
  
  tlsSecretName: valkey-poc-tls
  
  disableNonTlsPort: true
  tlsPort: 6379
  tlsAuthClients: true
  tlsCluster: true
  tlsReplication: true
  tlsCiphers:
  tlsProtocols: TLSv1.2 TLSv1.3
  tlsDhParamsFile:
    valueFrom:
      configMapKeyRef:
        name: my-dh-params-config
        key: myDhParams
  
  # should it be emphased that it is UTC (as it should be)?
  tlsRefreshRestartWindow: "0 4 * * 6,0" (every sat, sun at 4:00 AM)
```

```yaml
apiVersion: valkey.io/v1alpha1
kind: ValkeyClusterAcl
metadata:
  name: valkey-poc-acl
spec:
  # the generated secret that will be mounted in valkey nodes
  # imutable
  secretName: valkey-poc-acl-defs
  secretType: Secret                # Secret, SecretsStore
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
  acl-definitions.yaml: |
    # for password rotation, multiple passwords can be provided
    valkey-user:
      name: valkey-admin              # if empty, legacy mode used (with "default" user)
      # important: if there are several passwords, the first one will be picked by sidecar
      pass: ["myAdminPass"]
      # acl: "+@all ~*"               # harcoded, it will be minimal
    replication-user:
      name: primary-replication-user
      # important: if there are several passwords, the first one will be picked in valkey.conf
      # this means that if password rotation is required, the first one is the new one, and the rest are old
      pass: ["myClusterReplicationPass"]
      # acl: "+@replication"          # harcoded, they will be minimal
    exporter-user:
      name: exporter-user
      # important: if there are several passwords, the first one will be picked in valkey.conf
      # this means that if password rotation is required, the first one is the new one, and the rest are old
      pass: ["myExporterPass"]
      # acl: "+@replication"          # harcoded, they will be minimal
    users:
    - name: "valkey-user"
      pass: ["otherPass"]
      acl: "+GET +SET ~app:*"
```

## Valkey Operator Defaults

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

## Cluster Status

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

## Certificates

```yaml
# this is created upon valkey operator install
apiVersion: valkey.io/v1alpha1
kind: ValkeyCA
metadata:
  name: valkey-root-ca
spec:
  description: "Root CA for Valkey (TLS)/mTLS"
  secretType: Secret                       # Secret, SecretsStore
  caCertSecretRef: valkey-root-ca-cert     # Secret containing the root CA certificate and key
  validForDays: 3650                       # validity period for issued certs - only if operator is the issuer
```

```yaml
# this is created upon valkey cluster creation (note ownerReferences)
# it is the intermediate CA for mTLS and (possible (TLS)
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
    secretName: valkey-poc-mtls-secret
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

## Cluster Backup

TBD

## Cluster Restore

TBD
