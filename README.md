# Multi-Master-HA-Kubernetes-Cluster-Setup :computer:
In this show, we will look into that how can we deploy the Multi-Master HA Kubernetes cluster on CentOS/Redhat. 

## Little Background 
For just little background before going to run the commands on the nodes. Lets talk about the architechure for the cluster setup. In this setup, we have utilized the 8 Virtual Machines having RedHat 7.9 installed on it. The VMs are created on the VMware ESXi Hypervisor. The distribution of VMs are as following:

1. 2 X HAProxy  
2. 3 X Master Nodes
3. 3 X Worker Nodes 

In this setup, we tried to follow the best practices as recommended by the Kubernetes documentation. As we have the two ways to setup the High Available Kubernetes architechure, one of them is stacked architecure and other having seperate etcd cluster nodes. But for this setup, we make the things simple, as we follow to use the stacked architechure in which the etcd component is a part of same control plane having other components of the kubernetes. Furthermore, as the swap should be disabled in the kubernetes nodes, due to which we didn't configure the swap from scratch while creation of a VM. 

We use the two nodes for HAProxy loadbalancer for Master nodes to ensure the HA at loadbalancer level too. All our pods will be run on the woker nodes. Why we need the three Master nodes? For little bit depth, its better to look into the kubernetes official documentation. We setup the cluster by using the kubeadm.


#### Note: As human, we are vulnerable to make mistakes. My repositories are always open for the feedback or recommendation. Please suggest anything that help us to share with community.

#### Note: The reference links are also provided with the commands that will redirect to official doumentation of the kubernetes. 

## Installation 

### Pre-Req
In our setup, the control plane and worker nodes are with same specifications and network too. All have their hosts entries (including Loadbalancers) in /etc/hosts file and they should be sync. Furthermore all the commands will be executed on a single control plane node and after initializing the cluster, the other nodes will join based on their category. 

In case of the loadbalancer, we can use easily find tutorial or blog on that "How to setup the HAProxy as High Available with KeepAliveD" over the internet.

### Installing Container Runtime - Containerd
At this moment, we will not discuss that why we proceed to use the containerD rather then the docker Engine. Its will be good learning oppertunity to read its detail in documentation. 
```bash   
#Ref: https://github.com/containerd/containerd/blob/main/docs/getting-started.md - centos section
#Ref: https://docs.docker.com/engine/install/centos/
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.107-3.el7.noarch.rpm
yum install -y containerd.io
```
```bash   
#setting up that which underline driver will we opt for resouces managment 
containerd config default | tee /etc/containerd/config.toml
sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
```

### Install kubectl, kubeadm, kubelet
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

```bash
# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet-1.25.0-0 kubeadm-1.25.0-0 kubectl-1.25.0-0 --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

### Enable kernel modules
These modules and configuration would allow the communications of pods.
```bash
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

### Setting parameters, Restart the services/daemons
To be more honest, I was stuck to set the parameter values for the kubelet. So one of my friend help me in this regard. For better understanding, please share your knowledge for it. 
```bash
systemctl restart containerd
systemctl enable containerd
systemctl status containerd


echo 'KUBELET_EXTRA_ARGS="--fail-swap-on=false --container-runtime=remote --container-runtime-endpoint=/run/containerd/containerd.sock"' > /etc/sysconfig/kubelet
systemctl daemon-reload
systemctl restart containerd
systemctl start kubelet
```

### Proxy Setup (Optional: If the Kubernetes cluster is behind the proxy)
This step is optional but become crucial when you have setup behind proxy server.
```bash
mkdir /etc/systemd/system/containerd.service.d
cat << EOF > /etc/systemd/system/containerd.service.d/http_proxy.conf
[Service]
Environment="HTTP_PROXY=http://x.x.x.x:<PORT>/"
Environment="HTTPS_PROXY=http://x.x.x.x:<PORT>/"
EOF

cat <<EOF > /etc/systemd/system/containerd.service.d/no_proxy.conf
[Service]
Environment="NO_PROXY=10.96.0.0/8,10.93.98.0/24,10.244.0.0/16,127.0.0.1/8"
EOF

systemctl daemon-reload
systemctl restart containerd
systemctl show conatinerd --property Environment
```

### Pull Images (Optional: otherwise done by after issuing initialzing the cluster)
The image installation step can be done itself by initializing the cluster. But we can also do it separately, as for this i have to use the proxy in ENV variable to pull the images. After pulling the images, I have unset those setted proxy variable to avoid any issue for the kubernetes cluster networking.
```bash
kubeadm config images pull --v=5
unset https_proxy
unset http_proxy
```

### Initialize the kubernetes cluster
Now we are ready to initialize the cluster, as we will provide the loadbalancer hostname/IP and also provided the CIDR. I have used this IP range because, I will use the calico service for networking. 
```bash
kubeadm init --control-plane-endpoint="<LoadBalancer>:6443" --pod-network-cidr=192.168.0.0/16 --v=5
```

Output with joining cluster commands - Don't worry: As it will be provided by the above command in its stdout 
```bash
#command to join the cluster as control plane
kubeadm join <LoadBalancer>:6443 --token <Token> \
        --discovery-token-ca-cert-hash sha256:<hashvalue> \
        --control-plane --certificate-key <hashvalue> --v=5

#command to join the cluster as worker node
kubeadm join <LoadBalancer>:6443 --token <Token> \
        --discovery-token-ca-cert-hash sha256:<hashvalue> \
```

### Calico Networking
For kubernetes networking solution, we opted the calico. We downloaded its deployment YAML file and deployed it on the master node.

<h2 align="center">
CONGRATS - I HOPE THE CLUSTER HAS BEEN SETUP :innocent:
</h2>
        
### RESET THE NODE
In case of any trouble happens on the control plane and worker node, I used this patch to reset the node
```bash
kubeadm reset --v=5

yum remove kubeadm kubelet kubectl containerd -y
yum remove container-selinux -y
rm -rf /root/.kube
rm -rf /etc/cni/net.d

rm -rf /etc/kubernetes/
rm -rf /etc/containerd
rm -rf /etc/modules-load.d/containerd.conf 
rm -rf /etc/systemd/system/containerd.service.d/
rm -rf /etc/containerd/config.toml
rm -rf /etc/systemd/system/kubelet.service.d/
rm -rf /etc/sysconfig/kubelet
```

## THE END 
#### Happy Learning - ALHMD

