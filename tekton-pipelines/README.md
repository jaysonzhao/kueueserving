# Kueue Tekton Pipelines for Multi-Tenant Quota Management and Model Deployment

## Overview

This directory contains Tekton Pipelines and Tasks for automating:
1. **Tenant Quota Management** - Cluster admins provision capacity for new tenants
2. **Model Deployment** - Developers deploy models with Kueue workload management

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Tenant Quota Management Pipeline                   │
│                                                                     │
│  Parameters: tenant-name, cpu-quota, memory-quota, pod-quota,       │
│              gpu-quota                                               │
│                                                                     │
│  ┌─────────────────┐    ┌─────────────────┐                        │
│  │ Create Default  │    │ Create GPU      │                        │
│  │ ResourceFlavor  │    │ ResourceFlavor  │                        │
│  └────────┬────────┘    └────────┬────────┘                        │
│           │                      │                                  │
│           └──────────┬───────────┘                                 │
│                      ▼                                              │
│              ┌───────────────┐                                     │
│              │ Create        │                                     │
│              │ ClusterQueue  │                                     │
│              └───────────────┘                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    Model Deployment Pipeline                         │
│                                                                     │
│  Parameters: model-name, namespace, cluster-queue-name,             │
│              image-url, cpu-request, memory-request, gpu-request    │
│                                                                     │
│  ┌─────────────────┐                                               │
│  │ Create          │                                               │
│  │ LocalQueue      │                                               │
│  └────────┬────────┘                                               │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                               │
│  │ Deploy Model    │                                               │
│  │ (Deployment or  │                                               │
│  │  InferenceSvc)  │                                               │
│  │ + Kueue Label   │                                               │
│  └─────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────┘
```

## Directory Structure

```
tekton-pipelines/
├── README.md
├── rbac/
│   ├── 01-serviceaccount.yaml          # Pipeline service accounts
│   ├── 02-clusterrole-quota-admin.yaml  # Cluster admin role for quota mgmt
│   └── 03-role-model-deployment.yaml    # Role for model deployment
├── tasks/
│   ├── 01-create-resource-flavor.yaml  # Create ResourceFlavor task
│   ├── 02-create-cluster-queue.yaml     # Create ClusterQueue task
│   ├── 03-create-local-queue.yaml      # Create LocalQueue task
│   └── 04-deploy-model.yaml            # Deploy model task
├── pipelines/
│   ├── 01-tenant-quota-management.yaml # Admin pipeline for quota setup
│   └── 02-model-deployment.yaml        # Developer pipeline for deployment
└── samples/
    ├── 01-pipelinerun-quota-tenant-a.yaml   # Sample quota PipelineRuns
    └── 02-pipelinerun-model-deployment.yaml # Sample deployment PipelineRuns
```

## Prerequisites

1. OpenShift Pipelines (Tekton) operator installed
2. Red Hat build of Kueue installed
3. `kueue.openshift.io/managed=true` label on target namespaces

## Deployment Instructions

### 1. Apply RBAC Configuration

```bash
# Apply ServiceAccounts
oc apply -f rbac/01-serviceaccount.yaml

# Apply ClusterRole and ClusterRoleBinding for quota management
oc apply -f rbac/02-clusterrole-quota-admin.yaml

# Apply Role and RoleBinding for model deployment
oc apply -f rbac/03-role-model-deployment.yaml
```

### 2. Apply Tasks

```bash
oc apply -f tasks/01-create-resource-flavor.yaml
oc apply -f tasks/02-create-cluster-queue.yaml
oc apply -f tasks/03-create-local-queue.yaml
oc apply -f tasks/04-deploy-model.yaml
```

### 3. Apply Pipelines

```bash
oc apply -f pipelines/01-tenant-quota-management.yaml
oc apply -f pipelines/02-model-deployment.yaml
```

## Usage

### Pipeline 1: Tenant Quota Management

Run as cluster admin to provision quota for a new tenant:

```bash
# GPU-enabled tenant
oc create -f samples/01-pipelinerun-quota-tenant-a.yaml -n openshift-pipelines

# CPU-only tenant
oc create -f samples/01-pipelinerun-quota-tenant-b.yaml -n openshift-pipelines

# Or via tkn CLI
tkn pipeline start tenant-quota-management \
  -n openshift-pipelines \
  -p tenant-name=tenant-c \
  -p cpu-quota=16 \
  -p memory-quota=64Gi \
  -p pod-quota=30 \
  -p enable-gpu=true \
  -p gpu-quota=8 \
  -s quota-management-pipeline-sa
```

**Expected Output:**
- `tenant-c-default` ResourceFlavor
- `tenant-c-gpu` ResourceFlavor (if GPU enabled)
- `tenant-c-cluster-queue` ClusterQueue with quotas

### Pipeline 2: Model Deployment

Run as developer/tenant to deploy a model:

```bash
# Deployment (non-KServe)
oc create -f samples/02-pipelinerun-model-deployment.yaml -n openshift-pipelines

# Or via tkn CLI
tkn pipeline start model-deployment \
  -n openshift-pipelines \
  -p model-name=my-llm \
  -p namespace=tenant-namespace \
  -p cluster-queue-name=tenant-c-cluster-queue \
  -p image-url=quay.io/model/vllm:v1 \
  -p cpu-request=4 \
  -p memory-request=8Gi \
  -p gpu-request=1 \
  -p gpu-limit=1 \
  -p deployment-type=deployment \
  -s model-deployment-pipeline-sa
```

**Expected Output:**
- `my-llm-queue` LocalQueue in tenant namespace
- `my-llm` Deployment with Kueue label

## Pipeline Parameters

### Tenant Quota Management Pipeline

| Parameter | Description | Required |
|-----------|-------------|----------|
| `tenant-name` | Name for the tenant (used in resource names) | Yes |
| `cpu-quota` | CPU quota (e.g., "8") | Yes |
| `memory-quota` | Memory quota (e.g., "40Gi") | Yes |
| `pod-quota` | Pod quota (e.g., "10") | Yes |
| `enable-gpu` | Enable GPU quota ("true"/"false") | No (default: false) |
| `gpu-quota` | GPU quota (e.g., "4") | No (default: 0) |
| `gpu-node-label-key` | Node label key for GPU nodes | No |
| `gpu-node-label-value` | Node label value for GPU nodes | No |

### Model Deployment Pipeline

| Parameter | Description | Required |
|-----------|-------------|----------|
| `model-name` | Name of the model/deployment | Yes |
| `namespace` | Target namespace | Yes |
| `cluster-queue-name` | ClusterQueue to attach LocalQueue | Yes |
| `image-url` | Container image URL | Yes |
| `cpu-request` | CPU request | No (default: 1) |
| `memory-request` | Memory request | No (default: 2Gi) |
| `cpu-limit` | CPU limit | No (default: 2) |
| `memory-limit` | Memory limit | No (default: 4Gi) |
| `gpu-request` | GPU request (0 or 1) | No (default: 0) |
| `gpu-limit` | GPU limit | No (default: 0) |
| `deployment-type` | "deployment" or "inferenceservice" | No (default: deployment) |
| `model-format` | Model format for KServe | No (default: vLLM) |
| `storage-uri` | Model storage URI | No |

## Verification

### Check PipelineRun Status

```bash
# List PipelineRuns
oc get pipelinerun -n openshift-pipelines

# Watch PipelineRun
oc get pipelinerun quota-management-tenant-a -n openshift-pipelines -w
```

### Verify Created Resources

```bash
# Check ClusterQueue
oc get clusterqueue

# Check ResourceFlavor
oc get resourceflavor

# Check LocalQueue
oc get localqueue -n <namespace>

# Check Kueue Workloads
oc get workloads.kueue.x-k8s.io -n <namespace>
```

## Security Notes

- The `quota-management-pipeline-sa` has cluster-wide permissions to create ClusterQueues
- The `model-deployment-pipeline-sa` has namespace-scoped permissions for LocalQueues and Deployments
- Both service accounts should be granted least-privilege access to only the namespaces they manage
- For production, consider using `PodIdentity` or `WorkspaceBinding` for secure credential management
