---
sidebar_position: 
title: Pod Sandboxing on AKS
sidebar_label: "Pod Sandboxing"
---

Pod Sandboxing on AKS, currently in Public Preview, provides an isolation boundary between the container application and the shared kernel and compute resources of the container host such as CPU, memory, and networking. 

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

## Objectives

Pod Sandboxing on AKS is currently in Public Preview.
This session is designed to help you understand how to isolate workloads using the lightweight Kata virtual machines introduced by Pod Sandboxing. The goal is to allow you to get hands-on with the feature and different use cases in a clusters.

As you progress through the workshop, you will learn how to:

- Set up a workplace environment
- Deploying pods (Kata and non-Kata)
- Explore resource isolation between Kata pods and non-Kata pods
- Set resource constraints on Kata pods

## Prerequisites

Before starting this lab, please ensure your lab environment is set up properly. Follow the guide [here](https://azure-samples.github.io/aks-labs/docs/getting-started/setting-up-lab-environment/).

Please also familiarize yourself with the basic concepts laid out, and ensure you have the prerequisities laid out in the Microsoft Learn page for [Pod Sandboxing on AKS](https://learn.microsoft.com/en-us/azure/aks/use-pod-sandboxing).

## Setting up Pod Sandboxing on your AKS Node Pool(s)

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


# Demonstrating compute/network isolation

## Deploying your pods

In order to demonstrate the usefullness of resource isolation, we will deploy applications to the same nodepool, only enabling Pod Sandboxing on subset of applications.

For the purposes of the demo, we will borrow one of the sample Conto projects that we can deploy on AKS [Contoso Ship Manager](https://github.com/Azure-Samples/aks-contoso-ships-sample)

<!--
I think there is already image on ACR
```azurecli
git clone https://github.com/Azure-Samples/aks-contoso-ships-sample.git
cd contoso-ships

```
-->
```azurecli
kubectl create deployment contoso-air \
--image=mcr.microsoft.com/mslearn/samples/contoso-ship-manager \
--port=3000 \
--dry-run=client \
--output yaml > manifests/contoso-air-deployment.yaml
```

Now we deploy resources with kubectl
```azurecli
kubectl apply -f manifests/contoso-air-deployment.yaml
```


## Introducing Common Vulnerabilities

We will introduce number of common vulnerabilities to the application and see if Pod Sandboxing can provide additional protection. Here are some of the examples:

:::info
We will provide the patch that you can apply to the deployment. Paragraph below is just for descriptions
:::

- Add a Privileged Capability
Modify the container spec to include elevated privileges:
```
securityContext:
  privileged: true

```
This simulates a misconfigured container that could access host resources—something Kata Containers would block.
- Mount Host Paths
Mont a sensitive host directory:
```
volumeMounts:
- name: host-root
  mountPath: /host
volumes:
- name: host-root
  hostPath:
    path: /

```
This allows the container to read host files—again, blocked in sandboxed pods.
- Expose Docker Socket
Mount the Docker socket to simulate container escape:
```
- mountPath: /var/run/docker.sock
  name: docker-socket

```
Then run a script inside the container to list or control other containers.
- Inject a Reverse Shell
Add a vulnerable endpoint in the backend API that executes shell commands:
```
[HttpPost("exec")]
public IActionResult Exec([FromBody] string cmd)
{
    var output = System.Diagnostics.Process.Start("bash", $"-c \"{cmd}\"");
    return Ok(output);
}
```
This simulates remote code execution (RCE) from a compromised container.


Once the vulnerabilities are in place, we will now demonstrate:

- Host mounts fail
- Privileged mode is blocked
- Docker socket is inaccessible
- RCE is contained within the UVM

thus protecting the workload and contain vulnerabilities to individual pod level.


### Compute isolations

Each sandboxed pod runs inside its own lightweight virtual machine (UVM), which includes its own kernel, memory, and network stack. This means that even though multiple pods may share the same subnet, their network interfaces are isolated at the virtualization layer. This prevents direct access between pods unless explicitly configured.

This makes Pod Sandboxing on AKS ideal for multi-tenant scenarios, where you have untrusted workload from tenants running on the same node.

### Network isolations

Some of the key characteristics in Pod Sandboxing for networking are:

- Has its own virtual NIC (network interface card).
- Uses a separate network namespace from the host and other pods.
- Can have dedicated firewall rules, routing, and DNS settings.
- Isolated from other pods while running on same subnet.

Traffic between sandboxed pods on the same node is routed through virtual NICs and virtual switches inside the hypervisor layer. This provides strong isolation even though the subnet is shared.

## Exploring compute/network isolation for Kata pods

<!-- [Networing] Pinging from Kata to non-Kata pod (this should work by default -->
<!-- [Networking] Setting up subnets for Kata pods different from non-Kata pods -->
<!-- [Compute] Simulate a noisy neighbor on a Kata pod vs. non-Kata pod -->

# Limitations

<!-- Note on performance limitations -->
<!-- Note on privileged pods, but capabailities should still work (showcase this). This should get fixed come GA. -->
<!-- Note on peripheral devices -->
<!-- Note on persistent storage limitations-->


# Summary

🎉 Congratulations on completing this lab! You should now have some hands-on experience with **Pod Sandboxing** on AKS, with a solid foundation on utilizing Kata containers to isolate your workloads from one another and the host. 

## What we learned

In this lab, we you:
- ✅ Set up Pod Sandboxing on AKS.
- ✅ Deployed both sandboxed and non-sandboxed pods on a cluster.
- ✅ Explored isolation provided by Pod Sandboxing.
- ✅ Simulated workload stress scenarios, and seeing how Pod Sandboxing can help isolate other workloads from the fallout.
- ✅ Explored limitations of Pod Sandboxing.

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

### Commonly asked questions

### Takeaways


