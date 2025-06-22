---
sidebar_position: 
title: Pod Sandboxing on AKS
sidebar_label: "Pod Sandboxing"
---

To help secure and protect your container workloads from untrusted or potentially malicious code, AKS now includes a mechanism called Pod Sandboxing (preview). Pod Sandboxing provides an isolation boundary between the container application and the shared kernel and compute resources of the container host such as CPU, memory, and networking. Pod Sandboxing complements other security measures or data protection controls with your overall architecture to help you meet regulatory, industry, or governance compliance requirements for securing sensitive information.

## Objectives

This session is designed to help you understand how to isolate workloads in multi-tenant environments using lightweight virtual machines, and how to get hands-on with it in your own clusters.

Pod Sandboxing on AKS is currently in Public Preview.

## Objectives

As you progress through the workshop, you will learn how to:

- Setting up workplace environment
- Spinning up pods (Kata and non-Kata)
- Showing the differences between the pods 
- Networking between Kata pods and non-Kata pods
- Setting resource constraints on Kata pods

## Prerequisites

Please familiarize yourself with the basic consepts from the Microsft learn page.

[Pod Sandboxing on AKS](https://learn.microsoft.com/en-us/azure/aks/use-pod-sandboxing) explains basic concepts of the feature and how to enable the preview flags for AKS.

### Setting up workplace

```bash
az login
```

Run the following command to register preview features.

```bash
az extension add --name aks-preview
```

In order to demonstrate the difference between Kata and non Kata pods, we will deploy multiple pods, both with Pod Sandboxing enabled and disabled.

### Demonstrating compute/network isolation

### Resource Isolations in Kata Pods

### What you can do and can't do 

## Takeaways

### Confidential Pods on AKS

