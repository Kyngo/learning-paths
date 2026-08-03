---
title: "DevOps & CI/CD"
weight: 82
bookCollapseSection: true
---

# DevOps & CI/CD

Master the practices, tools, and culture that enable teams to deliver software rapidly, reliably, and sustainably. This path covers the philosophy behind DevOps, the mechanics of continuous integration and delivery, and the tooling ecosystem that makes modern software delivery possible.

## What You'll Learn

- DevOps principles, culture, and how to measure engineering effectiveness
- Continuous Integration fundamentals — from commit to verified build
- Deployment strategies that minimize risk and maximize confidence
- GitLab CI and GitHub Actions — the two dominant CI/CD platforms in detail
- Artifact management, container registries, and versioning in pipelines
- Infrastructure automation, GitOps, and policy as code
- Observability practices that close the feedback loop on deployments

## Planned Sections

| # | Section | Focus |
|---|---------|-------|
| 1 | [DevOps Principles]({{< relref "01-devops-principles" >}}) | Culture, DORA metrics, shift-left, feedback loops |
| 2 | [CI Fundamentals]({{< relref "02-ci-fundamentals" >}}) | Pipeline anatomy, triggers, build/test/lint stages |
| 3 | [CD & Deployment Strategies]({{< relref "03-cd-deployment-strategies" >}}) | Blue-green, canary, rolling, feature flags, rollback |
| 4 | [GitLab CI]({{< relref "04-gitlab-ci" >}}) | .gitlab-ci.yml, stages, runners, DAG pipelines |
| 5 | [GitHub Actions]({{< relref "05-github-actions" >}}) | Workflows, matrix builds, reusable workflows |
| 6 | [Artifact Management]({{< relref "06-artifact-management" >}}) | Registries, versioning, image tagging, scanning |
| 7 | [Infrastructure Automation]({{< relref "07-infrastructure-automation" >}}) | GitOps, IaC in pipelines, policy as code |
| 8 | [Observability in CI/CD]({{< relref "08-observability-in-cicd" >}}) | Pipeline metrics, DORA tracking, post-deploy verification |

## Prerequisites

- **Git fundamentals** — branching, merging, pull/merge requests (see [Tools path]({{< relref "/paths/tools/01-git" >}}))
- **Basic terminal/shell** usage (see [Scripting path]({{< relref "/paths/scripting" >}}))
- **Docker basics** — building and running containers (see [Tools path]({{< relref "/paths/tools/02-docker" >}}))
- **One programming language** — examples use Python, JavaScript, and Go but concepts are language-agnostic
- **Basic cloud awareness** — understanding of compute, storage, and networking concepts

## Who This Path Is For

- Developers who want to understand and improve their team's delivery pipeline
- Operations engineers transitioning to a DevOps or platform engineering role
- Team leads and architects designing CI/CD strategies for their organisations
- Anyone preparing for DevOps-related certifications or interviews
