---
title: "Secure Kubernetes on AWS"
summary: "Security hardened k3s cluster on EC2 built entirely with Terraform. Security group least privilege, scoped IAM (no wildcards), CloudTrail audit logging, K8s RBAC, default deny NetworkPolicy, and non root pod security contexts. Vulnerability Prioritization Scorer deployed as a real workload."
date: "Jun 09 2026"
draft: false
tags:
- Kubernetes
- AWS
- Terraform
- Security
- k3s
- RBAC
- CloudTrail
- GuardDuty
demoUrl: "http://13.60.220.109:30001"
repoUrl: "https://github.com/TamasCzaban/secure-k8s-demo"
---

Security hardened k3s cluster on EC2, built entirely as Terraform and committed K8s manifests. Defence in depth across the network, IAM, runtime, and audit layers on a t3.micro. No manual console steps.

## Security Controls

| Layer | Control | Implementation |
|-------|---------|----------------|
| Network | Security group least privilege | SSH (22) and k3s API (6443) restricted to a single operator `/32` CIDR, no `0.0.0.0/0` |
| Network | K8s NetworkPolicy | Default deny on `dev` namespace. TCP 80 ingress and DNS/HTTPS egress only |
| IAM | Scoped EC2 role | No wildcard `Action` or `Resource`. Explicit `ec2:DescribeInstances` and scoped CloudWatch Logs |
| IAM | K8s RBAC | `app-sa` can `get`/`list` pods. Cannot `delete` pods or read secrets |
| Runtime | Non root container | nginx runs as uid 101 (`runAsNonRoot: true`), capabilities dropped, privilege escalation blocked |
| Audit | CloudTrail | Management events to encrypted, versioned, public access blocked S3 |
| Threat detection | GuardDuty | Detector configured via Terraform. Account activation pending (tracked in issue #1) |

## Architecture

```
Your IP (/32)
    │
    ▼
[Security Group]  SSH:22, API:6443 — operator CIDR only
    │
    ▼
[EC2 t3.micro]  Amazon Linux 2023 · IAM role (scoped)
    │
    ▼
[k3s]  Single-node Kubernetes (v1.35)
    │
    ▼
[Namespace: dev]
    ├── ServiceAccount: app-sa  (automount disabled)
    ├── Role: app-reader        (get/list pods only)
    ├── NetworkPolicy: deny-all-except-http
    └── Deployments: nginx:1.27-alpine + vuln-prioritization-scorer (Streamlit)
```

## Key Design Decisions

**k3s over EKS.** EKS charges $70/month for the control plane. k3s is free, installs in two minutes, and RBAC, NetworkPolicy, and pod security contexts work the same way at the manifest level.

**No wildcard IAM.** The EC2 instance role uses explicit `Action` and `Resource` ARNs. Same principle as the K8s RBAC Role, applied one layer up in the stack.

**emptyDir for nginx writable paths.** nginx:alpine needs `/var/cache/nginx` and `/var/run` writable at startup. `emptyDir` volumes at those paths keep the root filesystem read only without changing the security context.

**Security group over SSH bastion.** Operator CIDR on ports 22 and 6443 gives the same isolation as a bastion host. For a single node demo, it's simpler.

## Live Infrastructure

Region: eu-north-1 (Stockholm) · EC2: t3.micro · k3s v1.35.5+k3s1

The Vulnerability Prioritization Scorer runs on the cluster as a NodePort service on 30001.
