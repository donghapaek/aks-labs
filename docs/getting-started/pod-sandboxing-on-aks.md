---
sidebar_position: 
title: Pod Sandboxing on AKS
sidebar_label: "Pod Sandboxing"
---

To help secure and protect your container workloads from untrusted or potentially malicious code, AKS now includes a mechanism called Pod Sandboxing (preview). Pod Sandboxing provides an isolation boundary between the container application and the shared kernel and compute resources of the container host such as CPU, memory, and networking. Pod Sandboxing complements other security measures or data protection controls with your overall architecture to help you meet regulatory, industry, or governance compliance requirements for securing sensitive information.

Traditionally, Kubernetes deployments rely on namespace isolations Namespaces don't protect against kernel-level attacks, because:

- All containers share the same kernel.
- vulnerability in the kernel (e.g., dirty COW, runc escape, Spectre/Meltdown side channels) can allow cross-container attacks.
- If a container escapes its namespace, it could compromise other containers or the host.

This is especially risky in multi-tenant clusters, where:

- One tenant’s workload might be malicious or compromised.
- Traditional namespace isolation won't stop a kernel-level exploit from jumping across tenants.

Kata Containers are ideal when you need strong tenant isolation—for example:

- SaaS platforms with untrusted workloads
- Regulated environments (e.g., finance, healthcare)
- Running low-trust 3rd-party code
- Clusters with shared responsibility models

Pod Sandboxing complements other security measures or data protection controls with your overall architecture to help you meet regulatory, industry, or governance compliance requirements for securing sensitive information. Pod Sandboxing could allow you to run untrusted or potentially malicious code securely, isolate workloads of different tenants from one another on the same cluster, or allow you to securely mix and match trusted and untrusted workloads on the same cluster. 

# Objectives

Pod Sandboxing on AKS is currently in Public Preview.

As you progress through the workshop, you will learn how to:

- Set up a workplace environment
- Deploying pods (Kata and non-Kata)
- Explore resource isolation between Kata pods and non-Kata pods
- Set resource constraints on Kata pods

## Prerequisites

Before starting this lab, please ensure your lab environment is set up properly. Follow the guide [here](https://azure-samples.github.io/aks-labs/docs/getting-started/setting-up-lab-environment/).

Please familiarize yourself with the basic concepts from the Microsft learn page.

[Pod Sandboxing on AKS](https://learn.microsoft.com/en-us/azure/aks/use-pod-sandboxing) explains basic concepts of the feature and how to enable the preview flags for AKS.
[AKS fundamentals](https://azure-samples.github.io/aks-labs/docs/getting-started/k8s-aks-fundamentals) for key concepts on AKS

### Setting up workplace

```bash
kubectl get pods
```

then delete the pod(s) in question

```bash
az extension add --name aks-preview
```

In order to demonstrate the difference between Kata and non Kata pods, we will deploy multiple pods, both with Pod Sandboxing enabled and disabled.

## Demonstrating compute/network isolation

### Kernel

Each Kata pod runs inside it's own lightweight VM, each with their own dedicated guest kernel. This ensures that:

- Kernel exploits in one pod do not effect another pod nor the host.
- Stronger isolation from other pods.

### Networking

Some of the key characteristics in Pod Sandboxing for networking are:

- Has its own virtual NIC (network interface card).
- Uses a separate network namespace from the host and other pods.
- Can have dedicated firewall rules, routing, and DNS settings.
- Isolated from other pods while running on same subnet.

Traffic between sandboxed pods on the same node is routed through virtual NICs and virtual switches inside the hypervisor layer. This provides strong isolation even though the subnet is shared.

### What you can do and can't do 

can do multi container pods

### Commonly asked questions

### Takeaways

### Confidential Pods on AKS

