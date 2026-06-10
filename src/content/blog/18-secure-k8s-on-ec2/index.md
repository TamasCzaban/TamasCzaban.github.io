---
title: "A Security Hardened k3s Cluster on AWS"
summary: "Security hardened k3s cluster on EC2 built with Terraform. Covers security group least privilege, scoped IAM, CloudTrail, RBAC, NetworkPolicy, non root pods, and the three things that broke."
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
---

This project showcases a [secure k8s demo](https://github.com/TamasCzaban/secure-k8s-demo), a k3s cluster on EC2, built entirely with Terraform, with a real application deployed on top.

Anyone can claim AWS experience. I wanted a repo that showed the reasoning behind each control, not just that it was present. Network level (security groups and K8s NetworkPolicy), IAM (cloud role and K8s RBAC), runtime (pod security context), and audit (CloudTrail). Several are a single YAML field. Getting all of them applied together without leaving gaps is the actual work.

## The stack

k3s rather than EKS. EKS charges about $70/month for the control plane. k3s is free, installs in two minutes, and RBAC, NetworkPolicy, and PodSecurityContext work identically at the manifest level.

All infrastructure is Terraform. All K8s manifests are committed. No manual console steps. Clone the repo, run `terraform apply`, get the same cluster.

## Security layer by layer

### Network security group

`0.0.0.0/0` on port 22 is the tutorial default. The security group allows SSH (22) and the k3s API (6443) only from my operator `/32` CIDR. Nothing else reaches the instance.

### Network Kubernetes NetworkPolicy

The `dev` namespace has a default deny policy. No pod can receive or send traffic without an explicit allow rule. The only rules that exist:

- TCP 80 ingress (the nginx demo pod)
- TCP 8501 ingress (the Streamlit app)
- DNS UDP/TCP 53 to kube-dns
- TCP 443 for Streamlit's outbound calls to the NVD and EPSS APIs

A compromised container in `dev` can't reach other workloads or exfiltrate over arbitrary ports.

### IAM scoped EC2 role

The EC2 instance role has no `*` in `Action` or `Resource`. Two permissions `ec2:DescribeInstances` and `logs:PutLogEvents` scoped to a specific ARN. Same principle as the RBAC Role, one layer up. A compromised instance has a very small blast radius via the metadata endpoint.

### IAM Kubernetes RBAC

The `app-sa` ServiceAccount is bound to a Role with `get` and `list` on pods in `dev`. That's the full permission set.

```bash
kubectl -n dev auth can-i get pods    --as=system:serviceaccount:dev:app-sa  # yes
kubectl -n dev auth can-i delete pods --as=system:serviceaccount:dev:app-sa  # no
kubectl -n dev auth can-i get secrets --as=system:serviceaccount:dev:app-sa  # no
```

`automountServiceAccountToken: false` on the ServiceAccount. Pods don't get the token unless they explicitly request it.

### Runtime non root pod security context

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 101
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

nginx runs as uid 101. No capabilities. No privilege escalation. A compromised container runs as a non root user with no Linux capabilities.

### Audit CloudTrail

`ec2:RunInstances`, `iam:AttachRolePolicy`, every management plane API call lands in S3. The bucket is AES 256 encrypted, versioned, and public access blocked. Trail covers all regions.

### Threat detection GuardDuty

The Terraform code is there. The detector resource is defined. I hit `SubscriptionRequiredException`. GuardDuty requires a 24 hour activation window on new accounts before the API accepts detector creation. Tracked in [issue #1](https://github.com/TamasCzaban/secure-k8s-demo/issues/1). The Terraform is correct. The account needs to age.

## What broke

### t3.micro ran out of memory

k3s sits at around 598 MB of RAM. A t3.micro has 1 GB total. I deployed the first pod and the node started OOMKilling. The API server went unreachable.

A 2 GB swapfile fixed it. Swap adds latency on a burstable instance so this isn't a production setup, but for a two pod demo cluster it holds. A t3.small (2 GB RAM) avoids the problem entirely.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### nginx CrashLoopBackOff with runAsUser 101

nginx:alpine starts as root, binds port 80, then drops to the worker user. Set `runAsNonRoot: true` and `runAsUser: 101`, and that initial root step fails. uid 101 can't `mkdir /var/cache/nginx/client_temp` on the container's root filesystem.

Relaxing the security context would fix the crash and break the whole point. `emptyDir` volumes at `/var/cache/nginx` and `/var/run` give the process writable paths without touching the security context:

```yaml
volumeMounts:
  - name: nginx-cache
    mountPath: /var/cache/nginx
  - name: nginx-run
    mountPath: /var/run
volumes:
  - name: nginx-cache
    emptyDir: {}
  - name: nginx-run
    emptyDir: {}
```

`emptyDir` volumes are writable by the pod's user. The root filesystem stays read only. Any official image that needs writable paths at startup gets `emptyDir` mounts at those paths, not a relaxed security context.

## What's running on the cluster

The nginx pod is a sanity check. The actual workload is the [Vulnerability Prioritization Scorer](https://github.com/TamasCzaban/vuln-prioritization-scorer), a Streamlit app that enriches CVE lists with NVD and EPSS data, scores them via a configurable composite formula, and exports ranked reports. NodePort service on 30001.

## Caveats

Single node cluster, free tier EC2, Stockholm. No TLS on the K8s API, no cert manager, no Ingress controller, no Velero, no cluster autoscaler.

Every security control works as documented. The RBAC boundaries hold under testing. The audit trail runs. Production clusters have cert manager, Velero, and a proper Ingress stack on top of these controls. This cluster has the controls.

The repo is at [github.com/TamasCzaban/secure-k8s-demo](https://github.com/TamasCzaban/secure-k8s-demo). The scorer runs at [scorer.czaban.dev](https://scorer.czaban.dev).
