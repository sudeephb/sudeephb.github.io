---
title: "Building a K8s cluster with kubeadm"
categories:
  - Kubernetes
tags:
  - kubeadm
---

In this article, let's create a basic Kubernetes cluster using `kubeadm`. `kubeadm` is a tool that simplifies the process of creating a k8s cluster.

I have set up three Ubuntu 20.04 VMs on the same virtual network with hostnames `k8s-control`, `k8s-worker1` and `k8s-worker2`. The first one will be the control plane node and the rest two will be worker nodes. 

Before we set up our cluster, there are a few packages and configurations we need to set up. First of all, let's install a `container runtime` which enables us to run pods in each of the nodes. There are several container runtimes supported by Kubernetes, here we will go with `containerd`.

Before installing `containerd`, there are a few prerequisites we need to install and configure, let's get them done in each of the Ubuntu servers:

```shell
# Updating kernel conf to load some kernel modules(overlay and netfilter) on startup
cat << EOF | sudo tee /etc/modules-load.d/containerd.conf
> overlay
> br_netfilter
> EOF

# Enabling these modules in the current session as well
sudo modprobe overlay
sudo modprobe br_netfilter

# Add some network configuration for k8s to work properly:
cat << EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
> net.bridge.bridge-nf-call-iptables = 1
> net.ipv4.ip_forward = 1
> net.bridge.bridge-nf-call-ip6tables = 1
> EOF

# These configurations will be loaded on startup. Applying them immediately as well
sudo sysctl --system
```

Now, we are ready to install the container runtime `containerd` on each of the machines:
```
sudo apt update && sudo apt install -y containerd
```

Now, let's create a default containerd configuration and save it to a file `etc/containerd/config.toml`. To make sure containerd starts using that configuration immediately, we'll also restart it:
```shell
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

sudo systemctl restart containerd
```

Kubernetes requires swap to be disabled. So, let's disable it on each of the machines:
```shell
sudo swapoff -a
```

We will be installing Kubernetes packages from the custom `apt` repository provided by Kubernetes. Let's first install the packages needed to use the repository:

```shell
sudo apt update && sudo apt install -y apt-transport-https curl
```

Now, let's download the Google Cloud public signing key and then add the Kubernetes `apt` repository as suggested in the [Kubernetes docs][install-kubeadm-docs]. We will also update our local package listings.

```shell
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
```

Now, let's install the 1.24 version of Kubernetes packages:

```
sudo apt install -y kubelet=1.24.0-00 kubeadm=1.24.0-00 kubectl=1.24.0-00
```

While we're at it, let's make sure that these packages aren't updated automatically:

```
sudo apt-mark hold kubelet kubeadm kubectl
```

Finally, we are ready to initialize our Kubernetes cluster. Let's create a cluster by explicitly specifying the k8s version and also the network cidr available for the pods. We will do that by running the following command on the `k8s-control` server. This might take a couple minutes to run.
```shell
sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.24.0
```

Now, on the same machine `k8s-control`, let's configure the `kubeconfig` file. It is the file that allows us to authenticate and communicate with our cluster.
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now, we can use `kubectl` to verify that our cluster is up and running:
```shell
kubectl get nodes
```

The output of the command is as:
```shell
NAME          STATUS     ROLES           AGE     VERSION
k8s-control   NotReady   control-plane   6m41s   v1.24.0
```

Here, we see a single node `k8s-control`. This is expected because we haven't configured the worker nodes to be part of the cluster yet. However, the `Status` is `NotReady`. This is because set up a [networking plugin][network-plugins] yet. There are multiple choices for a networking plugin, here we will go with [Calico][about-calico]. Installing `Calico` is very simple. We will just use the manifest file provided by calico:
```shell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

After a few moments, the output of `kubectl get nodes` will show the `Status` of `k8s-control` control-plane node as `Ready`. 
```shell
NAME          STATUS   ROLES           AGE     VERSION
k8s-control   Ready    control-plane   7m22s   v1.24.0
```

Now, we are ready to add the worker nodes(`k8s-worker1` and `k8s-worker2`) to our the cluster. We will do it using the `kubeadm join` command. First, let's create a token on `k8s-control`:
```shell
kubeadm token create --print-join-command
```

The output is as:
```
kubeadm join 10.0.1.101:6443 --token nlq1pb.12mzri195rwupusj --discovery-token-ca-cert-hash sha256:b348bd389511386883ac392b3a92cddf6dced0fe4ef6366cd29ea09138868f0a
```

Now, we run this command on both of the worker machines to join them to the cluster. After waiting a few moments, we can see that the worker nodes and up and running and in `Ready` state along with the control plane node.
```
kubectl get nodes
NAME          STATUS   ROLES           AGE   VERSION
k8s-control   Ready    control-plane   12m   v1.24.0
k8s-worker1   Ready    <none>          95s   v1.24.0
k8s-worker2   Ready    <none>          47s   v1.24.0
```

We now have a running kubernetes cluster where we can add container workloads.

[install-kubeadm-docs]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
[network-plugins]:   https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/
[about-calico]: [https://projectcalico.docs.tigera.io/about/about-calico]
