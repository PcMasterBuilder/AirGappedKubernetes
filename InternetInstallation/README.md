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

Until now you should've been running all the commands on all your nodes. Kubectl specifically only needs to be installed where you want to control the cluster from. This can be on all your nodes or just your master.

Download the binary (make sure you're in your download directory)
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

On the master node:
```console
sudo kubeadm init --config /path/to/your/kubelet-config.yaml
```
And if you did everything correctly until now 
(haha jk no way it works for you first try) it should return `Your Kubernetes control-plane has initialized successfully!` along with some commands.

First of all make sure to run what it told you:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Then take note of the join command it gave you. It should be something like 
```console
kubeadm join ip.ip.ip.ip:port --token tokentokentoken \
        --discovery-token-ca-cert-hash sha256:superlongstringofcharacters
```
Add `sudo` and run on each of your worker nodes

If all went well, you can run `kubectl get nodes` and see your nodes. Currently their state will be `NotReady`, don't worry! This is ok. We need to install a CNI for them to be ready. We will use flannel
### Installing Flannel (following [this](https://medium.com/@chandnamanan2004/part-3-initializing-the-control-plane-node-for-your-kubernetes-cluster-3fd3a04f8858#:~:text=Step%203%3A%20Install%20a%20CNI%20(Container%20Network%20Interface)))

From now on we'll only need to run the commands on our master (or where we installed kubectl)
Let's download the kube-flannel.yaml to our downloads folder 
```console
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
Run `ip a` and take note of your main interface name (mine is enp0s3 but it might be eth0 or something else)

`nano kube-flannel.yml` to edit the CIDR range for flannel

Make sure `"Network"` value is 10.200.0.0/16 (or change if needed)
```yaml
 net-conf.json: |
    {
      "Network": "10.200.0.0/16",
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }
```
Look for this part-
```yaml
 containers:
      - args:
        - --ip-masq
        - --kube-subnet-mgr
```
and add
```yaml
        - --iface=your-network-interface
```
Once done editing, apply the Flannel CNI YAML:
```console
kubectl apply -f kube-flannel.yml
```
And check if Flannel is running:
```console
kubectl get pods -A
```
And if you wait long enough (about a minute) all of your containers should come up
```console
$ kubectl get pods -A
NAMESPACE      NAME                                READY   STATUS    RESTARTS        AGE
kube-flannel   kube-flannel-ds-8mpkc               1/1     Running   0               91s
kube-flannel   kube-flannel-ds-csbcr               1/1     Running   0               91s
kube-flannel   kube-flannel-ds-t7pz9               1/1     Running   0               91s
kube-system    coredns-7d764666f9-hh2tz            1/1     Running   0               38m
kube-system    coredns-7d764666f9-rbbjz            1/1     Running   0               38m
kube-system    etcd-firstrhel                      1/1     Running   0               38m
kube-system    kube-apiserver-firstrhel            1/1     Running   0               38m
kube-system    kube-controller-manager-firstrhel   1/1     Running   0               38m
kube-system    kube-proxy-jp25n                    1/1     Running   0               28m
kube-system    kube-proxy-rsfpx                    1/1     Running   0               38m
kube-system    kube-proxy-vvkzb                    1/1     Running   0               21m
kube-system    kube-scheduler-firstrhel            1/1     Running   1 (4m41s ago)   38m
```
Which also means that if you run `kubectl get nodes` your nodes should all be ready!
```console
$ kubectl get nodes
NAME         STATUS   ROLES           AGE   VERSION
firstrhel    Ready    control-plane   38m   v1.35.0
secondrhel   Ready    <none>          28m   v1.35.0
thirdrhel    Ready    <none>          21m   v1.35.0
```
