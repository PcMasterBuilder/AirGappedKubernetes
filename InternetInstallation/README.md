# Air Gapped Kubernetes (K8s) Installation
#### I'm assuming an ideal environment of latest linux, internet connection
 -  At least two machines (one master and one worker)
 - Each has their own ip and can communicate with each other (test by SSHing to eachother)

## Pre-install setup (do these on all machines):

`sudo visudo`

Scroll down to `Defaults    secure_path = ` and add `:/usr/local/bin` at the end of the line

Create a kubelet-config.yaml, doesn’t matter where we place it, because we're passing the file in –config arg later

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "Your-Master-IP"  # Insert your Master Node IP
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
failSwapOn: false
memorySwap:
  swapBehavior: NoSwap
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: stable
networking:
  podSubnet: "10.200.0.0/16"
```

### A few networking stuff and we'll get started:
Enabling `br_netfilter` module, configuring bridged network traffic and enabling IPv4 forwarding
```console
sudo modprobe br_netfilter
echo 'br_netfilter' | sudo tee /etc/modules-load.d/k8s.conf

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
ls /proc/sys/net/bridge/bridge-nf-call-iptables  # Verify (no error)

sudo systemctl restart kubelet containerd
```

#### Open ports needed:

On master node:

`sudo firewall-cmd --permanent --add-port={6443/tcp,2379-2380/tcp,10250/tcp,10259/tcp,10257/tcp} && sudo firewall-cmd --reload`

On worker node(s):

`sudo firewall-cmd --permanent --add-port={10250/tcp,10256/tcp,30000-32767/tcp,30000-32767/udp} && sudo firewall-cmd --reload`

## Installing containerd (following [docs](https://github.com/containerd/containerd/blob/main/docs/getting-started.md))

> **January 2026**
> You can download the following files from 
Download the `containerd-<VERSION>-<OS>-<ARCH>.tar.gz` archive from https://github.com/containerd/containerd/releases ,
verify its sha256sum, and extract it under `/usr/local`:

```console
tar Cxzvf /usr/local containerd-1.6.2-linux-amd64.tar.gz
```

### containerd via systemd

Download [containerd.service](https://raw.githubusercontent.com/containerd/containerd/main/containerd.service) and put it in `/usr/local/lib/systemd/system/containerd.service`

```console
cp containerd.service /usr/local/lib/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable --now containerd
```

### Installing runc

