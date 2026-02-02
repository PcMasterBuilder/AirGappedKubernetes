# Air Gapped Kubernetes (K8s) Installation
#### I will be doing this all on RHEL 8.5
 -  At least two machines (one master and one worker)
 - Each has their own ip and can communicate with each other (test by SSHing to eachother)

## Pre-install setup (do these on all machines):

> Sorry in advance, there are lots of dependencies and installations before we actually get K8s up and running
>
>Also, I kinda just put sudo in front of everything that gave me permission denied (essentialy everything)

> All files that need to be downloaded should be available in this repository at https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/InternetInstallation/files


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
sudo vi /etc/modules-load.d/k8s.conf
# Add these lines and save changes
overlay
br_netfilter

# Then run
sudo modprobe br_netfilter
sudo modprobe overlay


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

```sudo firewall-cmd --permanent --add-port={6443/tcp,2379-2380/tcp,10250-10252/tcp,10259/tcp,10257/tcp,7946/tcp,7946/udp,9443/tcp,8472/udp,30000-32767/tcp} && sudo firewall-cmd --reload```

On worker node(s):

```sudo firewall-cmd --permanent --add-port={10250/tcp,10256/tcp,30000-32767/tcp,30000-32767/udp,7946/tcp,7946/udp,8472/udp} && sudo firewall-cmd --reload```



## Installing containerd 
Bring the tgz from [here](files/containerd-1.7.18-linux-amd64.tar.gz) and install with 

```console
sudo tar Cxzvf /usr/local containerd-1.7.18-linux-amd64.tar.gz
```

### systemd for containerd
Download [containerd.service](files/containerd.service) and put it in `/usr/local/lib/systemd/system/containerd.service`:

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

Look for `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]` (or use ctrl+w)
 and change `SystemdCgroup = false` to `true` 

### Installing runc
Download the [runc.amd64](files/runc.amd64), and install it as `/usr/local/sbin/runc`:


```console
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Installing CNI plugins

Download [cni-plugins-linux-amd64-v1.3.0.tgz](files/cni-plugins-linux-amd64-v1.3.0.tgz), and extract it under `/opt/cni/bin`:

```console
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz
```
## K8s Components (following [docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl))

> For now ignoring crictl because I don't think we need it
> Download [crictl-v1.28.0-linux-amd64.tar.gz](files/crictl-v1.28.0-linux-amd64.tar.gz)

#### Install kubeadm, kubelet and add a kubelet systemd service:
Download [kubeadm](files/kubeadm) and [kubelet](files/kubelet), perform:
```console
sudo chmod +x kubeadm kubelet
sudo cp kubeadm kubelet /usr/local/bin
```

Download [10-kubeadm.conf](files/10-kubeadm.conf) and [kubelet.service](files/kubelet.service).

Make sure you're in the directory where you placed the files, and run:

```console
sudo cp ./kubelet /usr/bin/kubelet
sudo chmod +x /usr/bin/kubelet
sudo cp kubelet.service /usr/lib/systemd/system/kubelet.service
sudo sed -i 's|/usr/bin|/usr/bin|g' /usr/lib/systemd/system/kubelet.service
sudo mkdir -p /usr/lib/systemd/system/kubelet.service.d
sudo cp 10-kubeadm.conf /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
sudo sed -i 's|/usr/bin|/usr/bin|g' /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
sudo systemctl enable --now kubelet
```

### Installing kubectl
Download [kubectl binary](files/kubectl) and install:

```console
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
Test
```console
kubectl version --client
```

### Running kubeadm!!

Here is the stage where we see if all this guesswork was for naught

On the master node:
```console
sudo kubeadm init --config /path/to/your/kubelet-config.yaml
```