# k8s-installation-kubeadm

##  Pre-requisite 

*     Master node - 2 cpu x 2 GB memory
*     Worker node - 1 cpu x 2 GB memory

## Pre-installation steps

###  The below steps will be performed on both master and worker node 

*  1.  Turn of Swap

`   apt-get update`

`   swapoff -a ` 

*  2.  Comment swap FS from /etc/fstab 

`   vi /etc/fstab`

`   Comment any line that has swap written` 

*  3.  Edit /etc/hostname and edit hostname to match the host of your choice 

*  4.  Get private ip address of all hosts 

`   ip addr ` 

*  5.  Edit /etc/hosts to add hostname and IP address on all nodes 

`   vi /etc/hosts ` 

~~~
    kmaster 192.168.0.2  -- private IP address from previous step
    knode1  192.168.0.3  -- provate IP address from previous step 
~~~


## Installation Procedure 

### Install kubelet kubeadm and kubectl 

* **Ubuntu**

```
apt-get update && apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update

apt-get install -y kubelet kubeadm kubectl

apt-mark hold kubelet kubeadm kubectl

```

* **Centos**

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

setenforce 0

sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet

```

### Install Docker 

* **Ubuntu**

```
sudo apt-get update

sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```

* **Centos**

```
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

sudo yum install -y docker-ce docker-ce-cli containerd.io --nobest

systemctl start docker

```


### Configure Cluster using kubeadm

**The below steps will be performed** __**ONLY ON MASTER NODE**__

* Get the IP address of master

```
ip addr
```


* Initialize the cluster

```
kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address={{IP_ADDR_MASTER}}

```

* Your output should look like below - 

```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.138.0.2:6443 --token 3ccgnq.2owa1scoiqqoqhdq \
    --discovery-token-ca-cert-hash sha256:04ff7a9148ae02db8a4be70f016fe66d6d5870ea09641e48f6a7db54747e1acd 

```

Preserve the above output as it contains the token required for node configuration. 

* Copy over the configuration files

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```


* Install Networking component (CNI)

```
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml

```



* Join the worker node 


The below steps are to be run __**only on the worker node**__ 

The output of the kubeadm init command will provide the kubeadm join as its output. Run the kubeadm join command on the worker nodes. 



```
kubeadm join 10.138.0.2:6443 --token 3ccgnq.2owa1scoiqqoqhdq \
    --discovery-token-ca-cert-hash sha256:04ff7a9148ae02db8a4be70f016fe66d6d5870ea09641e48f6a7db54747e1acd 

```

The output will be as below 

```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

```


Install a Kubernetes Cluster on RHEL8 with conatinerd:


For this, we will walk-through a multi-node Kubernetes cluster installation on RHEL 8. This tutorial is command-line based so you will need access to your terminal window . Will be performing the steps on GCP.
Starting from RHEL 8, docker has now natively been replaced by podman and buildah which are tools from Redhat. As a matter of fact, the docker package has now been removed from the default package repository and at the end we all are working with the same API , then we don’t have to lock-in into a specific tool.
These are the popular Container runtimes and being used mainly
•	Docker
•	CRI-O
•	Containerd
For this cluster , we are going to use Containerd as it’s Container runtimes.
Prerequisites:
1.	Three nodes running RHEL 8 out from which 1 Master Node and 2 Worker Nodes.
2.	It is recommended that your nodes should have at least 2 CPUs with 2GB RAM or more per machine. This is not a strict requirement but good to have.
3.	Internet connectivity on all your nodes. We will be fetching Kubernetes and other required packages from the repository. Equally, you will need to make sure that the DNF package manager is installed by default and can fetch packages remotely.
* showcased only worker node
Installation of Kubernetes Cluster on Master-Node
Kubernetes makes use of various ports for communication and access and these ports need to be accessible to Kubernetes and not limited by the firewall. If your cluster behaves abnormally , you can configure the firewall rules on the ports.
 
 
k8ports
Step 1: Login to the node and here i will be performing all the operation on root
[root@k8master ~]# cat /etc/redhat-release
Red Hat Enterprise Linux release 8.2 (Ootpa)
________________________________________
Step 2 : Install Container runtime on RHEL 8
This section contains the necessary steps to use containerd as CRI runtime.
Use the following commands to install Containerd on your system:
Prerequisites

cat > /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
Install containerd
### Install required packages
yum install dnf
dnf install -y  yum-utils device-mapper-persistent-data lvm2


## Add docker repository

yum install dnf-plugins-core
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

Adding repo from: https://download.docker.com/linux/centos/docker-ce.repo

[root@k8master yum.repos.d]# dnf update -y && dnf install -y containerd.io

## Configure containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# Restart containerd 
systemctl restart containerd

# Enable containerd on boot root@k8master ~]# 
systemctl enable containerd
Step 3: Install Kubernetes (Kubeadm, kubelet and kubectl) on RHEL 8
Next, you will need to add Kubernetes repositories manually as they do not come installed by default on RHEL8.
Kubeadm helps you bootstrap a Kubernetes cluster. With kubeadm, you are going to create/enable single-control-plane.
Kubeadm also supports other cluster lifecycle functions, such as upgrades, downgrade, and managing bootstrap tokens
# Add yum repo file for Kubernetes  # cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
#Install Kubernetes (kubeadm, kubelet and kubectl) 
[root@k8master ~]# dnf install -y kubeadm-1.17.0 kubelet-1.17.0 kubectl-1.17.0
When the installation completes successfully, enable and start the kubelet service
[root@k8master ~]# systemctl enable kubelet

[root@k8master ~]# echo 'KUBELET_EXTRA_ARGS="--fail-swap-on=false"' > /etc/sysconfig/kubelet

[root@k8master ~]# systemctl start kubelet
Step 4: Create a control-plane Master with kubeadm
[root@k8master]# kubeadm init --pod-network-cidr=192.168.0.0/16
--
--
--
Your Kubernetes control-plane has initialized successfully!
Next, copy the following join-token and store it somewhere, as we required to run this command on the worker nodes later.
kubeadm join 10.128.15.211:6443 --token 0xbszv.o9j5fim3j21xz47a \    --discovery-token-ca-cert-hash sha256:876637c5e5c74fcafa05372d187b58d36aa418b9faa46e1ec4c7388443143087
Once Kubernetes initialized successfully, you must enable your user to start using the cluster. In our scenario, we will be using the root user. You can also start the cluster using sudo user as shown.
To use root, run:
To start using your cluster, you need to run the following as a regular user:  

mkdir -p $HOME/.kube 
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
chown $(id -u):$(id -g) $HOME/.kube/config
Now confirm that the kubectl command is activated.
[root@k8master ~]# kubectl get nodes 
NAME       STATUS     ROLES    AGE   VERSION
k8master   NotReady   master   35m   v1.17.0
ATM, you will see the status of the k8-master is ‘NotReady’. This is because we are yet to deploy the overlay network for the cluster.
Step 5: Setup Your Pod Network
We are going to use calico as our CNI or pod network

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# You should see the following output.configmap/calico-config createdcustomresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org createdclusterrole.rbac.authorization.k8s.io/calico-kube-controllers createdclusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers createdclusterrole.rbac.authorization.k8s.io/calico-node createdclusterrolebinding.rbac.authorization.k8s.io/calico-node createddaemonset.apps/calico-node createdserviceaccount/calico-node createddeployment.apps/calico-kube-controllers createdserviceaccount/calico-kube-controllers created
Now if you check the status of your master-node, it should be ‘Ready’.
[root@k8master ~]# kubectl get nodes 
NAME  STATUS  ROLES  AGE  VERSION
k8master Ready master 43m v1.17.0 
Next, we add the worker nodes to the cluster.
Adding Worker Nodes to Kubernetes Cluster
You can follow the same steps starting from 1-3 i.e till( Kubeadm, kubelet and kubectl) on the worker nodes and once everything set up on the worker node.
Join the Worker Node to the Kubernetes Cluster
We now require the token that kubeadm init generated, to join the cluster. You can copy and paste it to your k8worker if you had copied it somewhere.
kubeadm join 10.128.15.211:6443 --token 0xbszv.o9j5fim3j21xz47a \    --discovery-token-ca-cert-hash sha256:876637c5e5c74fcafa05372d187b58d36aa418b9faa46e1ec4c7388443143087
Or you can generate token freshly by executing the same command, if you have not stored the command.
kubeadm token create --print-join-command
Go back to your master-node and verify if worker k8worker have joined the cluster using the following command.
root@k8mrhel swapnasagar_pradhan]# kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
k8master   Ready    master   5h23m   v1.17.0
k8worker   Ready    <none>   5h6m    v1.17.0
If all the steps run successfully, then, you should see k8worker in ready status on the master-node. At this point, you have now successfully deployed a Kubernetes cluster on CentOS 8



----------------------------------------------------
For this, we will walk-through a multi-node Kubernetes cluster installation on RHEL 7. This tutorial is command-line based so you will need access to your terminal window . Will be performing the steps on GCP.
Starting from RHEL 8, docker has now natively been replaced by podman and buildah which are tools from Redhat. As a matter of fact, the docker package has now been removed from the default package repository and at the end we all are working with the same API , then we don’t have to lock-in into a specific tool.
These are the popular Container runtimes and being used mainly
•	Docker
•	CRI-O
•	Containerd
For this cluster , we are going to use Containerd as it’s Container runtimes.
Prerequisites:
1.	Three nodes running RHEL 8 out from which 1 Master Node and 2 Worker Nodes.
2.	It is recommended that your nodes should have at least 2 CPUs with 2GB RAM or more per machine. This is not a strict requirement but good to have.
3.	Internet connectivity on all your nodes. We will be fetching Kubernetes and other required packages from the repository. Equally, you will need to make sure that the DNF package manager is installed by default and can fetch packages remotely.
* showcased only worker node
Installation of Kubernetes Cluster on Master-Node
Kubernetes makes use of various ports for communication and access and these ports need to be accessible to Kubernetes and not limited by the firewall. If your cluster behaves abnormally , you can configure the firewall rules on the ports.
 
 
k8ports
Step 1: Login to the node and here i will be performing all the operation on root
[root@k8master ~]# cat /etc/redhat-release
Red Hat Enterprise Linux release 8.2 (Ootpa)
________________________________________
Step 2 : Install Container runtime on RHEL 7
This section contains the necessary steps to use containerd as CRI runtime.
Use the following commands to install Containerd on your system:
Prerequisites
cat > /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.

cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
Install containerd
### Install required packages
yum install dnf
dnf install -y  yum-utils device-mapper-persistent-data lvm2


## Add docker repository
yum install dnf-plugins-core
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
Adding repo from: https://download.docker.com/linux/centos/docker-ce.repo

[root@k8master yum.repos.d]# dnf update -y && dnf install -y containerd.io

## Configure containerd

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# Restart containerd 
systemctl restart containerd

# Enable containerd on boot root@k8master ~]# 
systemctl enable containerd
Step 3: Install Kubernetes (Kubeadm, kubelet and kubectl) on RHEL 8
Next, you will need to add Kubernetes repositories manually as they do not come installed by default on RHEL8.
Kubeadm helps you bootstrap a Kubernetes cluster. With kubeadm, you are going to create/enable single-control-plane.
Kubeadm also supports other cluster lifecycle functions, such as upgrades, downgrade, and managing bootstrap tokens
# Add yum repo file for Kubernetes  # cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

#Install Kubernetes (kubeadm, kubelet and kubectl) 

[root@k8master ~]# dnf install -y kubeadm-1.17.0 kubelet-1.17.0 kubectl-1.17.0
When the installation completes successfully, enable and start the kubelet service
[root@k8master ~]# systemctl enable kubelet

[root@k8master ~]# echo 'KUBELET_EXTRA_ARGS="--fail-swap-on=false"' > /etc/sysconfig/kubelet[root@k8master ~]# systemctl start kubelet
Step 4: Create a control-plane Master with kubeadm
[root@k8master]# kubeadm init --pod-network-cidr=192.168.0.0/16
--
--
--
Your Kubernetes control-plane has initialized successfully!
Next, copy the following join-token and store it somewhere, as we required to run this command on the worker nodes later.
kubeadm join 10.128.15.211:6443 --token 0xbszv.o9j5fim3j21xz47a \    --discovery-token-ca-cert-hash sha256:876637c5e5c74fcafa05372d187b58d36aa418b9faa46e1ec4c7388443143087
Once Kubernetes initialized successfully, you must enable your user to start using the cluster. In our scenario, we will be using the root user. You can also start the cluster using sudo user as shown.
To use root, run:
To start using your cluster, you need to run the following as a regular user:  
mkdir -p $HOME/.kube 
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
chown $(id -u):$(id -g) $HOME/.kube/config
Now confirm that the kubectl command is activated.
[root@k8master ~]# kubectl get nodes 
NAME       STATUS     ROLES    AGE   VERSION
k8master   NotReady   master   35m   v1.17.0
ATM, you will see the status of the k8-master is ‘NotReady’. This is because we are yet to deploy the overlay network for the cluster.
Step 5: Setup Your Pod Network
We are going to use calico as our CNI or pod network
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml# You should see the following output.configmap/calico-config createdcustomresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org createdcustomresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org createdclusterrole.rbac.authorization.k8s.io/calico-kube-controllers createdclusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers createdclusterrole.rbac.authorization.k8s.io/calico-node createdclusterrolebinding.rbac.authorization.k8s.io/calico-node createddaemonset.apps/calico-node createdserviceaccount/calico-node createddeployment.apps/calico-kube-controllers createdserviceaccount/calico-kube-controllers created

wget https://docs.projectcalico.org/manifests/calico.yaml
 
Now if you check the status of your master-node, it should be ‘Ready’.
[root@k8master ~]# kubectl get nodes 
NAME  STATUS  ROLES  AGE  VERSION
k8master Ready master 43m v1.17.0 
Next, we add the worker nodes to the cluster.
Adding Worker Nodes to Kubernetes Cluster
You can follow the same steps starting from 1-3 i.e till( Kubeadm, kubelet and kubectl) on the worker nodes and once everything set up on the worker node.
Join the Worker Node to the Kubernetes Cluster
We now require the token that kubeadm init generated, to join the cluster. You can copy and paste it to your k8worker if you had copied it somewhere.
kubeadm join 10.128.15.211:6443 --token 0xbszv.o9j5fim3j21xz47a \    --discovery-token-ca-cert-hash sha256:876637c5e5c74fcafa05372d187b58d36aa418b9faa46e1ec4c7388443143087
Or you can generate token freshly by executing the same command, if you have not stored the command.
kubeadm token create --print-join-command
Go back to your master-node and verify if worker k8worker have joined the cluster using the following command.
root@k8mrhel swapnasagar_pradhan]# kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
k8master   Ready    master   5h23m   v1.17.0
k8worker   Ready    <none>   5h6m    v1.17.0
If all the steps run successfully, then, you should see k8worker in ready status on the master-node. At this point, you have now successfully deployed a Kubernetes cluster on CentOS 8








