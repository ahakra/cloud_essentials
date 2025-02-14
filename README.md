## Config haproxy

```bash
#/etc/haproxy/haproxy.cfg
frontend kubernetes-api
    bind *:6443
    mode tcp
    default_backend kubernetes-masters

backend kubernetes-masters
    mode tcp
    balance roundrobin
    server master1 IPADD_OR_HOSTNAME:6443 check
    server master2 IPADD_OR_HOSTNAME:6443 check

```

## Config KeepAlived For HAPROXY

```
    #/etc/keepalived/keepalived.conf
    vrrp_instance VI_1 {
    state MASTER     # For master
    interface ens33  # Change to your network interface
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass mysecurepassword
    }
    virtual_ipaddress {
        192.30.0.35 #same ip for all
    }
}



vrrp_instance VI_1 {
    state BACKUP	 # For backup
    interface ens33  # Change to your network interface
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass mysecurepassword
    }
    virtual_ipaddress {
        192.30.0.35 #same ip for all
    }
}


```

## Config HAProxy to output log to /var/log/haproxy.log

```
#/etc/rsyslog.d/haproxy.conf

  $ModLoad imudp
  $UDPServerAddress 127.0.0.1
  $UDPServerRun 514

  local2.*       /var/log/haproxy.log

#systemctl restart rsyslog && ldconfig

#/etc/haproxy/haproxy.cfg
global

        log 127.0.0.1 local2

defaults
        log global
        option httplog


```

##

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

## For calico on all nodes and others

```bash
    ## one error of calico
    ##bird: Unable to open configuration file /etc/calico/confd/config/bird.cfg: No such file or directory bird: Unable to open configuration ##file /etc/calico/confd/config/bird6.cfg: No such file or directory

    sudo semanage port -a -t http_port_t -p tcp 6443
	sudo semanage port -a -t http_port_t -p tcp 2379
	sudo semanage port -a -t http_port_t -p tcp 2380

    sudo iptables -A INPUT -p tcp --dport 179 -j ACCEPT
	sudo iptables -A INPUT -p tcp --dport 9099 -j ACCEPT
	sudo iptables -A INPUT -p tcp --dport 5473 -j ACCEPT
	sudo iptables-save
	sudo firewall-cmd --zone=public --add-port=179/tcp --permanent
	sudo firewall-cmd --zone=public --add-port=9099/tcp --permanent
	sudo firewall-cmd --zone=public --add-port=5473/tcp --permanent
	sudo firewall-cmd --reload

    #additional step maybe needed
    #on each node
    sudo iptables -A INPUT -s  ALL_OTHER_NODES -p tcp --dport 179 -j ACCEPT

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
>  [plugins."io.containerd.grpc.v1.cri"]  
>  sandbox_image = "registry.k8s.io/pause:3.10" \
>  in /etc/containerd/config.toml

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
sudo kubeadm init  --control-plane-endpoint=k8s-master01  --pod-network-cidr=192.168.0.0/16 #example
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

## In case failing coredns pods under kubesystem namespace

```bash
#try :
kubectl get node k8s-master01 -o=jsonpath='{.spec.podCIDR}'
#if it didn't return a result , try patching using below command
kubectl get node k8s-master01 -p '{"spec":{"podCIDR":"192.168.0.0/16"}}'
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

## To join master

```bash
    kubeadm init phase upload-certs --upload-certs #on master
    kubeadm join ip-address:6443 --token TOKENID --discovery-token-ca-cert-hash sha256:19d029f15366d2b098db58c29c6fdsa216d1f4612a40eff5c6dc04007505f44b05cb \
		--control-plane --certificate-key FROMPREVIOUSSTEP --v=6
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

🚨 **Important Note:** In case of doing kubeadm for master node, and when joing back i cause errors like
2379","attempt":0,"error":"rpc error: code = Unavailable desc = etcdserver: unhealthy cluster"

```bash
# execute bash on any etcd
etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key  member list -w table
etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key member remove fd63c5ff2b206142
```

## Installing etcdctl

```bash
kubectl get pods -n kube-system -l component=etcd -o jsonpath="{.items[0].spec.containers[0].image}"
ETCD_VERSION="3.5.16" #based on line before
wget https://github.com/etcd-io/etcd/releases/download/v${ETCD_VERSION}/etcd-v${ETCD_VERSION}-linux-amd64.tar.gz
tar xvf etcd-v3.5.16-linux-amd64.tar.gz
sudo mv etcd-v3.5.16-linux-amd64/etcdctl /usr/local/bin/
sudo mv etcd-v3.5.16-linux-amd64/etcdutl /usr/local/bin/

etcdctl version
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

```

## Backup/Restore ETCD

```bash
kubectl describe -n kube-system pod etcd-kub-red03 | grep -i file
>output
>SeccompProfile:       RuntimeDefault
>      --cert-file=/etc/kubernetes/pki/etcd/server.crt
>      --key-file=/etc/kubernetes/pki/etcd/server.key
>      --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
>      --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
>      --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
>      --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt


ETCDCTL_API=3 etcdctl snapshot save \
  --endpoints=https://127.0.0.1:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  /opt/etcd-backup-$(date +"%Y-%m-%d_%H-%M-%S").db

ETCDCTL_API=3 etcdutl snapshot status /opt/etcd-backup-name.db

#get dir of data
kubectl -n kube-system get pod -l component=etcd -o yaml | grep -A 5 "command:"

#Check volume mounts
kubectl -n kube-system get pod -l component=etcd -o yaml | grep -A 10 "volumeMounts"


ETCDCTL_API=3 etcdctl snapshot restore /var/lib/etcd/backup.db \
  --data-dir /var/lib/etcd

systemctl restart kubelet

```
