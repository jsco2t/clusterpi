# K8s control-plane setup

A few items of information will be needed for this step:

- IP Address for the Control Plane Node

- Version of `k8s` to use (this document uses 1.24)

- The `container networking` (CNI) environment to use (`calico` used below)

- This setup uses `containerd` as the CRI (parameter `--cri-socket`)

- You need to select a `cidr` networking range to use for the `pods` on the `worker nodes`. It's **critical** that this IP address range not conflict with the network that the `control plane` is running on.

### Install components

```bash
sudo apt update && sudo apt install -y kubelet=1.24.6-00 kubectl=1.24.6-00 kubeadm=1.24.6-00

sudo apt-mark hold kubelet kubeadm kubectl
```

### Bootstrap control-plane with kubeadm command

Run `kubeadm` in _dry run mode_ first to see if there are any issues:

```bash
sudo kubeadm init \
  --control-plane-endpoint 192.168.1.86 \
  --kubernetes-version 1.24.6 \
  --pod-network-cidr 10.10.0.0/16 \
  --cri-socket unix:///var/run/containerd/containerd.sock \
  --apiserver-advertise-address 192.168.1.86 \
  --dry-run
```

If everything looks ok, then run the same command without `--dry-run`:

```bash
sudo kubeadm init \
  --control-plane-endpoint 192.168.1.86 \
  --kubernetes-version 1.24.6 \
  --pod-network-cidr 10.10.0.0/16 \
  --cri-socket unix:///var/run/containerd/containerd.sock \
  --apiserver-advertise-address 192.168.1.86
```

**!! IMPORTANT !!**

When the `kubeadm` command completes it will give you two important pieces of information:

1. How to setup `kubectl` so that you can talk to `k8s`.

    This will look something like this:

    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

    Run these commands **NOW**. They will be required for the next steps in this document.

1. The command you will need to use to bootstrap the `worker nodes` in the cluster.

    This will look _similar_ to the following (but the cert/hash details **will be different**):

    ```bash
    kubeadm join 192.168.1.86:6443 \
    --token 12te2e.u58fxyqhtqwk8mo3 \
    --discovery-token-ca-cert-hash sha256:febe6724af16d688345aa99d890ec8d1ee47a75e0c7672c6bd0948f8b8fbac19
    ```

    Save this information as you will need it when you setting up the `worker nodes`.

### Download and configure calico (manifest version - setup)

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/calico.yaml -O
```

Open the `calico.yaml` manifest and setup the pod network cidr:

```
nvim ./calico.yaml

## -- Find near the bottom the pod network cidr (likely 192.168.0.0/16)
## -- Change it to the following:

10.10.0.0/16
```

Apply the calico as the `cni` for `k8s`:

```
kubectl apply -f ./calico.yaml
```

### Check on the control plane pod status

This is an important step to perform before moving forward. You need to let the `control plane` _stabilize_ before setting up the `worker nodes`.

First check to see the node status:

```bash
kubectl get nodes
```

You want to see the `control plane` node report it's ready:

```bash
NAME           STATUS   ROLES           AGE   VERSION
clusterpi-01   Ready    control-plane   10m   v1.24.6
```

The first time you check - it's likely to be `NotReady`. To see how things are going - run the following:

```bash
kubectl -n kube-system get pods
```

You may need to run that command a few times before all of the control plane pods are up and running. After this - when you request the nodes status it should indicate it's `ready`
