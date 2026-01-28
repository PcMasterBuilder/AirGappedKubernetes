# Air Gapped Kubernetes (K8s) Installation
#### Installation guide for a RHEL 8.5 environment with no access to the internet
- At least two machines (one master and one worker)
- Each has its own ip and can communicate with each other (test by SSHing to eachother)

## Pre-install setup (do these on all machines):

> All files that need to be downloaded should be available in this repository at https://github.com/PcMasterBuilder/AirGappedKubernetes/blob/main/AirgappedInstallation/files - as of January 2026

### User setup
```
sudo hostnamectl set-hostname username
su - 
visudo
```
Add the following line to the end of the file:

`username ALL=(ALL) NOPASSWD: ALL`

### Shared Folder
>VirtualBox window → Devices → Insert Guest Additions CD Image

Run:
```
sudo mkdir -p /mnt/cdrom
sudo mount /dev/cdrom /mnt/cdrom
sudo sh /mnt/cdrom/VBoxLinuxAdditions.run
sudo modprobe vboxsf
lsmod | grep vboxsf
sudo mkdir -p /Shared
sudo mount -t vboxsf SharedOnline /Shared
reboot
```

### Disable firewalld service
Run:

`sudo systemctl disable firewalld`

### Disable swap

Run:

```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Kernel modules

```
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Sysctl

```
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sudo sysctl --system
```

## Installation

### RPMs
Run:

```
cd ~/k8s-rpms
sudo dnf install -y *.rpm --disablerepo='*'
```

If DNF complains, try:

```
sudo rpm -Uvh *.rpm
```

Verify:

```
containerd --version
kubeadm version
kubelet --version
kubectl version --client
```

### Containerd

Run:
```
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Then:
```
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml
```

Start it:
`sudo systemctl enable --now containerd`

Finally, Check:
`systemctl status containerd --no-pager`

### Import Images
Run:
```
sudo ctr -n k8s.io images import ~/k8s-core-images.tar
sudo ctr -n k8s.io images import ~/calico-images.tar
```

Verify:
```
sudo ctr -n k8s.io images ls
```

You should see:
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- kube-proxy
- etcd
- pause
- CoreDNS
- calico/*

## Init command for master

Run:
```
sudo kubeadm init \
  --apiserver-advertise-address=<HOST_IP> \
  --pod-network-cidr=192.168.0.0/16 \
  --image-repository=registry.k8s.io
```

## Configuration for kubectl
Run:
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Test by running `kubectl get node`

## Add Calico or Flannel
> I added Calico, for Flannel add the files and yaml seperatly

Run:
`kubectl apply -f calico.yaml`

Then check it by running `kubectl get pods -n kube-system`
