
# Main Components

## 1. Valkey Operator

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

## 2. Valkey Sidecar

**Role:** Valkey node coordinator

**Functionalities:**
- listen for commands from **operator**
- reports statuses back to **operator**
- promote/demote Valkey cluster nodes
- trap SIGTERM and acts on pod deletion
- responsible with pod readiness and liveness
- heartbeat back to the **operator**

## 3. Valkey Operator Web UI

**Role:** UI for Operator
