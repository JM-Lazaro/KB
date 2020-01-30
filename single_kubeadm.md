# How to Create a Single Node Kubeadm in Vagrant Ubuntu

# Introduction

This document aims for you to have a quick Kubernetes Cluster installation. Please check out the links for in-depth explanation of the steps or if you want to deviate from any of the set-up below. 

https://kubernetes.io/docs/setup/independent/install-kubeadm/
https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#before-you-begin


# Vagrant VM Set-up

Create your vagrant instance:

```
vagrant init ubuntu/bionic64
vagrant up
```

# Kubeadm Installation

## Turn off the page swap

```
vagrant ssh 

sudo su -

swapoff -a
```

## Install Docker Community Edition

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -;

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable";

sudo apt-get update;

sudo apt-get install -y docker-ce;
```

## Install Kubernetes: kubeadm-kubelet-and-kubectl

```
apt-get update && apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update

apt-get install -y kubelet kubeadm kubectl

```


## Initialize Kubeadm


```
kubeadm init --pod-network-cidr=10.244.0.0/16
```

You will be instructed to executed the commands below. You will need this to make kubectl work properly:

```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Additional Installations for Flannel

There are several networking options in the official website but let's just use Flannel for now. 

NOTE: Flannel gets updated regularly which makes the second command fail. When that happens, go to this page and get the new link/command: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network

```
sysctl net.bridge.bridge-nf-call-iptables=1

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml

```

Removing  Taints

Taints can be placed in nodes to let Kubernetes know not to schedule pods on them. The master nodes usually have one by default so we 'll have to remove them if we want to deploy pods on them.

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```
  
