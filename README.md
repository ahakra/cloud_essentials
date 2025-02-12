
## Set up a unique name
```bash 
sudo hostnamectl set-hostname "k8s-master01" && exec bash //for master
sudo hostnamectl set-hostname "k8s-worker01" && exec bash //for workes
```


## Append all to hosts
```bash 
sudo nano /etc/hosts
#add all hosts to all nodes
    192.168.56.102 k8s-master01
    192.168.56.103 k8s-worker01
```

## Disable Swap Space on Each Node
```bash 
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
free -h #To check if swap is off and having 0b
```

## Open ports on control nodes
```bash 
sudo firewall-cmd --permanent --add-port={6443,2379,2380,10250,10251,10252,10257,10259,179}/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --reload
```

## Open ports on worker nodes
```bash 
sudo firewall-cmd --permanent --add-port={179,10250,30000-32767}/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --reload
```

## Add kernel modules and parameters
```bash 
sudo echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/containerd.conf && sudo modprobe overlay && sudo modprobe br_netfilter
sudo echo -e "net.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1\nnet.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/k8s.conf
sudo sysctl --system
```

## Install Containerd Runtime
```bash 
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install containerd.io -y
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

🚨 **Important Note:**

> Update to match kubernets version or you will receive a warning when using kubeadm init command \
>    [plugins."io.containerd.grpc.v1.cri"]  
>        sandbox_image = "registry.k8s.io/pause:3.10" \
>       in /etc/containerd/config.toml
## Restart Containerd
```bash
 sudo systemctl restart containerd
 sudo systemctl enable containerd
 sudo systemctl status containerd
```

## Install Kubernetes tools
🚨 **Important Note:** make sure to update kubernetes version.
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF


sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
sudo systemctl status kubelet
```

## Init Kubernetes cluster
```bash
sudo kubeadm init  --control-plane-endpoint=k8s-master01  --pod-network-cidr=10.244.0.0/16 #example
```

## To reset Kubernetes cluster
```bash
sudo kubeadm reset
rm -rf ~/.kube
```

## To start interacting with Kubernetes cluster
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## In case failing  coredns pods under  kubesystem namespace
```bash
#try :
kubectl get node k8s-master01 -o=jsonpath='{.spec.podCIDR}'
#if it didn't return a result , try patching using below command
kubectl get node k8s-master01 -p '{"spec":{"podCIDR":"10.244.0.0/16"}}'
#maybe restart
sudo systemctl restart kubelet


```
## To install calico on control
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml
```

## In case you want to deploy apps on control-plane(Not Recommended)
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

```

## To join workers 
🚨 **Important Note:** when you run kubeadm init, it generates a token, but it might be expired until you set up work,
so use command bellow
```bash
kubeadm token create --print-join-command #on master
#copy and paste output command on worker

```

## To enable IPv4 on RHEL for VirtualBox Host-only Adapter
```bash
sudo nano /etc/sysconfig/network-scripts/ifcfg-enp0s8
TYPE=Ethernet
BOOTPROTO=dhcp
NAME=enp0s8
DEVICE=enp0s8
ONBOOT=yes


sudo systemctl restart NetworkManager

```