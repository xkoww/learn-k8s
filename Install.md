# Create K8S

## resource
```
2 GB or more of RAM per machine 

2 CPUs or more.
```

## [need port](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)

```
check
netstat -tulpn | grep :port
```

## Forwarding IPv4 and letting iptables see bridged traffic 

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
### Verify Forwarding IPv4 and letting iptables see bridged traffic 

```
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

## [Install Container runtimes and start containerd service](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

```
wget https://github.com/containerd/containerd/releases/download/v1.6.16/containerd-1.6.16-linux-amd64.tar.gz

tar Cxzvf /usr/local containerd-1.6.16-linux-amd64.tar.gz 

create file 
/usr/local/lib/systemd/system/containerd.service

systemctl daemon-reload
systemctl enable --now containerd
```

## [Installing runc](https://github.com/opencontainers/runc/releases)

```
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64

install -m 755 runc.amd64 /usr/local/sbin/runc
```

## [Installing CNI plugins](https://github.com/containernetworking/plugins/releases)

```
wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz

mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.2.0.tgz 
```

## Initial containerd config
```
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

## Configuring the systemd cgroup driver 
To use the systemd cgroup driver in /etc/containerd/config.toml with runc, set
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true


sudo systemctl restart containerd
```
## Closeing swap
```
swapoff -a && sysctl -w vm.swappiness=0
sed '/swap.img/d' -i  /etc/fstab
```

## Installing kubeadm, kubelet and kubectl

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

## Initial kubeadm

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## [Install Calico (CNI)](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart)

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml
watch kubectl get pods -n calico-system
```

## Joining

```
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

