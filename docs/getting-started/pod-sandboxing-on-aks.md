---
sidebar_position: 
title: Pod Sandboxing on AKS
sidebar_label: "Pod Sandboxing"
---

Pod Sandboxing on AKS, currently in Public Preview, provides an isolation boundary between the container application and the shared kernel and compute resources of the container host such as CPU, memory, and networking. 

Pod Sandboxing complements other security measures or data protection controls with your overall architecture to help you meet regulatory, industry, or governance compliance requirements for securing sensitive information. Pod Sandboxing could allow you to run untrusted or potentially malicious code securely, isolate workloads of different tenants from one another on the same cluster, or allow you to securely mix and match trusted and untrusted workloads on the same cluster. 

# Objectives

This session is designed to help you understand how to isolate workloads using the lightweight Kata virtual machines introduced by Pod Sandboxing. The goal is to allow you to get hands-on with the feature and different use cases in a clusters.

As you progress through the workshop, you will learn how to:

- Set up a workplace environment
- Deploying pods (Kata and non-Kata)
- Explore resource isolation between Kata pods and non-Kata pods
- Set resource constraints on Kata pods

# Prerequisites

Before starting this lab, please ensure your lab environment is set up properly. Follow the guide [here](https://azure-samples.github.io/aks-labs/docs/getting-started/setting-up-lab-environment/).

Please also familiarize yourself with the basic concepts laid out, and ensure you have the prerequisities laid out in the Microsoft Learn page for [Pod Sandboxing on AKS](https://learn.microsoft.com/en-us/azure/aks/use-pod-sandboxing).

# Setting up Pod Sandboxing on your AKS Node Pool(s)

You have can either spin up a new cluster or add nodepools to an existing cluster to experiment with the Pod Sandboxing feature.

## Deploy a new cluster

If you want to deploy a new AKS cluster with Pod Sandboxing enabled, you can simply call on `az aks create` to create a new cluster. When setting up the clusters, however, please specify the following parameters:

- `--workload-runtime`: Should be `KataMshvVmIsolation` to enable the Pod Sandboxing feature in the node pool. With this parameter selected, the subsequent two parameters must fit the requirements, or else an error will be returned.
- `--os-sku`: Ensure you select `AzureLinux`, as that is the only OS that supports this feature currently.
- `--node-vm-size`: Please ensure you select a VM size that both supports generation 2 VMs and nested virtualization. The [Dsv3 series](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/dsv3-series?tabs=sizebasic) of VMs are a great example.

Here is an example of a command that will create a cluster, *myAKSCluster*, with 1 node in *myResourceGroup*:

```azurecli-interactive
az aks create
    --name myAKSCluster \
    --resource-group myResourceGroup \
    --os-sku AzureLinux \
    --workload-runtime KataMshvVmIsolation \
    --node-vm-size Standard_D4s_v3 \
    --node-count 1 \
    --generate-ssh-keys
```

Once the cluster is created, ensure you get the access credentials for the Kubernetes cluser.

```azurecli-interactive
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

## Deploy to an existing cluster

The prerequisites above hold true if you intend to deploy Pod Sandboxing to an existing cluster. The difference is instead of creating a cluster via `az aks create`, you will simply add node pools to an existing cluster using `az aks nodepool add`. The parameter requirements laid out above remain the same.

In order to demonstrate the difference between Kata and non Kata pods, we will deploy multiple pods, both with Pod Sandboxing enabled and disabled.

This example adds a node pool to *myAKSCluster* with one node pool, *newnodepool* in *myResourceGroup*:

```azurecli
az aks nodepool add \
    --cluster-name myAKSCluster \
    --resource-group myResourceGroup \
    --name nodepool2 \ 
    --os-sku AzureLinux \ 
    --workload-runtime KataMshvVmIsolation \
    --node-vm-size Standard_D4s_v3
```

# Deploying your pods

<!--  Going over how to deploy Kata and non-Kata pods. -->

## Exploring Kata Pods

<!-- Showcase isolation of Kata pods -->

# Resource Isolations in Kata Pods

<!-- Set up resource limits on Kata pods, showcase a Kata pod using those resources -->

## Exploring compute/network isolation for Kata pods

<!-- [Networing] Pinging from Kata to non-Kata pod (this should work by default -->
<!-- [Networking] Setting up subnets for Kata pods different from non-Kata pods -->
<!-- [Compute] Simulate a noisy neighbor on a Kata pod vs. non-Kata pod -->

# Limitations

<!-- Note on performance limitations -->
<!-- Note on privileged pods, but capabailities should still work (showcase this). This should get fixed come GA. -->
<!-- Note on peripheral devices -->

# Summary

🎉 Congratulations on completing this lab! You should now have some hands-on experience with **Pod Sandboxing** on AKS, with a solid foundation on utilizing Kata containers to isolate your workloads from one another and the host. 

## What we learned

In this lab, we you:
✅ Set up Pod Sandboxing on AKS.
✅ Deployed both sandboxed and non-sandboxed pods on a cluster.
✅ Explored isolation provided by Pod Sandboxing.
✅ Simulated workload stress scenarios, and seeing how Pod Sandboxing can help isolate other workloads from the fallout.
✅ Explored limitations of Pod Sandboxing.

## Next steps

# Cleanup (Optional)

If you are done and no longer need the cluster(s) created during this lab, you can delete your cluster with the following command:

```azurecli-interactive
az aks delete --resource-group myResourceGroup --name myAKSCluster
```

Alternatively, if you want to keep your cluster but want to get rid of any specific pods, you can first find the pods in your cluster:

```bash
kubectl get pods
```

then delete the pod(s) in question

```bash
kubectl delete pod <pod_name>
```



