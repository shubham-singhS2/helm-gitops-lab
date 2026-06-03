# Helm GitOps Lab with ArgoCD, Multi-Environment Deployments and ApplicationSets

## Introduction

This repository demonstrates advanced GitOps concepts using:

* Helm Charts
* ArgoCD
* Multi-Environment Deployments
* ApplicationSets
* Git Directory Generators

This repository is a continuation of the foundational GitOps concepts covered in the `gitops-lab` repository.

While the previous repository focused on plain Kubernetes manifests, this repository introduces Helm-based application deployments and scalable GitOps patterns used in real-world Kubernetes environments.

---

# Learning Objectives

By completing this lab, you will learn:

* Helm Chart fundamentals
* Helm deployments through ArgoCD
* Environment-specific configuration using values files
* Multi-environment GitOps workflows
* ApplicationSet fundamentals
* List Generators
* Git Directory Generators
* Automatic Application Discovery
* ArgoCD Go Templates
* Scalable GitOps repository structures

---

# Architecture Overview

Traditional ArgoCD Application:

```text
Git Repository
      ↓
ArgoCD Application
      ↓
Helm Chart
      ↓
Kubernetes
```

ApplicationSet Architecture:

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

ApplicationSets do not deploy workloads directly.

ApplicationSets create Applications.

Applications deploy workloads.

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

Before starting this lab:

* Kubernetes Cluster
* ArgoCD Installed
* Helm Installed
* kubectl Configured
* GitHub Account

Recommended cluster:

```text
1 Control Plane
2 Worker Nodes
```

---

# Creating Helm Chart

Create chart:

```bash
helm create nginx-helm
```

Generated structure:

```text
nginx-helm/
├── Chart.yaml
├── values.yaml
├── templates/
└── charts/
```

Verify rendering:

```bash
helm template nginx-helm .
```

This command generates Kubernetes manifests from the Helm chart.

---

# Deploying Helm Chart Through ArgoCD

Create namespace:

```bash
kubectl create namespace helm-demo
```

Create ArgoCD Application:

```text
Application Name:
nginx-helm
```

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

Verify:

```bash
kubectl get all -n helm-demo
```

---

# Understanding values.yaml

The file:

```yaml
replicaCount: 2
```

controls:

```text
Deployment Replica Count
```

Changing:

```yaml
replicaCount: 5
```

and pushing to Git automatically updates the deployment through ArgoCD.

No kubectl apply or helm upgrade commands are required.

---

# Multi-Environment Deployments

Real-world deployments usually require:

```text
Development
QA
Production
Performance Testing
```

Instead of creating multiple charts, one chart is reused with multiple values files.

Example:

Development:

```yaml
replicaCount: 1
```

QA:

```yaml
replicaCount: 2
```

Production:

```yaml
replicaCount: 5
```

Performance:

```yaml
replicaCount: 3
```

---

# Environment Directory Structure

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

Each directory represents one environment.

---

# ApplicationSet Fundamentals

ApplicationSets automate Application creation.

Without ApplicationSets:

```text
Create Application:
nginx-dev

Create Application:
nginx-qa

Create Application:
nginx-prod

Create Application:
nginx-perf
```

With ApplicationSets:

```text
ApplicationSet
      ↓
Creates
      ↓
nginx-dev
nginx-qa
nginx-prod
nginx-perf
```

Automatically.

---

# List Generator

The first ApplicationSet implementation used a List Generator.

Example:

```yaml
elements:
  - env: dev
  - env: qa
  - env: prod
```

Each entry generated a separate Application.

This approach is simple but requires manual updates whenever a new environment is added.

---

# Git Directory Generator

To improve scalability, the List Generator was replaced with a Git Directory Generator.

Example:

```yaml
directories:
  - path: environment/*
```

ArgoCD scans Git and discovers:

```text
environment/dev
environment/qa
environment/prod
environment/perf
```

Applications are generated automatically.

---

# Go Templates

ArgoCD 3.x uses Go Templates.

Enable:

```yaml
goTemplate: true
```

Example:

```yaml
name: nginx-{{ .path.basename }}
```

and

```yaml
namespace: {{ .path.basename }}
```

Directory:

```text
environment/prod
```

Generates:

```text
Application:
nginx-prod

Namespace:
prod
```

---

# Automatic Application Discovery

Adding a new environment requires only:

```text
Create Directory
Commit
Push
```

Example:

```text
environment/perf/
```

ApplicationSet automatically creates:

```text
nginx-perf
```

No ApplicationSet modification is required.

---

# GitOps Workflow

Developer:

```text
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

No manual Kubernetes commands are necessary.

---

# Key Concepts Learned

## Helm

Package manager for Kubernetes applications.

## values.yaml

Environment-specific configuration.

## Application

Deploys workloads.

## ApplicationSet

Creates Applications.

## List Generator

Manually defined Application generation.

## Git Directory Generator

Git-driven Application generation.

## Go Templates

Dynamic Application creation based on Git paths.

## Auto Sync

Automatically applies Git changes.

## Self Heal

Automatically corrects cluster drift.

## Prune

Automatically removes deleted resources.

---

# Real-World Benefits

This architecture enables:

* Scalable environment management
* Consistent deployments
* Git as the source of truth
* Reduced manual operations
* Easier cluster management
* Faster onboarding of new environments

---

# Learning Outcomes

After completing this lab, you should understand:

* Helm-based GitOps workflows
* Multi-environment deployments
* Environment-specific values files
* ApplicationSet architecture
* Git Directory Generators
* Go Templates
* Automatic Application generation
* Scalable ArgoCD repository design

---

# Next Steps

After mastering this repository, continue with:

* App of Apps Pattern
* ArgoCD Projects
* Multi-Cluster GitOps
* ArgoCD Image Updater
* CI/CD + GitOps Integration
* Cluster Bootstrap Repositories

The next repository in the learning journey is:

```text
cluster-bootstrap
```

which introduces the App of Apps pattern and cluster bootstrapping using ArgoCD.
