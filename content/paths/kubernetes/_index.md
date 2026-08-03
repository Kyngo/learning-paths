---
title: "Kubernetes"
weight: 87
bookCollapseSection: true
---

# Kubernetes

Container orchestration at scale — from pods to production-grade clusters.

## Overview

Kubernetes (K8s) is the industry-standard platform for automating deployment, scaling, and management of containerized applications. This learning path takes you from understanding why Kubernetes exists through to operating production clusters with confidence.

You will learn how to model workloads declaratively, manage networking and storage, secure multi-tenant clusters, package applications with Helm, and operate the platform day-to-day including monitoring, debugging, and disaster recovery.

## Prerequisites

Before starting this path, you should be comfortable with:

- **Containers** — Docker basics (building images, running containers, Dockerfiles)
- **Linux fundamentals** — shell navigation, processes, networking basics
- **YAML** — syntax and structure (used extensively in Kubernetes manifests)
- **Networking concepts** — IP addresses, ports, DNS, HTTP/HTTPS
- **Cloud basics** — familiarity with at least one cloud provider (AWS, GCP, or Azure)

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Core Concepts]({{< relref "01-core-concepts" >}}) | Cluster architecture, pods, namespaces, labels and selectors |
| 2 | [Workloads]({{< relref "02-workloads" >}}) | Deployments, StatefulSets, DaemonSets, Jobs, CronJobs |
| 3 | [Networking]({{< relref "03-networking" >}}) | Services, Ingress, NetworkPolicies, DNS, service mesh |
| 4 | [Storage]({{< relref "04-storage" >}}) | Volumes, PersistentVolumes, StorageClasses, ConfigMaps, Secrets |
| 5 | [Configuration & Scaling]({{< relref "05-configuration" >}}) | Resource management, autoscaling, scheduling, affinity rules |
| 6 | [Helm & Packaging]({{< relref "06-helm-and-packaging" >}}) | Helm charts, Kustomize, release management |
| 7 | [Security]({{< relref "07-security" >}}) | RBAC, PodSecurityStandards, secrets management, admission controllers |
| 8 | [Operations]({{< relref "08-operations" >}}) | kubectl, debugging, monitoring, logging, upgrades, backup, managed K8s |

## Learning Approach

Each section includes:

- Conceptual explanations with architecture diagrams
- Real YAML manifests you can apply to a cluster
- Tables summarizing key resources and their purposes
- Key takeaways to reinforce critical concepts

## Recommended Lab Environment

To follow along practically, set up one of:

- **minikube** — single-node local cluster (good for learning)
- **kind** (Kubernetes in Docker) — lightweight multi-node clusters
- **k3s** — minimal Kubernetes distribution (good for edge/local)
- **EKS/GKE/AKS** — managed cloud clusters (for production-like experience)
