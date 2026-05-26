---
title: "OpenStack On-Call to Kubernetes: The Concepts Map Directly"
summary: "Ericsson Tier-2 24/7, 95% on-time delivery, 90% queue utilisation — diagnosing Nova compute scheduling failures, Neutron flapping interfaces, and Swift storage under a structured RCA framework. Here's why that maps straight to Kubernetes."
date: "May 26 2026"
tags: ["Kubernetes", "Automation", "CI/CD", "DevEx"]
draft: false
---

Two years at Ericsson, Tier-2 24/7 on-call for enterprise OpenStack infrastructure. Nova compute scheduling failures, Neutron east-west networking incidents, Swift object storage degradation. 95% on-time delivery across the queue. 90% utilisation while managing concurrent critical cases.

The common reaction from infrastructure recruiters is that OpenStack is a gap in the context of a Kubernetes role. The mapping is direct.

## What Tier-2 means

Tier-1 is basic triage: restart the service, check the logs, confirm the alert is real. By the time a case reaches Tier-2, that has already failed. Something has broken in a way that basic triage cannot resolve, there are real users affected right now, and the on-call engineer has to figure out what it is under pressure.

The scope at Ericsson covered three major subsystems: Nova for compute scheduling, Neutron for east-west networking, and Swift for object storage. The on-call engineer does not have the luxury of reproducing the issue in a controlled environment. The problem is happening in production, with a client's SLA clock running.

## The structured RCA format

After every incident, I wrote a structured RCA report with five fields:

**Mechanism**: what broke at the technical level.

**Trigger**: what caused it to break at this moment.

**Contributing factors**: what made it worse than it needed to be.

**Fix**: what resolved it.

**Prevention**: what stops it from recurring.

This is not bureaucracy. "The compute scheduler was misconfigured" does not help the next on-call engineer. "Nova's `ram_allocation_ratio` was set to 1.0 on a subset of compute hosts after the maintenance window, causing those hosts to appear full to the scheduler while physically underutilised" is a runbook entry that prevents a repeat. Repeat failures dropped once this practice was in place because prevention recommendations got implemented rather than filed.

## The Kubernetes mapping

The conceptual surface of Kubernetes looks different from OpenStack. The API objects have different names, the YAML has a different shape. The operational problems are the same problems.

**Nova compute scheduling = Kubernetes scheduler.** Both place a workload on a host satisfying resource requests, affinity rules, and node conditions. When Nova fails to schedule a VM, the diagnostic path is: check host availability, check resource quotas, check affinity filter configurations. When a Kubernetes pod stays in `Pending`, the diagnostic path is: check node capacity, check resource requests against allocatable resources, check node affinity and taint/toleration rules. The mechanism is different. The reasoning is the same.

**Neutron flapping interfaces = CNI plugin behaviour and pod network policies.** A Neutron flapping interface investigation starts with OVS port state, binding on the physical host, and VLAN tag consistency across the path. A CNI-level networking issue in Kubernetes starts with interface state on the node, the pod's network namespace, and whether the CNI plugin's route tables are consistent. The specific tools differ. The diagnostic method does not.

**Swift object storage = PersistentVolumes and object storage backends.** Swift failures are often quorum issues: a ring degradation where enough nodes are unavailable to satisfy write consistency. Diagnosing that requires understanding how consistency is maintained across nodes and what happens when the quorum threshold is not met. The same reasoning applies to distributed storage backends in Kubernetes and to understanding when a PVC in `Pending` state reflects a missing provisioner versus an underlying storage degradation.

## The transferable skill

The Kubernetes API can be learned from documentation. A week of hands-on practice is enough to build working fluency with the object model. The Helm chart format, the kubectl verb syntax, the structure of a Deployment and a Service, these are learnable from a book.

Incident response at scale is not learnable from a book. Diagnosing an actively failing system under pressure, with incomplete information, while a client's SLA clock runs, while managing concurrent cases, and then writing an RCA that prevents recurrence rather than just documenting what happened, that comes from doing it. Two years of Tier-2 24/7 on-call is the practice environment for that skill.

The OpenStack API surface is not Kubernetes. The way I reason about a failing system is the same skill. Nobody learns the RCA framework from documentation. They learn it from incident after incident until the diagnostic reasoning becomes the default mode of thinking about infrastructure.

That is what the Ericsson years produced.
