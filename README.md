# Helm GitOps Lab with ArgoCD, Multi-Environment Deployments and ApplicationSets

## Introduction

This repository demonstrates Helm-based GitOps workflows using ArgoCD.

The lab progresses from deploying a single Helm chart to managing multiple environments using ApplicationSets and Git Directory Generators.

Topics covered:

* Helm Charts
* ArgoCD Applications
* Auto Sync
* Self Heal
* Prune
* Multi-Environment Deployments
* ApplicationSets
* List Generators
* Git Directory Generators
* Go Templates
* Automatic Application Discovery

This repository is a continuation of the foundational GitOps concepts introduced in the earlier `gitops-lab` repository.

---

# Architecture Overview

## Traditional ArgoCD Application

```text
Git Repository
      ↓
ArgoCD Application
      ↓
Helm Chart
      ↓
Kubernetes
```

## ApplicationSet Architecture

```text
Git Repository
      ↓
ApplicationSet
      ↓
Applications
      ↓
Helm Charts
      ↓
Kubernetes
```

Important:

```text
ApplicationSet
      ↓
Creates Applications

Application
      ↓
Deploys Workloads
```

---

# Repository Structure

```text
helm-gitops-lab/

├── environment
│   ├── dev
│   │   └── values.yaml
│   │
│   ├── qa
│   │   └── values.yaml
│   │
│   ├── prod
│   │   └── values.yaml
│   │
│   └── perf
│       └── values.yaml
│
└── nginx-helm
    ├── Chart.yaml
    ├── values.yaml
    ├── templates
    └── charts
```

---

# Prerequisites

Before starting:

* Kubernetes Cluster
* ArgoCD Installed
* Helm Installed
* kubectl Configured
* GitHub Account

Recommended Lab Cluster:

```text
Control Plane: 1
Worker Nodes: 2
```

---

# Step 1 - Create Helm Chart

Generate chart:

```bash
helm create nginx-helm
```

Verify structure:

```bash
tree nginx-helm
```

Render templates:

```bash
cd nginx-helm

helm template nginx-helm .
```

Expected:

```text
Kubernetes YAML manifests rendered successfully
```

This confirms the chart is valid.

---

# Step 2 - Deploy Helm Chart Through ArgoCD

Create namespace:

```bash
kubectl create namespace helm-demo
```

Create ArgoCD Application.

Source:

```text
Repository:
https://github.com/<username>/helm-gitops-lab.git

Path:
nginx-helm
```

Destination:

```text
Namespace:
helm-demo
```

Enable:

```text
Auto Sync
Self Heal
Prune
```

Verify deployment:

```bash
kubectl get all -n helm-demo
```

Expected:

```text
Deployment
ReplicaSet
Pods
Service
```

---

# Step 3 - Understanding values.yaml

Default:

```yaml
replicaCount: 2
```

Verify:

```bash
kubectl get deploy -n helm-demo
```

Expected:

```text
READY   2/2
```

Change:

```yaml
replicaCount: 5
```

Commit and push.

Wait for ArgoCD reconciliation.

Verify:

```bash
kubectl get deploy -n helm-demo
```

Expected:

```text
READY   5/5
```

This demonstrates GitOps-driven deployment updates.

---

# Step 4 - Multi Environment Deployments

Real environments:

```text
dev
qa
prod
perf
```

Example replica counts:

| Environment | Replicas |
| ----------- | -------- |
| dev         | 1        |
| qa          | 2        |
| prod        | 5        |
| perf        | 3        |

Environment configuration:

```text
environment/

├── dev
│   └── values.yaml

├── qa
│   └── values.yaml

├── prod
│   └── values.yaml

└── perf
    └── values.yaml
```

Example:

```yaml
replicaCount: 1
```

inside:

```text
environment/dev/values.yaml
```

---

# Step 5 - ApplicationSet Using List Generator

Create namespaces:

```bash
kubectl create namespace dev
kubectl create namespace qa
kubectl create namespace prod
kubectl create namespace perf
```

Create:

```bash
nano nginx-appset-list.yaml
```

Paste the List Generator YAML.

Apply:

```bash
kubectl apply -f nginx-appset-list.yaml
```

Verify:

```bash
kubectl get applicationsets -n argocd
```

Expected:

```text
nginx-appset
```

Verify Applications:

```bash
kubectl get applications -n argocd
```

Expected:

```text
nginx-dev
nginx-qa
nginx-prod
nginx-perf
```

Verify deployments:

```bash
kubectl get deploy -A
```

Applications should deploy into their respective namespaces.

---

# List Generator YAML

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: nginx-appset
  namespace: argocd

spec:
  generators:
  - list:
      elements:
      - env: dev
        namespace: dev
        valuesFile: values-dev.yaml

      - env: qa
        namespace: qa
        valuesFile: values-qa.yaml

      - env: prod
        namespace: prod
        valuesFile: values-prod.yaml

      - env: perf
        namespace: perf
        valuesFile: values-perf.yaml

  template:
    metadata:
      name: 'nginx-{{env}}'

    spec:
      project: default

      source:
        repoURL: https://github.com/shubham-singhS2/helm-gitops-lab.git
        targetRevision: HEAD
        path: nginx-helm

        helm:
          valueFiles:
            - '{{valuesFile}}'

      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'

      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

# Limitation of List Generator

Adding:

```text
staging
```

requires:

1. Creating values file
2. Creating namespace
3. Editing ApplicationSet YAML
4. Reapplying ApplicationSet

Not scalable.

---

# Step 6 - Git Directory Generator

Delete previous ApplicationSet:

```bash
kubectl delete applicationset nginx-appset -n argocd
```

Create:

```bash
nano nginx-appset-git.yaml
```

Paste Git Directory Generator YAML.

Apply:

```bash
kubectl apply -f nginx-appset-git.yaml
```

Verify:

```bash
kubectl get applicationsets -n argocd
```

Expected:

```text
nginx-appset
```

Verify:

```bash
kubectl get applications -n argocd
```

Expected:

```text
nginx-dev
nginx-qa
nginx-prod
nginx-perf
```

---

# Git Directory Generator YAML

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: nginx-appset
  namespace: argocd

spec:
  goTemplate: true

  generators:
  - git:
      repoURL: https://github.com/shubham-singhS2/helm-gitops-lab.git
      revision: HEAD

      directories:
      - path: environment/*

  template:
    metadata:
      name: 'nginx-{{ .path.basename }}'

    spec:
      project: default

      source:
        repoURL: https://github.com/shubham-singhS2/helm-gitops-lab.git
        targetRevision: HEAD
        path: nginx-helm

        helm:
          valueFiles:
            - '../{{ .path.path }}/values.yaml'

      destination:
        server: https://kubernetes.default.svc
        namespace: '{{ .path.basename }}'

      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

# Step 7 - Verify Go Templates

Inspect generated Application:

```bash
kubectl get application nginx-dev -n argocd -o yaml
```

Verify:

```yaml
namespace: dev
```

and:

```yaml
valueFiles:
- ../environment/dev/values.yaml
```

This confirms template rendering is working.

---

# Step 8 - Automatic Application Discovery

Create new environment:

```bash
mkdir -p environment/staging
```

Create:

```bash
nano environment/staging/values.yaml
```

Example:

```yaml
replicaCount: 4
```

Commit and push.

Create namespace:

```bash
kubectl create namespace staging
```

Wait for reconciliation.

Verify:

```bash
kubectl get applications -n argocd
```

Expected:

```text
nginx-staging
```

without modifying the ApplicationSet.

---

# Common Issues Encountered

## Issue 1 - ArgoCD 3.x Go Templates

Incorrect:

```yaml
{{path.basename}}
```

Correct:

```yaml
{{ .path.basename }}
```

Incorrect:

```yaml
{{path.path}}
```

Correct:

```yaml
{{ .path.path }}
```

Required:

```yaml
goTemplate: true
```

Without this configuration, generated Applications may fail with:

```text
{{path.path}}/values.yaml
no such file or directory
```

---

# GitOps Workflow

```text
Developer
    ↓
Edit values.yaml
    ↓
Commit
    ↓
Push
```

ArgoCD:

```text
Detect Change
      ↓
Sync
      ↓
Deploy
```

No manual deployment commands required.

---

# Key Concepts Learned

* Helm
* values.yaml
* ArgoCD Applications
* ApplicationSets
* List Generators
* Git Directory Generators
* Go Templates
* Auto Sync
* Self Heal
* Prune
* Automatic Application Discovery

---

# Learning Outcomes

After completing this lab, you should understand:

* Helm-based GitOps
* Multi-environment deployments
* Environment-specific values files
* ApplicationSet architecture
* Git Directory Generators
* Go Templates
* Automatic Application generation
* Scalable GitOps repository design

---

# Next Steps

After completing this repository:

* App of Apps Pattern
* ArgoCD Projects
* Multi-Cluster GitOps
* ArgoCD Image Updater
* CI/CD + GitOps Integration
* Cluster Bootstrap Repositories

The next repository in this learning journey is:

```text
cluster-bootstrap
```

which introduces the App of Apps pattern and full cluster bootstrapping using ArgoCD.
