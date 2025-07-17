---
sidebar_position: 
title: Pod Sandboxing on AKS
sidebar_label: "Pod Sandboxing"
---

Pod Sandboxing on AKS, currently in Public Preview, provides an isolation boundary between the container application and the shared kernel and compute resources of the container host such as CPU, memory, and networking. 

Traditionally, Kubernetes deployments rely on namespace isolation. Namespaces don't protect against kernel-level attacks, because:

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

# Objectives

Pod Sandboxing on AKS is currently in Public Preview.
This session is designed to allow you to a more hands on view of how pods isolated in a Pod Sandbox behave. You will be able to observe the potential benefits that Kata podsmight bring to your workloads.

As you progress through the workshop, you will learn how to:

- Set up a workplace environment
- Deploying pods (Kata and non-Kata)
- Explore resource isolation between Kata pods and non-Kata pods utilizing the open-source [Kubernetes Goat](https://madhuakula.com/kubernetes-goat/) repo.

# Prerequisites

Before starting this lab, please ensure your lab environment is set up properly. Follow the guide [here](https://azure-samples.github.io/aks-labs/docs/getting-started/setting-up-lab-environment/).

Please also familiarize yourself with the basic concepts laid out, and ensure you have the prerequisites laid out in the Microsoft Learn page for [Pod Sandboxing on AKS](https://learn.microsoft.com/en-us/azure/aks/use-pod-sandboxing).

As we are using Kubernetes Goat, there are some requirements that it requires as well. [Please ensure you have those set up](https://madhuakula.com/kubernetes-goat/docs/how-to-run/kubernetes-goat-on-microsoft-azure-kubernetes-cluster).

## Setting up Pod Sandboxing on your AKS Node Pool(s)

You can either spin up a new cluster or add node pools to an existing cluster to experiment with the Pod Sandboxing feature.

## Deploy a new cluster

For the purposes of this lab, we will create a new cluster with Pod Sandboxing enabled.

When deploying a new AKS cluster with Pod Sandboxing enabled, you can simply call on `az aks create` to create a new cluster. When setting up the clusters, however, please specify the following parameters:

- `--workload-runtime`: Should be `KataMshvVmIsolation` to enable the Pod Sandboxing feature in the node pool. With this parameter selected, the subsequent two parameters must fit the requirements, or else an error will be returned.
- `--os-sku`: Ensure you select `AzureLinux`, as that is the only OS that supports this feature currently.
- `--node-vm-size`: Please ensure you select a VM size that both supports generation 2 VMs and nested virtualization. The [Dsv3 series](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/dsv3-series?tabs=sizebasic) of VMs are a great example.

Here is an example of a command that will create a cluster, *myAKSCluster*, with 3 nodes in *myResourceGroup*:

```azurecli-interactive
az aks create
    --name myAKSCluster \
    --resource-group myResourceGroup \
    --os-sku AzureLinux \
    --workload-runtime KataMshvVmIsolation \
    --node-vm-size Standard_D4s_v3 \
    --node-count 3 \
    --generate-ssh-keys
```

Once the cluster is created, ensure you get the access credentials for the Kubernetes cluster.

```azurecli-interactive
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

## Setting up Kubernetes GOAT

Once your cluster is up and running, navigate to a location that you would like the Kubernetes Goat files to sit at.
`cd <location_path_of_your_choice>`

Once there, clone the repo:
`git clone https://github.com/madhuakula/kubernetes-goat.git`

Then navigate into the folder it is installed at:
`cd kubernetes-goat/`

Kubernetes GOAT offers a [number of scenarios](https://madhuakula.com/kubernetes-goat/docs/scenarios) that one can go through to test their pod's security. For the purposes of this lab, we will only go through a few scenarios that will best illustrate the effect of Pod Sandboxes.

## Privileged Kata Pods

At points in this lab, we will be running some privileged Kata pods. This requires you to [configure your containerd Kata runtime appropriately](https://www.bing.com/search?pglt=171&q=privileged+kata+pod&cvid=8bb71690a33842b0b1e7ac56f70499c0&gs_lcrp=EgRlZGdlKgYIABBFGDkyBggAEEUYOTIGCAEQABhAMgYIAhAAGEAyBggDEAAYQDIGCAQQABhAMgYIBRAAGEAyBggGEAAYQDIGCAcQABhAMgYICBAAGEAyCAgJEOkHGPxV0gEIMTYwM2owajGoAgCwAgA&FORM=ANNAB1&PC=U531).

A quick method to do so will be via launching a debug pod. First, use `kubectl get nodes` to get the name of your node, and plug it into the command below:
`kubectl debug node/<node_name> -it --image=ubuntu:22.04 -- bash`

Once in your node, you can use an editor of your choice. If you don't have it, you can also quickly install `vim` via `apt update && apt install vim -y`

Then edit your containerd config: 
- First, run `vim /host/etc/containerd/config.toml`.
- Look for the section under `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]`, hit "O" for a new line, and insert `privileged_without_host_devices = true`
- Press "esc", then enter `:wq` to save your changes.
- Restart containerd ` chroot /host systemctl restart containerd`

# Scenarios

## Container Escape

The [Container Escape to the Host System](https://madhuakula.com/kubernetes-goat/docs/scenarios/scenario-4/container-escape-to-the-host-system-in-kubernetes-containers/welcome) scenario illustrates what might happen if permissions that are not required are given to users. In this scenario, you can observe a privileged container escape to gain access to the host system.

### Non-Kata Demonstration

We will only deploy the specific pod we need for this scenario. In this case, we will first deploy the pods corresponding to the container escape scenario:
`kubectl apply -f scenarios/system-monitor/`

Make sure that the pod is up and running. You can do so via `kubectl get pods`.

Once the pod is up and running, we can exec directly into the pod to run some tests. 
- Grab your pod name: `kubectl get pods`
- Exec into your pod: `kubectl exec -it <pod-name> -- bash`

Navigating to the localhost port should bring up a web terminal. This is where we will do our testing. 

To see the current capabilities of our pod, run `capsh --print`.

At a glance, you should be able to see many capabilities that you would associate with a privileged pod, such as `CAP_SYS_ADMIN`.

Running `mount` will display the mounted filesystems and their mount points. Take a second to look through the mounted host points. We should see some points that would bring up concerns; a line similar to `http://dev/sda3%20on%20etc/hostname%20type%20ext4%20(rw,relatime)`, for instance, would hint that the host's root filesystem (`/dev/sda`) is mounted at `/host` inside the container, a potential major security vulnerability.
- You can further explore the mounted filesystem by navigating to the `/host-system/` path via `ls /host-system/`

Using `chroot`, we can get direct access to host system privileges: `chroot /host-system bash`

You should now see that you have access to host system resources by running `crictl pods`.

### Kata Demonstration

Now we want to see if running the pod as a Kata pod brings about any differences. 

Let's first delete our previous deployment. The simpliest way would be via `kubectl delete -f scenarios/system-monitor/`

Let's head over to the deployment YAML, which should be located at `...\kubernetes-goat\scenarios\system-monitor`. 

In the runtime spec, all you will need to do is add in one line:

```YAML
spec:
  selector:
    matchLabels:
      app: system-monitor
  template:
    metadata:
      labels:
        app: system-monitor
    spec:
      runtimeClassName: kata-mshv-vm-isolation # one line change to make this pod a Kata sandbox.
      hostPID: true
      hostIPC: true
      #hostNetwork: true
```

Let's deploy the updated pod, and run the same tests as before:
- We'll first exec into our pod (`kubectl exec -it <pod-name> -- bash`)
- Run `capsh --print`. This time around, we should notice much less capabilities show up.
- Run `mount`. Like `capsh`, we should see considerable less mounted resources than before.
- Let's attempt to get direct access to host system privileges again, and see what host resources we have access to.
   - We'll first run `chroot /host-system bash`.
   - Next, we'll run `crictl pods`. This time around, we should be greeted with an error stating that the file/directory does not exist. From the Kata pod, we won't have access to host resources.
 
With Kata pods, we can see that we generally have less (or in some cases, no) access to resources that we previously had in a non-Kata pod!

To clean up this first demonstration, run `kubectl delete -f scenarios/system-monitor/` again.

## Scenario #2 
