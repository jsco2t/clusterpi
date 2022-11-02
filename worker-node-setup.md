# K8s worker-node setup

Compared to the control-plane setup, setting up a worker node is relatively short. 

### Install components

```bash
sudo apt update && sudo apt install -y kubectl=1.24.6-00 kubeadm=1.24.6-00

sudo apt-mark hold kubeadm kubectl
```

### Bootstrap the node to the cluster

From the `control-plane` setup there was a note to copy the command for bootstrapping a node. That's the command you want to run now.

The command will look *something* like the following:

```bash
kubeadm join 192.168.1.86:6443 \
--token 13te2e.zz8fxyqhtqwk8mo3 \
--discovery-token-ca-cert-hash sha256:febe6524aa16c68e3f5aa19d8a0ec8d1ee37a74e0c6672c6bd0948f8b8fbac19
```

### Verifying the node has joined the cluster

You will want to SSH back into the `control-plane` node for the cluster. Run the following command a few times - it will take a couple minutes but the worker node should join the cluster and become available:

```bash
kubectl get nodes
```