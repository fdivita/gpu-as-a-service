# GPU-as-a-Service: Multi-Tenant GPU Quota Management on OpenShift AI

## Use Case

Organizations running AI/ML workloads on OpenShift often face a shared GPU scarcity problem: multiple teams (e.g. inference serving and model training) compete for the same limited pool of GPUs, leading to resource starvation, unpredictable scheduling, and manual intervention.

This repository provides a **GPU-as-a-Service** solution that uses **Red Hat Build of Kueue** to enforce fair, priority-aware GPU quota management across teams. It implements:

- **Guaranteed GPU quotas per team** with elastic borrowing when capacity is idle, ensuring no GPU sits unused while workloads wait.
- **Priority-based preemption** so that high-priority inference workloads can reclaim GPUs from lower-priority training jobs when demand spikes.
- **Hardware Profiles** integrated with the OpenShift AI dashboard, giving data scientists a self-service experience — they select a profile and the platform handles queue routing, priority, and GPU scheduling transparently.
- **Grafana monitoring** with a purpose-built dashboard tracking workload lifecycle, admission latency, resource utilization, borrowing, and Prometheus scrape health.

The entire configuration is declarative (Kustomize) and GitOps-ready (ArgoCD), making it reproducible and auditable.

## Prerequisites

| Component | Version | Notes |
|-----------|---------|-------|
| **Red Hat OpenShift** | 4.20+ | Cluster with at least one NVIDIA GPU node |
| **Red Hat OpenShift AI** | 3.3+ | Installed via OperatorHub | Kueue enabled in the Openshift AI istance (state: Unmanaged)
| **Red Hat Build of Kueue** | Enabled in the `DataScienceCluster` CR | Provides `ClusterQueue`, `LocalQueue`, `Cohort`, `WorkloadPriorityClass` |
| **Grafana Operator** | Community edition (v5) | Installed via OperatorHub in the `grafana` namespace |
| **NVIDIA GPU Operator** | Compatible with cluster GPUs | Provides the `nvidia.com/gpu` resource and node labels |

## Architecture Overview

```
                         ┌──────────────┐
                         │  gpu-cohort   │
                         │   (Cohort)    │
                         │  4 GPUs total │
                         └──────┬───────┘
                                │
                ┌───────────────┼───────────────┐
                │                               │
       ┌────────┴─────────┐           ┌─────────┴────────┐
       │   inference-cq    │           │   training-cq     │
       │  (ClusterQueue)   │           │  (ClusterQueue)   │
       │                   │           │                   │
       │  Guaranteed: 3 GPU│           │  Guaranteed: 1 GPU│
       │  Borrow:  up to 1 │           │  Borrow:  up to 2 │
       │  Max:     4 GPUs  │           │  Max:     3 GPUs  │
       │  Priority: 1000   │           │  Priority: 100    │
       └────────┬──────────┘           └─────────┬─────────┘
                │                                │
       ┌────────┴──────────┐           ┌─────────┴─────────┐
       │   team-inference   │           │   team-training    │
       │   (LocalQueue)     │           │   (LocalQueue)     │
       │   ns: team-inference│          │   ns: team-training│
       └────────┬──────────┘           └─────────┬─────────┘
                │                                │
       ┌────────┴──────────┐           ┌─────────┴─────────┐
       │   highpriority     │           │   lowpriority      │
       │ (HardwareProfile)  │           │ (HardwareProfile)  │
       │   GPU: 1-4         │           │   GPU: 1-3         │
       └───────────────────┘           └────────────────────┘
```

## GPU Quota Allocation

| ClusterQueue | Guaranteed GPUs | Borrowing Limit | Max Usable | Priority |
|---|---|---|---|---|
| `inference-cq` | 3 | 1 | 4 | 1000 (high) |
| `training-cq` | 1 | 2 | 3 | 100 (low) |
| **Cohort total** | **4** | | | |

### Scheduling Scenarios

| Scenario | Inference GPUs | Training GPUs | Total |
|----------|---------------|---------------|-------|
| Only inference active | 4 (3 guaranteed + 1 borrowed) | 0 | 4 |
| Only training active | 0 | 3 (1 guaranteed + 2 borrowed) | 3 |
| Both teams active | 3 (guaranteed) | 1 (guaranteed) | 4 |

### Preemption Rules

- **Inference can preempt training to borrow GPUs.** The `borrowWithinCohort: LowerPriority` policy with `maxPriorityThreshold: 100` allows inference (priority 1000) to evict training workloads (priority 100) when it needs to borrow from the cohort.
- **Inference can always reclaim its lent quota.** With `reclaimWithinCohort: Any`, if training borrowed inference's idle GPUs and inference needs them back, inference preempts any training workload regardless of priority.
- **Training cannot preempt inference.** Training has no `borrowWithinCohort` policy, so it can only use GPUs that are genuinely idle in the cohort.
- **Training reclaim is limited.** With `reclaimWithinCohort: LowerPriority`, training can only reclaim its lent quota by preempting lower-priority workloads — which excludes inference (priority 1000 > 100).

## Resource Flavors

| Flavor | Description | Node Selector | Tolerations |
|--------|-------------|---------------|-------------|
| `gpu-flavor` | NVIDIA GPU nodes | `nvidia.com/gpu.present: "true"` | `nvidia.com/gpu` (NoSchedule) |
| `default-flavor` | CPU/Memory (any node) | None | None |

## Hardware Profiles (OpenShift AI Dashboard)

| Profile | Display Name | Queue | Priority Class | GPU Range |
|---------|-------------|-------|----------------|-----------|
| `highpriority` | highPriority | `team-inference` | `inference-priority` | 1-4 |
| `lowpriority` | lowPriority | `team-training` | `training-priority` | 1-3 |

Hardware profiles integrate with the OpenShift AI dashboard, allowing data scientists to select a profile when launching a workbench or serving runtime. The platform automatically routes the workload to the correct LocalQueue with the appropriate priority class.

## Monitoring

The `monitoring/` layer deploys a Grafana dashboard and supporting resources for observing Kueue behavior in real time. The dashboard (**Kueue — OpenShift AI Monitoring**) includes the following sections:

| Section | Panels |
|---------|--------|
| **Workload Overview** | Active Workloads, Pending Workloads, Total Admitted (24h), Evictions (24h) |
| **Workload Lifecycle** | Workloads by ClusterQueue (time series), ClusterQueue Status (table) |
| **Admission Latency** | Wait Time p50/p95/p99, Latency p95 by ClusterQueue |
| **Resource Utilization & Quota** | Usage / Nominal Quota, Usage Over Time, Borrowing Usage / Limit, Nominal Quota Configuration |
| **Throughput & Rates** | Admission & Eviction Rate per minute, Hourly Admission Throughput |
| **Prometheus Scrape Health** | Scrape Target Status, Scrape Duration, Scraped Samples |

### Monitoring Components

| Resource | Purpose |
|----------|---------|
| `grafana-dashboard.yaml` | `GrafanaDashboard` CR with inline JSON definition |
| `grafana-datasource.yaml` | `GrafanaDatasource` CR pointing at the OpenShift Thanos Querier |
| `serviceMonitor.yaml` | `ServiceMonitor` for Prometheus to scrape Kueue controller metrics |
| `kueue-prometheus-rules.yaml` | `PrometheusRule` with recording rules and alerts |
| `serviceAccount.yaml` | `RoleBinding` granting Prometheus read access to Kueue metrics |

## Directory Structure

```
base/
├── kustomization.yaml
├── namespaces/
│   ├── kustomization.yaml
│   ├── team-inference-namespace.yaml
│   └── team-training-namespace.yaml
├── resourceflavor/
│   ├── kustomization.yaml
│   ├── default-resourceflavor.yaml
│   └── ResourceFlavor-4GPUs.yaml
├── kueue/
│   ├── queue/
│   │   ├── kustomization.yaml
│   │   ├── cohort.yaml
│   │   ├── clusterqueue-inference.yaml
│   │   ├── clusterqueue-training.yaml
│   │   ├── localqueue-inference.yaml
│   │   └── localqueue-training.yaml
│   ├── priorityclass/
│   │   ├── kustomization.yaml
│   │   ├── inferencePriorityClass.yaml
│   │   └── trainingPriorityClass.yaml
│   └── hardware-profiles/
│       ├── kustomization.yaml
│       ├── highPriority-hardwareProfile.yaml
│       └── lowPriority-hardwareProfile.yaml
└── monitoring/
    ├── kustomization.yaml
    ├── grafana-dashboard.yaml
    ├── grafana-datasource.yaml
    ├── kueue-openshift-ai-dashboard.json
    ├── kueue-prometheus-rules.yaml
    ├── serviceMonitor.yaml
    └── serviceAccount.yaml
```

## Deployment

### Via ArgoCD (recommended)

Create an ArgoCD Application pointing to this repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gpu-as-a-service
  namespace: openshift-gitops
spec:
  destination:
    server: https://kubernetes.default.svc
  source:
    repoURL: <your-repo-url>
    path: base
    targetRevision: main
  project: default
```

### Manual

```bash
oc apply -k base/
```

### Post-deployment: Configure the Grafana Datasource Token

The Grafana datasource needs a ServiceAccount token to authenticate against the OpenShift Thanos Querier. After deployment, run:

```bash
TOKEN=$(oc get secret grafana-sa-token -n kueue-monitoring -o jsonpath='{.data.token}' | base64 -d)

oc patch grafanadatasource prometheus-thanos -n grafana \
  --type merge \
  -p "{\"spec\":{\"datasource\":{\"secureJsonData\":{\"httpHeaderValue1\":\"Bearer ${TOKEN}\"}}}}"
```
