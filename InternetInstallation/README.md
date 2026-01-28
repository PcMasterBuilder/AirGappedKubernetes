# Air Gapped Kubernetes (K8s) Installation
#### I'm assuming an ideal environment of latest linux, internet connection
 -  At least two machines (one master and one worker)
 - Each has their own ip and can communicate with each other (test by SSHing to eachother)

## Pre-install setup (do these on all machines):

> Sorry in advance, there are lots of dependencies and installations before we actually get K8s up and running

> All files that need to be downloaded should be available in this repository at https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files - as of January 2026

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
> You can download the following file from [here](https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files/containerd-2.2.1-linux-amd64.tar.gz)

Download the `containerd-<VERSION>-<OS>-<ARCH>.tar.gz` archive from https://github.com/containerd/containerd/releases ,
verify its sha256sum, and extract it under `/usr/local`:

```console
tar Cxzvf /usr/local containerd-2.2.1-linux-amd64.tar.gz
```

### containerd via systemd

> **January 2026**
> You can download the following file from [here](https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files/containerd.service)

Download [containerd.service](https://raw.githubusercontent.com/containerd/containerd/main/containerd.service) and put it in `/usr/local/lib/systemd/system/containerd.service`

```console
cp containerd.service /usr/local/lib/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable --now containerd
```

### Installing runc

> **January 2026**
> You can download the following file from [here](https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files/runc.amd64)

Download the `runc.<ARCH>` binary from https://github.com/opencontainers/runc/releases , and install it as `/usr/local/sbin/runc`.

```console
install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Installing CNI plugins

> **January 2026**
> You can download the following file from [here](https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files/cni-plugins-linux-amd64-v1.9.0.tgz)

Download the `cni-plugins-<OS>-<ARCH>-<VERSION>.tgz` archive from https://github.com/containernetworking/plugins/releases , and extract it under `/opt/cni/bin`:

```console
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.9.0.tgz
```

### Generate containerd config.toml

The default configuration can be generated via `containerd config default > /etc/containerd/config.toml`

