# Initial Setup (all Pi's)

### Required packages

Note: `neovim` is not _"required"_ for this setup, however all files edited in these setup steps use `neovim` (`nvim`). Replace with your preferred editor of choice.

``` bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl neovim htop
```

### Setup Static IP

Notes:

- For the `worker nodes` this is likely not required. For the `control plane node` this is highly suggested unless you have a local DNS server.

- The Raspberry Pi os seems to use `dhcpcd.conf` as _a_ way to configure static IP addresses. This is not the only way to configure _static IPs_ on linux boxes - this was just the way I solved the problem.

```bash
sudo nvim /etc/dhcpcd.conf

# -- add:

# static IP for primary k8s node (lower metric value == higher priority):
interface eth0
ipv4only
metric 100
static ip_address=192.168.1.86/24 # <--- needs to be unique per node
static routers=192.168.1.1
static domain_name_servers=1.1.1.1, 192.168.1.1

interface wlan0
ipv4only
metric 200
static ip_address=192.168.1.186/24 # <--- needs to be unique per node
static routers=192.168.1.1
static domain_name_servers=1.1.1.1, 192.168.1.1
```

### Disable Swap

Customize sysctl:

```bash
# -- edit:
sudo nvim /etc/sysctl.conf

# -- add (or update) to end:

# disable swap
vm.swappiness=0
```

```bash
sudo dphys-swapfile swapoff && \
sudo dphys-swapfile uninstall && \
sudo systemctl disable dphys-swapfile && \
sudo update-rc.d dphys-swapfile remove && \
sudo swapoff -a
```

### Configure ContainerD CRI

Note: `k8s` supports multiple `container runtimes` (called CRI). For this setup `containerd` is used instead of docker.

```bash
sudo apt install -y containerd.io
```

Setup default config:

```bash
sudo mkdir -p /etc/containerd

# -- switch to root
sudo bash
rm /etc/containerd/config.toml
containerd config default > /etc/containerd/config.toml
exit
```

Edit config to support `cri` (!! IMPORTANT !!)

```ini
sudo nvim /etc/containerd/config.toml

# -- Make sure `cri` is not listed as a disabled plugin
disabled_plugins = []

# -- Make sure `SystemCgroup` is set to `true`
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

Configure additional `containerd` features:

```bash
sudo nvim /etc/modules-load.d/containerd.conf

# -- add the following (and save the file)
overlay
br_netfilter

# -- reload modules and containerd
sudo modprobe overlay
sudo modprobe br_netfilter
sudo systemctl restart containerd
sudo systemctl status containerd
```

### Configure network routing/bridging

```bash
sudo nvim /etc/sysctl.d/99-kubernetes-cri.conf

# -- add the following (and save the file)
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1

# -- apply sysctl params without reboot
sudo sysctl --system
```

### Configure boot options

```
sudo nvim /boot/cmdline.txt

# -- add these:
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1
```

### REBOOT

(seriously - _reboot_)

```bash
sudo reboot
```
