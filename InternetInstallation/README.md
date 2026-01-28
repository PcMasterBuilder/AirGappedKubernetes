# Air Gapped Kubernetes (K8s) Installation
#### I'm assuming an ideal environment of latest linux, internet connection
 -  At least two machines (one master and one worker)
 - Each has their own ip and can communicate with each other (test by SSHing to eachother)

## Pre-install setup (do these on all machines):

> Sorry in advance, there are lots of dependencies and installations before we actually get K8s up and running
>
>Also, I kinda just put sudo in front of everything that gave me permission denied (essentialy everything)

> All files that need to be downloaded should be available in this repository at https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files - as of January 2026

`sudo visudo`

Scroll down to `Defaults    secure_path = ` and add `:/usr/local/bin` at the end of the line

Create a kubelet-config.yaml, doesn’t matter where we place it, because we're passing the file in –config arg later

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta4
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
apiVersion: kubeadm.k8s.io/v1beta4
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
```

#### Open ports needed:

On master node:

```sudo firewall-cmd --permanent --add-port={6443/tcp,2379-2380/tcp,10250/tcp,10259/tcp,10257/tcp} && sudo firewall-cmd --reload```

On worker node(s):

```sudo firewall-cmd --permanent --add-port={10250/tcp,10256/tcp,30000-32767/tcp,30000-32767/udp} && sudo firewall-cmd --reload```

## Installing containerd (following [docs](https://github.com/containerd/containerd/blob/main/docs/getting-started.md))

> **January 2026**
>
> You can download the following file from [here](https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files/containerd-2.2.1-linux-amd64.tar.gz)

Download the `containerd-<VERSION>-<OS>-<ARCH>.tar.gz` archive from https://github.com/containerd/containerd/releases ,
verify its sha256sum, and extract it under `/usr/local`:

```console
sudo tar Cxzvf /usr/local containerd-2.2.1-linux-amd64.tar.gz
```

### systemd for containerd

> **January 2026**
>
> You can download the following file from [here](https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files/containerd.service)

Download [containerd.service](https://raw.githubusercontent.com/containerd/containerd/main/containerd.service) and put it in `/usr/local/lib/systemd/system/containerd.service`

```console
sudo mkdir -p /usr/local/lib/systemd/system/
sudo cp containerd.service /usr/local/lib/systemd/system/containerd.service
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```
### Generate containerd config.toml

Generate the default configuration via:
```
sudo mkdir -p /etc/containerd
sudo chmod 777 /etc/containerd
containerd config default > /etc/containerd/config.toml
```

Edit config with `sudo nano /etc/containerd/config.toml`

Look for `[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]` (or use ctrl+f)
 and change `SystemdCgroup = false` to `true` 

### Installing runc

> **January 2026**
>
> You can download the following file from [here](https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files/runc.amd64)

Download the `runc.<ARCH>` binary from https://github.com/opencontainers/runc/releases , and install it as `/usr/local/sbin/runc`.

```console
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Installing CNI plugins

> **January 2026**
>
> You can download the following file from [here](https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files/cni-plugins-linux-amd64-v1.9.0.tgz)

Download the `cni-plugins-<OS>-<ARCH>-<VERSION>.tgz` archive from https://github.com/containernetworking/plugins/releases , and extract it under `/opt/cni/bin`:

```console
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.9.0.tgz
```


## K8s Components (following [docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl))

Install CNI plugins (required for most pod network):
```console
CNI_PLUGINS_VERSION="v1.9.0"
ARCH="amd64"
DEST="/opt/cni/bin"
sudo mkdir -p "$DEST"
curl -L "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGINS_VERSION}/cni-plugins-linux-${ARCH}-${CNI_PLUGINS_VERSION}.tgz" | sudo tar -C "$DEST" -xz
```
Define the directory to download command files:
```console
DOWNLOAD_DIR="/usr/local/bin"
sudo mkdir -p "$DOWNLOAD_DIR"
```
Optionally install crictl (required for interaction with the Container Runtime Interface (CRI), optional for kubeadm):
```console
CRICTL_VERSION="v1.35.0"
ARCH="amd64"
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz" | sudo tar -C $DOWNLOAD_DIR -xz
```
Install kubeadm, kubelet and add a kubelet systemd service:
```console
RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"
ARCH="amd64"
cd $DOWNLOAD_DIR
sudo curl -L --remote-name-all https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet}
sudo chmod +x {kubeadm,kubelet}

RELEASE_VERSION="v0.18.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubelet/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /usr/lib/systemd/system/kubelet.service
sudo mkdir -p /usr/lib/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```
Enable the kubelet service before running kubeadm:
```console
sudo systemctl enable --now kubelet
```

### Installing kubectl

Download the binary
```console
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```
Install
```console
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
Test
```console
kubectl version --client
```
### Running kubeadm!!

#### Here we go bear with me

Finally we can run, passing in the file you created in the first step
```console
sudo kubeadm init --config /path/to/your/kubelet-config.yaml
```
And if you did everything correctly until now it should run smoothly 
(haha jk no way it works for you first try)
