# Air Gapped Kubernetes (K8s) Installation
#### I will be doing this all on RHEL 8.5
 -  At least two machines (one master and one worker)
 - Each has their own ip and can communicate with each other (test by SSHing to eachother)
 -  A docker registry (sryy they're not too crazy to set up)

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
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.56.101"  # Master Node IP
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
kubernetesVersion: v1.30.14
imageRepository: "192.168.56.103:5000"  # Registry IP
dns:
  imageRepository: "192.168.56.103:5000/coredns"
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


Download [crictl-v1.28.0-linux-amd64.tar.gz](files/crictl-v1.28.0-linux-amd64.tar.gz)

```console
sudo tar -C /usr/local/bin -xzf crictl-v1.28.0-linux-amd64.tar.gz
```

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

sudo nano /etc/containerd/config.toml

change <IP> & <Port>

```console
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."<IP>:<Port>"]
      endpoint = ["http://<IP>:<Port>"]
  [plugins."io.containerd.grpc.v1.cri".registry.configs]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."<IP>:<Port>".tls]
      insecure_skip_verify = true
```
Also look for `sandbox_image =` and change to your registry. (and make sure the tag version at the end is 3.9, not 3.8)

#### Crucial after changes:

Run `sudo systemctl restart containerd`

[source](https://dnelaturi.medium.com/how-to-configure-private-registry-for-containerd-in-k8s-environment-3b55a2caf79b)

Download tars in [folder](files/AirgapImages)

For each run
```console
docker load -i TAR
docker tag NAME YOUR-REGISTRY:5000/NAME-WITHOUT-PREFIX
docker push NEW-NAME-WITH-REGISTRY
```
> sudo ctr -n k8s.io images pull 192.168.56.103:5000/kube-apiserver:v1.30.14 --plain-http

> sudo ctr -n k8s.io images pull 192.168.56.103:5000/coredns:v1.11.3 --plain-http

> sudo ctr -n k8s.io images pull 192.168.56.103:5000/coredns/coredns:v1.11.3 --plain-http

> sudo ctr -n k8s.io images pull 192.168.56.103:5000/etcd:3.5.15-0 --plain-http

> sudo ctr -n k8s.io images pull 192.168.56.103:5000/kube-controller-manager:v1.30.14 --plain-http

> sudo ctr -n k8s.io images pull 192.168.56.103:5000/kube-proxy:v1.30.14 --plain-http

> sudo ctr -n k8s.io images pull 192.168.56.103:5000/kube-scheduler:v1.30.14 --plain-http

> sudo ctr -n k8s.io images pull 192.168.56.103:5000/pause:3.9 --plain-http


You may need to install some dependencies from [here](files/conntrack-offline) 

Go into that folder once downloaded and 
```console
sudo rpm -Uvh ./*.rpm
 ```

Ok I'll need to go over this and make it more legible, but run

```console 
sudo kubeadm init --config /path/to/your/kubelet-config.yaml
```
and then run the commands it gives you (if it works first try)
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Then make sure to save the join command it gives you

Add `sudo` and run on each of your worker nodes

If all went well, you can run `kubectl get nodes` and see your nodes. Currently their state will be `NotReady`, don't worry! This is ok. We need to install a CNI for them to be ready. We will use flannel

### Installing Flannel 

Let's download the [kube-flannel.yml](files/kube-flannel.yml) to our downloads folder 

Also download the two flannel images we need to run this yml: [flannel.tar](files/flannel.tar), [flannel-cni.tar](files/flannel-cni.tar)

Load them into the registry with
```console
docker load -i flannel.tar
docker load -i flannel-cni.tar
docker tag flannel/flannel:v0.24.3 
docker tag flannel/flannel-cni-plugin:v1.4.0-flannel1 192.168.56.103:5000/flannel-cni-plugin:v1.4.0-flannel1
REGISTRYIP:5000/flannel:v0.24.3
docker push REGISTRYIP:5000/flannel:v0.24.3
docker push REGISTRYIP:5000/flannel-cni-plugin:v1.4.0-flannel1
```

#### Now we're gonna make some tweaks to your kube-flannel.yml
First, go through the kube-flannel.yml you downloaded, search for `image`, and replace every instance with your updated image with your registry prefix.

> E.G. `image: docker.io/flannel/flannel:v0.24.3` now becomes `image: 192.168.56.103:5000/flannel:v0.24.3`

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


[Flannel error when first applying yaml, if there's no ip route defined](https://gemini.google.com/share/755464fd7d81) - bandage solution but still
```
sudo ip route add 10.96.0.0/12 dev enp0s3
```

Check if Flannel is running:
```console
kubectl get pods -A
```
And if you wait long enough (about a minute) all of your containers should come up

<details>
<summary>If you run `kubectl get nodes -o wide and there's no Internal IP</summary>

 - On Rhel1 (Master): Edit the Kubelet config file (usually `/var/lib/kubelet/kubeadm-flags.env` or /etc/sysconfig/kubelet):
 ```
 sudo vi /var/lib/kubelet/kubeadm-flags.env
 ```
 Add --node-ip=192.168.56.101 to the KUBELET_KUBEADM_ARGS string. It should look something like: KUBELET_KUBEADM_ARGS="--container-runtime-endpoint=unix:///run/containerd/containerd.sock --node-ip=192.168.56.101"

 - On Rhel2 (Worker): Do the same, but use its IP:
 ```
 sudo vi /var/lib/kubelet/kubeadm-flags.env
 ```
 Add --node-ip=192.168.56.102 to the args.\

 - Restart Kubelet on both nodes:
 ```
 sudo systemctl restart kubelet
 ```

</details>

(`kubectl get all -A` or `kubectl get nodes -o wide`)

Congradulations! You now have a "fully working" cluster!
However, if you want to access your programs externally you'll need to setup Ingress with a Load Balancer
# Ingress + Load Balancer
### Helm installation

Download [Helm](files/helm-v4.1.0-linux-amd64.tar.gz)

Unpack it
```console
tar -zxvf helm-v4.1.0-linux-amd64.tar.gz
sudo cp linux-amd64/helm /usr/local/bin/helm
sudo chmod +x /usr/local/bin/helm
```

### MetalLB (following [docs](https://metallb.io/installation/))

First, let's enable strict ARP mode. Edit the kube-proxy config with `kubectl edit configmap -n kube-system kube-proxy`, scroll down to `mode: ""` and set:
```yaml
mode: "ipvs"
```
A bit higher up you'll see `strictARP: false`. Change to true
```yaml
strictARP: true
```

#### Installation

Download [metallb-controller](files/metallb-controller.tar) and [metallb-speaker](files/metallb-speaker.tar)

Load, tag and push:
```console
docker load -i metallb-controller.tar
docker load -i metallb-speaker.tar
docker tag quay.io/metallb/controller:v0.15.3 192.168.56.103:5000/controller:v0.15.3
docker push 192.168.56.103:5000/controller:v0.15.3
docker tag quay.io/metallb/speaker:v0.15.3 192.168.56.103:5000/speaker:v0.15.3
docker push 192.168.56.103:5000/speaker:v0.15.3
```
Edit the metallb yaml