# Kueue LLM Queue Management Test

## Overview

This directory contains the YAML configurations for testing Kueue workload management on OpenShift with an LLM InferenceService deployment.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     OpenShift Cluster                           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Namespace: ai501                                      │   │
│  │                                                         │   │
│  │  ┌─────────────────┐    ┌─────────────────────────────┐ │   │
│  │  │ InferenceService│───▶│ Deployment: llama-32        │ │   │
│  │  │ llama-32       │    │ (underlying Pod managed     │ │   │
│  │  │ (KServe)       │    │  by Kueue)                 │ │   │
│  │  └─────────────────┘    └─────────────────────────────┘ │   │
│  │         │                        │                      │   │
│  │         ▼                        ▼                      │   │
│  │  ┌─────────────────────────────────────────────────┐ │   │
│  │  │ Label: kueue.x-k8s.io/queue-name: llm-local-queue│ │   │
│  │  └─────────────────────────────────────────────────┘ │   │
│  │                          │                              │   │
│  └──────────────────────────│──────────────────────────────┘   │
│                             │                                   │
│  ┌──────────────────────────│──────────────────────────────┐   │
│  │  LocalQueue: llm-local-queue                           │   │
│  │  Namespace: ai501                                     │   │
│  │  ─────────────────────────────────────────────────────│   │
│  │  spec.clusterQueue: llm-cluster-queue                  │   │
│  └──────────────────────────│──────────────────────────────┘   │
│                             │                                   │
│  ┌──────────────────────────│──────────────────────────────┐   │
│  │  ClusterQueue: llm-cluster-queue                        │   │
│  │  ─────────────────────────────────────────────────────│   │
│  │  ResourceGroups:                                        │   │
│  │    default-flavor:  8 CPU, 40Gi memory, 3 pods        │   │
│  │    gpu-flavor:       4 GPUs, 8 CPU, 40Gi memory       │   │
│  └──────────────────────────│──────────────────────────────┘   │
│                             │                                   │
│  ┌──────────────────────────│──────────────────────────────┐   │
│  │  ResourceFlavors:                                          │   │
│  │    default-flavor:  (no labels - matches all nodes)       │   │
│  │    gpu-flavor:      nodeLabels: nvidia.com/gpu.present=true│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Components

### 1. Kueue CR (`00-kueue-cr.yaml`)
Operator configuration enabling workload integrations:
- `Pod` - Direct pod management
- `Deployment` - ReplicaSet-based workloads
- `BatchJob` - Batch job support
- `RayJob` - Ray distributed training

### 2. ResourceFlavors

| Flavor | Purpose | Node Matching |
|--------|---------|---------------|
| `default-flavor` | CPU/memory workloads | All nodes |
| `gpu-flavor` | GPU workloads | `nvidia.com/gpu.present=true` |

### 3. ClusterQueue (`llm-cluster-queue`)

Defines resource quotas across two flavors:

| Resource | default-flavor | gpu-flavor |
|----------|---------------|------------|
| CPU | 8 | 8 |
| Memory | 40Gi | 40Gi |
| Pods | 3 | - |
| nvidia.com/gpu | - | 4 |

### 4. LocalQueue (`llm-local-queue`)
- Namespace: `ai501`
- Points to: `llm-cluster-queue`

### 5. Namespace (`ai501`)
- Key label: `kueue.openshift.io/managed=true`
- Enables Kueue management for all workloads in the namespace

### 6. InferenceService (`llama-32`)
- KServe InferenceService
- Label: `kueue.x-k8s.io/queue-name: llm-local-queue`
- Resources: 1 GPU, 4 CPU, 20Gi memory (limits)

## Test Results

### Test 1: Pod Quota Enforcement
**Goal:** Verify only 3 pods can be admitted (quota limit)

| Job | CPU | Status |
|-----|-----|--------|
| kueue-pod-test-1 | 100m | ✅ Admitted |
| kueue-pod-test-2 | 100m | ✅ Admitted |
| kueue-pod-test-3 | 100m | ✅ Admitted |
| kueue-pod-test-4 | 100m | ⏳ Suspended (pending) |

**Result:** PASS - 4th job correctly suspended when pod quota exceeded.

### Test 2: CPU Quota Enforcement
**Goal:** Verify jobs exceeding CPU quota are suspended

| Job | CPU Request | Status |
|-----|-------------|--------|
| kueue-cpu-test | 16 CPU | ⏳ Suspended (exceeds 8 CPU limit) |

**Result:** PASS - Job correctly suspended for exceeding single-job CPU limit.

### Test 3: InferenceService Management
**Goal:** Verify llama-32 deployment is managed by Kueue

| Check | Result |
|-------|--------|
| Kueue workload created | ✅ `pod-llama-32-predictor-xxx-xxx` |
| Workload admitted | ✅ `Admitted: True` |
| Quota reserved | ✅ `nvidia.com/gpu=1` from gpu-flavor |
| Pod running | ✅ Running on GPU node |

**Result:** PASS - llama-32 is fully managed by Kueue.

## ClusterQueue Status

Current resource usage:
```
flavorsUsage:
  default-flavor:
    cpu: 0
    memory: 0
    pods: 0
  gpu-flavor:
    cpu: 1010m (1.01 cores)
    memory: 8207Mi (~8Gi)
    nvidia.com/gpu: 1
```

## Key Commands

### Apply all configurations
```bash
oc apply -f 00-kueue-cr.yaml
oc apply -f 01-resourceflavor-default.yaml
oc apply -f 02-resourceflavor-gpu.yaml
oc apply -f 03-clusterqueue-llm.yaml
oc apply -f 04-localqueue-llm.yaml
oc apply -f 05-namespace-ai501.yaml
oc apply -f 06-inferenceservice-llama32.yaml
```

### Verify Kueue is managing workloads
```bash
# Check workloads
oc get workloads.kueue.x-k8s.io -n ai501

# Check ClusterQueue status
oc get clusterqueue llm-cluster-queue -o yaml | grep -A 20 "status:"

# Check LocalQueue status
oc get localqueue llm-local-queue -n ai501
```

### Trigger Kueue management for existing InferenceService
```bash
# Delete the existing pod to trigger recreation with Kueue
oc delete pod -n ai501 -l serving.kserve.io/inferenceservice=llama-32

# Watch workload creation
oc get workloads.kueue.x-k8s.io -n ai501 -w
```

## Notes

- The `gpu-flavor` ResourceFlavor uses `nvidia.com/gpu.present=true` node label to match GPU nodes
- Pods must have GPU tolerations to run on GPU nodes (already configured in InferenceService)
- The `kueue.openshift.io/managed=true` label on the namespace is required for OpenShift Kueue to manage workloads
- Quota is enforced at admission time; pending workloads are suspended until resources become available
