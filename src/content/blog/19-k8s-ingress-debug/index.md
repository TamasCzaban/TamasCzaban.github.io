---
title: "scorer.czaban.dev Was 404ing and kubectl Wouldn't Respond"
summary: "Tracing a dead subdomain through Cloudflare proxy, a missing Traefik Ingress, and a t3.micro that ran out of memory mid fix."
date: "Jun 11 2026"
draft: false
tags:
- Kubernetes
- Traefik
- Cloudflare
- AWS
- k3s
- Debugging
---

The scorer app was running fine at `http://13.60.220.109:30001`. The subdomain `scorer.czaban.dev` was returning 404. A git history rewrite the day before looked like the culprit. It wasn't.

## What was actually broken

`scorer.czaban.dev` is a Cloudflare proxied A record pointing to the EC2 IP. Cloudflare terminates HTTPS on their side and forwards plain HTTP to port 80 on the instance. Traefik sits on port 80 inside k3s and routes by hostname. For that routing to work, there needs to be a Kubernetes Ingress resource telling Traefik which service to send `scorer.czaban.dev` traffic to.

There wasn't one.

We applied the Ingress manually during the original setup and never committed it. When k3s restarted at some point, nothing persisted it. Traefik came back up with no rules for that hostname and returned its default `404 page not found`.

The git work had nothing to do with it. The Ingress was already gone.

## Confirming the path

```bash
curl -H "Host: scorer.czaban.dev" http://13.60.220.109:80/
# 404 page not found

curl http://13.60.220.109:30001/
# 200
```

Port 30001 is the NodePort and returns 200 because the pod is healthy. Port 80 is Traefik with no matching rule. The pod, DNS, and Cloudflare config were all fine. The Ingress was missing.

## kubectl wouldn't respond

The fix is a one file Ingress manifest. Applying it should take five seconds. It didn't.

```bash
sudo kubectl apply -f scorer-ingress.yaml
# net/http: TLS handshake timeout
```

The k3s API server was too slow to complete a TLS handshake. The t3.micro has 912 MB of RAM. k3s alone sits at around 440 MB. With the scorer pod, Traefik, CoreDNS, and metrics server running, the system was deep in swap. Every API call was waiting on disk I/O.

Restarting k3s made it worse. All pods restarted simultaneously, each competing for the same memory. The API server came back up but stayed unresponsive for several minutes.

k3s watches `/var/lib/rancher/k3s/server/manifests/` and applies any YAML dropped there automatically. No kubectl needed.

```bash
sudo tee /var/lib/rancher/k3s/server/manifests/scorer-ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: scorer-ingress
  namespace: dev
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: scorer.czaban.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: scorer
                port:
                  number: 8501
EOF
```

k3s picked it up. Traefik started routing. The scorer responded.

## The underlying problem

The t3.micro memory situation isn't a one off. It is the baseline state of the cluster. Any restart, any new pod, any kubectl operation that requires disk I/O from the swap saturated API server is going to be slow or fail. The swap file from blog 18 kept the node alive but it did not solve the memory constraint.

The right fix is a t3.small. 2 GB RAM, same instance family, no swap needed for this workload.

```hcl
instance_type = "t3.small"
```

`terraform apply` stops the instance, resizes it, and starts it again in about 90 seconds.

## The IP problem

Resizing an EC2 instance stops it. A stopped instance without an Elastic IP gets a new public IP on restart. The Cloudflare DNS record still points to the old IP. The fix is to add an Elastic IP before resizing so the address is stable across stop/start cycles.

```hcl
resource "aws_eip" "k3s_node" {
  instance = aws_instance.k3s_node.id
  domain   = "vpc"
  tags     = { Name = "secure-k8s-eip" }
}
```

Terraform creates and associates the EIP in the same apply as the instance type change. After that, update the Cloudflare A record to the new EIP address. Every subsequent restart keeps the same IP.

## What the Ingress commit fixes

The original cluster setup had the Ingress applied manually and not committed. That is what caused this failure. The manifest is now in the repo at `k8s/scorer-ingress.yaml` and applied via the manifests directory so it survives restarts without kubectl being available.

`scorer.czaban.dev` is running on a t3.small with a fixed IP. The repo reflects the actual state of the cluster.
