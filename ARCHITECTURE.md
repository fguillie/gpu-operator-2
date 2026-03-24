# NVIDIA GPU Operator - Architecture

## Overview

The NVIDIA GPU Operator is a Kubernetes operator that automates lifecycle management of NVIDIA GPU software components (drivers, container toolkit, device plugins, monitoring) on Kubernetes clusters. It eliminates manual GPU node configuration by treating GPU nodes like CPU nodes through standard OS images and operator-driven provisioning.

**Version:** v26.3.0
**Go Version:** 1.26.1

---

## Architecture

Two main controllers operate in a state machine pattern:

| Controller | CRD | Purpose |
|---|---|---|
| `ClusterPolicyReconciler` | `ClusterPolicy` (v1, stable) | Cluster-wide GPU setup |
| `NVIDIADriverReconciler` | `NVIDIADriver` (v1alpha1, alpha) | Individual driver deployments |
| `UpgradeReconciler` | — | Rolling driver upgrades |

```
┌─────────────────────────────────────────────────────────┐
│         GPU Operator Control Plane                      │
├─────────────────────────────────────────────────────────┤
│ Manager (controller-runtime)                            │
├─────────────────────────────────────────────────────────┤
│  ┌──────────────────┐   ┌──────────────────┐           │
│  │ ClusterPolicy    │   │ NVIDIADriver     │           │
│  │ Reconciler       │   │ Reconciler       │           │
│  └────────┬─────────┘   └────────┬─────────┘           │
│           │      State Manager   │                      │
│           └──────────┬───────────┘                      │
├──────────────────────┼───────────────────────────────────┤
│         ResourceManager & StateManager                  │
├──────────────────────────────────────────────────────────┤
│  Components Managed:                                    │
│  • NVIDIA Drivers (proprietary/open/precompiled)        │
│  • Container Toolkit                                    │
│  • Device Plugin                                        │
│  • DCGM & DCGM Exporter                                │
│  • GPU Feature Discovery (NFD)                          │
│  • MIG Manager, VFIO Manager, vGPU Manager             │
│  • Sandbox Device Plugins (KubeVirt/Kata)              │
│  • Validators & Monitoring                              │
└──────────────────────────────────────────────────────────┘
```

---

## Directory Structure

| Directory | Purpose |
|---|---|
| `api/` | CRD type definitions (`ClusterPolicy`, `NVIDIADriver`) |
| `controllers/` | Reconcilers, state machine, resource management |
| `cmd/` | Binaries: `gpu-operator`, `nvidia-validator`, `gpuop-cfg`, `manage-crds` |
| `internal/` | Core logic: state, conditions, rendering, config |
| `config/` | Kustomize configs, RBAC, CRD YAML |
| `deployments/` | Helm chart with `values.yaml` (600+ lines) |
| `validator/` | Post-deploy GPU validation manifests |
| `tests/e2e/` | End-to-end test framework (Ginkgo/Gomega) |
| `bundle/` | OLM (OperatorHub) bundle metadata |
| `vendor/` | Vendored Go dependencies |

---

## Key Source Files

**Controllers:**
- `controllers/clusterpolicy_controller.go` — main ClusterPolicy reconciliation (1000+ lines)
- `controllers/nvidiadriver_controller.go` — NVIDIADriver reconciliation
- `controllers/upgrade_controller.go` — rolling driver upgrades
- `controllers/state_manager.go` — state machine orchestration
- `controllers/resource_manager.go` — Kubernetes resource lifecycle
- `controllers/object_controls.go` — per-resource-type control functions

**Internal Libraries:**
- `internal/state/state.go` — state interface and sync state constants
- `internal/state/manager.go` — state manager implementation
- `internal/state/driver.go` — driver-specific state logic
- `internal/conditions/conditions.go` — CRD condition management
- `internal/render/` — manifest template rendering
- `internal/config/` — configuration parsing and management

**Entry Points:**
- `cmd/gpu-operator/main.go` — main operator binary
- `cmd/nvidia-validator/main.go` — post-deploy GPU validation tool
- `cmd/gpuop-cfg/` — ClusterPolicy/CSV validation CLI

---

## Custom Resource Definitions

### ClusterPolicy (v1 — Stable)
`api/nvidia/v1/clusterpolicy_types.go`

Manages cluster-wide GPU operator configuration. Singleton per cluster.

```go
ClusterPolicySpec {
  Operator              OperatorSpec
  Driver                DriverSpec
  Toolkit               ToolkitSpec
  DevicePlugin          DevicePluginSpec
  DCGMExporter          DCGMExporterSpec
  DCGM                  DCGMSpec
  GPUFeatureDiscovery   GPUFeatureDiscoverySpec
  MIG                   MIGSpec
  MIGManager            MIGManagerSpec
  Validator             ValidatorSpec
  GPUDirectStorage      GPUDirectStorageSpec
  SandboxWorkloads      SandboxWorkloadsSpec
  // ...
}
```

### NVIDIADriver (v1alpha1 — Alpha)
`api/nvidia/v1alpha1/nvidiadriver_types.go`

Controls individual driver deployments.

```go
NVIDIADriverSpec {
  DriverType        string  // gpu | vgpu | vgpu-host-manager
  UsePrecompiled    bool
  KernelModuleType  string  // auto | open | proprietary
  Image             string
  Version           string
  Repository        string
  ImagePullPolicy   string
  NodeSelector      map[string]string
  NodeAffinity      corev1.NodeAffinity
  Tolerations       []corev1.Toleration
  GPUDirectRDMA     GPUDirectRDMASpec
  GPUDirectStorage  GPUDirectStorageSpec
  GDRCopy           GDRCopySpec
  // ...
}
```

---

## Managed Components (Deployment Order)

1. **NVIDIA Driver** — proprietary, open, or precompiled kernel modules
2. **Container Toolkit** — enables GPU access in containers
3. **Device Plugin** — exposes GPUs as Kubernetes resources
4. **DCGM** — Data Center GPU Manager
5. **DCGM Exporter** — Prometheus metrics for GPU telemetry
6. **GPU Feature Discovery (NFD)** — labels nodes with GPU capabilities
7. **MIG Manager** — Multi-Instance GPU partitioning
8. **vGPU Manager** — virtual GPU support
9. **Sandbox Device Plugins** — KubeVirt/Kata container support
10. **Validator** — post-deploy verification

---

## Configuration

### Helm Chart (`deployments/gpu-operator/`)
- `Chart.yaml` — chart metadata
- `values.yaml` — default configuration (600+ lines)
- `templates/` — Go templates for Kubernetes manifests
- `crds/` — CRD YAML files
- `charts/node-feature-discovery/` — NFD subchart

### Kustomize (`config/`)
- `crd/` — CRD generation config
- `manager/` — operator deployment
- `rbac/` — Role-based access control
- `default/` — standard deployment overlays
- `samples/` — example ClusterPolicy resources

---

## Key Dependencies

| Package | Version | Purpose |
|---|---|---|
| `sigs.k8s.io/controller-runtime` | v0.23.3 | Reconciliation framework |
| `k8s.io/client-go` | v0.35.3 | Kubernetes client |
| `k8s.io/api` | v0.35.3 | Kubernetes API types |
| `NVIDIA/k8s-operator-libs` | — | Upgrade coordination |
| `NVIDIA/go-nvlib` | — | NVIDIA library wrapper |
| `NVIDIA/nvidia-container-toolkit` | — | Container runtime integration |
| `openshift/api` | — | OpenShift support |
| `prometheus-operator/prometheus-operator` | — | Monitoring integration |
| `onsi/ginkgo/v2` | — | E2E BDD test framework |
| `onsi/gomega` | — | Test assertions |
| `go.uber.org/zap` | — | Structured logging |

---

## Build System

```bash
make build          # Build all binaries
make build-image    # Build Docker image
make manifests      # Regenerate CRDs and RBAC
make generate       # Regenerate deepcopy methods
make unit-test      # Run unit tests with coverage
make coverage       # Generate coverage report
make lint           # Run golangci-lint
make bundle         # Generate OLM bundle
make fmt            # Format code
make goimports      # Organize imports
```

---

## Tests

**Unit Tests:** `*_test.go` files alongside source, run via `make unit-test`

Key files:
- `controllers/clusterpolicy_controller_test.go`
- `controllers/nvidiadriver_controller_test.go`
- `controllers/state_manager_test.go`
- `controllers/object_controls_test.go`
- `api/nvidia/v1alpha1/nvidiadriver_types_test.go`

**E2E Tests:** `tests/e2e/` using Ginkgo/Gomega against real Kubernetes clusters
- `framework/` — test infrastructure
- `suites/` — test cases
- `helpers/` — operator, pod, node, daemonset helpers
