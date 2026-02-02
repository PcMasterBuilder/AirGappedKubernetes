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
Bring the tgz from [here](containerd-1.7.18-linux-amd64.tar.gz) and install with 

```console
sudo tar Cxzvf /usr/local containerd-1.7.18-linux-amd64.tar.gz
```

### systemd for containerd
Download [containerd.service](files/containerd.service) and put it in `/usr/local/lib/systemd/system/containerd.service`

```console
sudo mkdir -p /usr/local/lib/systemd/system/
sudo cp containerd.service /usr/local/lib/systemd/system/containerd.service
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```