---
title: "Exploring cgroups v2 and MemoryQoS With EKS and Bottlerocket"
date: 2023-11-12T14:16:55+02:00
draft: false
---

# Exploring cgroups v2 and MemoryQoS With EKS and Bottlerocket
![Blame Midjourney for this image](https://raw.githubusercontent.com/antweiss/botllerocket-cgroupv2/main/antweiss_bottlerocket_with_memory_d4aa3a97-258b-4267-bdd7-94660146c5d1.webp)
Bottlerocket is a Linux-based operating system optimized for hosting containers. It was originally developed at AWS specifically for runnning secure and performant Kubernetes nodes. It's minimal, secure and supports atomic updates.

According to this [discussion](https://github.com/Bottlerocket-os/Bottlerocket/discussions/2874) - starting with Bottlerocket 1.13.0 (Mar 2023) new distributions will default to using Cgroups v2 interface for process organization and enforcing resource limits.

In this post I intend to explore how this works for EKS clusters running Kubernetes 1.26+ and what this change means for EKS users.

# Cgroups - An Intro

Cgroups (abbreviated from Control Groups) - is a Linux kernel feature that lies at the foundation of what we now know as Linux containers.

The feature allows to limit. account for and isolate resource usage for a collection of processes.

It was developed at Google circa 2007 and merged into Linux kernel mainline in 2008.

![Julia Evans' wonderful cgroups comic](https://wizardzines.com/images/uploads/cgroups.png)
## Cgroups and Kubernetes 

Kubernetes allows us to define resource usage for containers via the [resources](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#resources) map in the Pod API spec.
These definitions are then passed by the kubelet on to the container runtime on the node and translated into Cgroups configuration. 

Up until version 1.25 Kubernetes only supported Cgroups v1 by default. In 1.25 - stable support for Cgroups v2 was added. Now if running on a node with Cgroups v2 - the kubelet automatically identifies this and perfroms accordingly. But what does this mean for our workload configuration? In order to understand that we need to explain what Cgroups v2 is.

## Cgroups V2

Cgroups v2 was released in 2015 introducing API redesign - mainly for a unified hierarchy and improved consistency. The following diagram shows the change in how Cgroup controllers are ordered in v2 vs. v1:
![cgroup hierarchy](https://miro.medium.com/v2/resize:fit:640/format:webp/1*P7ZLLF_F4TMgGfaJ2XIfuQ.png)

According to [this architecture document](https://kubernetes.io/docs/concepts/architecture/cgroups/#:~:text=Some%20Kubernetes%20features%20exclusively%20use%20cgroup%20v2%20for%20enhanced%20resource%20management%20and%20isolation.%20For%20example%2C%20the%20MemoryQoS%20feature%20improves%20memory%20QoS%20and%20relies%20on%20cgroup%20v2%20primitives.) : *"Some Kubernetes features exclusively use cgroup v2 for enhanced resource management and isolation. For example, the MemoryQoS feature improves memory QoS and relies on cgroup v2 primitives."*

And when we look at the description of the aforementioned *MemoryQoS* feature we find out that "In cgroup v1, and prior to this feature, the container runtime never took into account and effectively ignored spec.containers[].resources.requests["memory"]." and that "Fortunately, cgroup v2 brings a new design and implementation to achieve full protection on memory... With this experimental feature, quality-of-service for pods and containers extends to cover not just CPU time but memory as well."

Well, first of all - **it's a bit shocking and even insulting to learn that container runtimes ignored our settings**! But I was also very curious to learn how this changes now that cgroups v2 support is introduced.

## MemoryQoS and Cgroups v2

According to this [page](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/):

Memory QoS uses the memory controller of cgroup v2 to guarantee memory resources in Kubernetes. Memory requests and limits of containers in pod are used to set specific interfaces `memory.min and `memory.high`` provided by the memory controller. When `memory.min`` is set to memory requests, memory resources are reserved and never reclaimed by the kernel; this is how Memory QoS ensures the availability of memory for Kubernetes pods. And if memory limits are set in the container, this means that the system needs to limit container memory usage, Memory QoS uses `memory.high` to throttle workload approaching it's memory limit, ensuring that the system is not overwhelmed by instantaneous memory allocation.
![container memory in cgroup v2](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/memory-qos-cal.svg)

This is all great! Let's now provision an EKS cluster with some Bottlerocket nodes and see how this works in practice.

To easily spin up a cluster - use the [cluster.yaml](https://github.com/antweiss/botllerocket-cgroupv2/blob/main/cluster.yaml) in the attached github repository:

generate ssh keys:
```
ssh-keygen -f ./mykey
```
and create the cluster
```
eksctl create cluster -f cluster.yaml
```
This will create a cluster with one Bottlerocket node. It also configures ssh access to the nodes by running the [Bottlerocket admin container](https://github.com/Bottlerocket-os/Bottlerocket-admin-container).

This means we can now access the node:

```bash
export NODE_IP=$(kubectl get node -oyaml | yq  '.items[].status.addresses[] | select(.type=="ExternalIP") | .address')
ssh -i mykey ec2-user@$NODE_IP

```

We get greeted with the following screen:

![](/img/bottlerocket-ssh.png)

As this says - we can get admin access to the Bottlerocket filesystem by running `sudo sheltie`. So let's do that!

```bash
[ec2-user@admin]$ sudo sheltie
[bash-5.1]$ whoami
root
```

Now we can check if we in fact have `cgroupv2` enabled:

```bash
[bash-5.1]$ stat -fc %T /sys/fs/cgroup/
cgroup2fs
```

Yup! This is cgroupv2! Were this `cgroupv1` the output would've been `tmpfs`.

## Let's Deploy a Pod

Ok, now let's deploy a pod to our node. We'll do that by creating a deployment based on the following `yaml` spec. This deploys [antweiss/busyhttp](https://github.com/antweiss/busyhttp), that I forked from [jpetazzo/busyhttp](https://github.com/jpetazzo/busyhttp) and added memory load and release endpoints to.
You'll notice that the pod runs a container with Guaranteed QoS - i.e memory and CPU limits are equal to requests:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: busyhttp
  name: busyhttp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busyhttp
  template:
    metadata:
      labels:
        app: busyhttp
    spec:
      containers:
      - image: otomato/busyhttp
        name: busyhttp
        resources:
          requests:
            memory: "200Mi"
            cpu: "250m"
          limits:
            memory: "200Mi"
            cpu: "250m"
```

This spec is found in [dep.yaml](https://github.com/antweiss/botllerocket-cgroupv2/blob/main/dep.yaml) and we can deploy it with:

```bash
kubectl apply -f dep.yaml
```

## Check the Cgroup Impact

Now let's go back to our node and see how our resource definitions are reflected in the `cgroup` config.

Back inside the `sheltie` prompt let's explore the containers running on Bottlerocket. Bottlerocket OS is using `containerd` container runtime. In order to interact with it we'll need to use [`ctr`](https://github.com/projectatomic/containerd/blob/master/docs/cli.md).

When we run `ctr help` - we get the following:

![](/img/ctr.png)

So `ctr` is unsupported. A bit discouraging, but well, it's working. Let's try to look at our containers:

```bash
bash-5.1$ ctr containers ls
CONTAINER    IMAGE    RUNTIME
```

No containers?! But I do see my pod running on the node! Where is my container? Well the answer to that is `namespaces`. Yup, just like kubernetes or linux kernel - containerd has namespaces. And all the containers executed by the kubelet live in a namespace called "k8s.io". We can see it by running:

```bash
bash-5.1$ ctr ns ls
NAME   LABELS
k8s.io
```

Ok, let's check the containers in the "k8s.io" namespace:

```bash
bash-5.1$ ctr -n k8s.io containers ls
CONTAINER                                                           IMAGE                                                                                                RUNTIME
0ed99eae66803896504d1853859d8866e00669b2610ba65cba6a17aa1300da48    112233445566.dkr.ecr.eu-central-1.amazonaws.com/eks/pause:3.1-eksbuild.1                             io.containerd.runc.v2
154d9b7b3a83e4db6e3e4ac4ac1f836321337c604c3b590b5188b7a0773bdae1    docker.io/otomato/busyhttp:latest                                                                    io.containerd.runc.v2
3b63efe56d15e9c315c668a5913e17ade420cf7fb5ff7fa62b3c9b0e1574eab4    112233445566.dkr.ecr.eu-central-1.amazonaws.com/amazon/aws-network-policy-agent:v1.0.7-eksbuild.1    io.containerd.runc.v2
3c85fa3829f59f517db1c766e490a014357a760ce12e2859004cdfb8ea3d7cc6    112233445566.dkr.ecr.eu-central-1.amazonaws.com/eks/pause:3.1-eksbuild.1                             io.containerd.runc.v2
420b62c7b2bdf2e7aa10baf8e4afd1ebda0cfff66300a23846758a029ad31222    112233445566.dkr.ecr.eu-central-1.amazonaws.com/eks/pause:3.1-eksbuild.1                             io.containerd.runc.v2
4e21008f8fc70580906990fb95bed91f9155495270fbac1efb043f81e62a1c51    112233445566.dkr.ecr.eu-central-1.amazonaws.com/eks/coredns:v1.11.1-eksbuild.4                       io.containerd.runc.v2
4f9d20de851414160a5003eb6988f2b0df81dfe3d72d4ba3705db01a4571b515    112233445566.dkr.ecr.eu-central-1.amazonaws.com/eks/kube-proxy:v1.29.0-minimal-eksbuild.1            io.containerd.runc.v2
5f8924422d852d09ad44f5d8579d9abaa78304d303007d566300db8f61978ee5    112233445566.dkr.ecr.eu-central-1.amazonaws.com/amazon-k8s-cni:v1.16.0-eksbuild.1                    io.containerd.runc.v2
7f5afdfbb9a8599c3c5888664f0df349aab8740be21d87e629ff7390e0524c2a    112233445566.dkr.ecr.eu-central-1.amazonaws.com/eks/pause:3.1-eksbuild.1                             io.containerd.runc.v2
80ed8ba3fd4624770eb17087c1a046c90be28e9fb2e31630c82e67b4c0ae19dd    112233445566.dkr.ecr.eu-central-1.amazonaws.com/eks/pause:3.1-eksbuild.1                             io.containerd.runc.v2
a0929884ccdd72de6bf848a037e80b206a4fb4e2f9b77be568bac8f51787cccb    112233445566.dkr.ecr.eu-central-1.amazonaws.com/eks/pause:3.1-eksbuild.1                             io.containerd.runc.v2
a0dd24ecb0aee4eb645d25c75a3eade0c8c35fb09127db5ff7d8136d7bb86efe    112233445566.dkr.ecr.eu-central-1.amazonaws.com/eks/coredns:v1.11.1-eksbuild.4                       io.containerd.runc.v2
c3a3a9214b11b808a88c0293312d8877497f06f87435af3a7334717a13588c26    112233445566.dkr.ecr.eu-central-1.amazonaws.com/amazon-k8s-cni-init:v1.16.0-eksbuild.1               io.containerd.runc.v2
```

Now we're talking! We have all the usual suspects here - coredns, kube-proxy, the omnipresent [pause](https://docs.mirantis.com/mke/3.4/ref-arch/pause-containers.html) containers. But right now we're interested in the container based on the `docker.io/otomato/busyhttp:latest` image.

Let's look for its cgroup definition in the cgroup filesystem we discovered previously. First we need to filter out the container id. `ctr` supports filters for its listing function. So the way to parse out the container id by image name is the following:

```
export CONTAINER_ID=$(ctr -n k8s.io containers ls -q image==docker.io/otomato/busyhttp:latest)
```

Note the `-q` that tells `ctr` to only output the id.

Now we can find the container's cgroup config:
```bash
find /sys/fs/cgroup/ -name *$CONTAINER_ID*
/sys/fs/cgroup/kubepods.slice/kubepods-pod5be5d94a_cbfe_416f_9010_6338003af666.slice/cri-containerd-154d9b7b3a83e4db6e3e4ac4ac1f836321337c604c3b590b5188b7a0773bdae1.scope
```
This gives us a long path somewhere inside a folder called `kubepods.slice`. Let's wrap this path in an environment variable and look around:

```
export MY_CGROUP_DIR=$(find /sys/fs/cgroup/ -name *$CONTAINER_ID*)
ls ${MY_CGROUP_DIR}
```

Whew! That's a lot of files! Now according to [this page on Memory QoS](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/) - our `requests.memory` should be translated to `memory.min` while `memory.high` is calculated the following way:

```
memory,high = (pod.spec.containers[i].resources.limits[memory] or nodeAllocatableMemory) * throttlingFactor
```

Let's look at the limit first: 
```
cat ${MY_CGROUP_DIR}/memory.high
max
```
Hmm. That's not a number. But we can also notice that there's a file called `memory.max`. Let's look inside that:
```
cat ${MY_CGROUP_DIR}/memory.high
209715200
```
Ok, here's our limit! 209715200 bytes is exactlymthe 200Mi we defined in the `resources` section of our pod spec.

Now what about the requests? Let's look at `memory.min`:

```
cat ${MY_CGROUP_DIR}/memory.min
0
```

0 is not the request we've defined. And that makes sense. [Memory QoS](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/) has been in alpha since Kubernetes 1.22 (August 2021) and according to the [KEP data](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/2570-memory-qos/kep.yaml) was still in alpha as of 1.27.

In order to see the actual request values for memory reflected in cgroup config one needs to enable the Memory QoS [feature gate in kubelet config](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/#kubelet-config-k8s-io-v1beta1-KubeletConfiguration:~:text=effect.%20Default%3A%2015-,featureGates,-map%5Bstring%5Dbool) as defined [here](https://github.com/kubernetes/kubernetes/blob/master/pkg/features/kube_features.go#L492):

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
featureGates:
  MamoryQoS: true
```

Trouble is - due to the atomic nature of Bottlerocket OS - we can't change its KubeletConfiguration file (found at /etc/kubernetes/kubelet/config) directly. We can only pass settings through `settings.kubernetes` via the API or a config file. But these currently don't support setting feature gates. So it looks like the only way to modify the Kubelet to support Memory QoS on EKS Bottlerocket nodes is to build our own Bottlerocket images. Which is a subject for a whole another blog post.

And for now - let's shrug our shoulders, scratch our heads and bring down our EKS cluster:

```
eksctl delete cluster -f cluster.yaml
```

## Summing it All Up

- `cgroup v2` is enabled by default in current Bottlerocket EKS instances. 

- this allows a better organized resource management on the nodes

- an important Kubernetes feature based on `cgroup v2` is Memory QoS that ensure that memory requests are actually allocated by the container runtime and not merely checked for by the Kubernetes scheduler

- MemoryQoS is still in `alpha` after 2 years

- There's no easy way to enable Memory QoS on Bottlerocket nodes without building the AMIs ourselves.

Anyway - this was an interesting exploration. And if there's anything I got wrong or didn't make clear - please let me know in comments.

May all your containers run smoothly!

The config files used in the blog post can be found [in this github repo](https://github.com/antweiss/botllerocket-cgroupv2)