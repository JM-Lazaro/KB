# How to Create a Multi-Node Cluster using Kubeadm on Vagrant VMs

# Introduction

This document aims for you to have a quick Kubernetes Cluster installation. Please check out the links for in-depth explanation of the steps or if you want to deviate from any of the set-up below. 

https://kubernetes.io/docs/setup/independent/install-kubeadm/
https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#before-you-begin


# Vagrant VM Set-up

Create your master and node's vagrant file using the templates below. The only difference with the OOTB config file would be the `config.vm.network` option and hostnames.

## Vagrantfile for Master:

```
Vagrant.configure("2") do |config|
 config.vm.box = "ubuntu/xenial64"                        # ubuntu version 16.04
 config.vm.network "private_network", ip: "192.168.44.10" # private IP within your host, has to be unique
 config.vm.define "k8smaster"                             # vm box name
 config.vm.hostname = "k8s-master"                        # hostname
end
```

## Vagrantfile for Minions:

```
Vagrant.configure("2") do |config|
 config.vm.box = "ubuntu/xenial64"
 config.vm.network "private_network", ip: "192.168.44.20" 
 config.vm.define "k8sminion1" .                           # change depending on how many 
 config.vm.hostname = "k8s-minion1"                        # minions you want to deploy
end
```

## Create the VM for the master node and Connect to the VMs via SSH then switch user to root for convenience:

```
/vagrant> vagrant up
/vagrant> vagrant ssh
ubuntu@k8s-master:~$ sudo su -
root@k8s-master:~#
```

# Kubeadm Installation

## Turn off the page swap

```
swapoff -a
```

## Install Docker CE 17.03

Note: There is an issue with using non 17.03 docker version with `--apiserver-advertise-address option` (will be used later)

```
apt-get update

apt-get install -y apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"

apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')
```

## kubeadm-kubelet-and-kubectl Installation

```
apt-get update && apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update

apt-get install -y kubelet kubeadm kubectl

apt-mark hold kubelet kubeadm kubectl #didnt do this
We won't need to change the kubelet configuration for this set-up but you can run these anyway for kubelet restart:

systemctl daemon-reload
systemctl restart kubelet
```


## Master-only steps

You have to do the steps above for both your master and minion nodes. However, you should only initiate kubeadm on your master.

```
kubeadm init --apiserver-advertise-address=192.168.44.10 --pod-network-cidr=10.244.0.0/16
```

#### Notes:

apiserver-advertise-address
    - The IP address the API server is accessible on, to use for the API server cert.
    - use the vm.network IP that you have defined in the master Vagrantfile

pod-network-cidr
    - required for most Container Network Interfaces (Calico, Flannel)


You will be instructed to executed the commands below. You will need this to make kubectl work properly:

```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### additional installations for flannel

```
sysctl net.bridge.bridge-nf-call-iptables=1

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/c5d10c8/Documentation/kube-flannel.yml
```


## Minion-only steps

At this point, you will only have to run the command that was given to you when you initiated the kubeadm on your master node:

Sample command:
```
kubeadm join 192.168.44.10:6443 --token bbk5kn.djzsw5ws27pss3cu --discovery-token-ca-cert-hash sha256:4e9acb7f661caa1ffe618527149365d15d9590c9fb9103cf9fa4f89e77299f36
```

### steps to enable kubectl in your minion

Copy the file /etc/kubernetes/admin.conf from the master to the minion, same filename and directory then execute the commands:

```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

You should be able to see your nodes like this:

```
root@k8s-master-2:/var/lib/kubelet# kubectl get nodes -o wide
NAME            STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k8s-master-2    Ready    master   46h   v1.13.1   10.0.2.15     <none>        Ubuntu 16.04.3 LTS   4.4.0-97-generic   docker://17.3.3
k8s-minion2-1   Ready    <none>   46h   v1.13.1   10.0.2.15     <none>        Ubuntu 16.04.3 LTS   4.4.0-97-generic   docker://17.3.3
```

Create a yaml file and paste the dd-agent daemonset from our website. You will see that it will only be deployed  in the minion node and not the master:

```
root@k8s-master-2:~# kubectl create -f datadog.yaml  # dd-agent deployment via template daemonset
root@k8s-master-2:/etc/kubernetes# kubectl get po -o wide --all-namespaces
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE   IP           NODE            NOMINATED NODE   READINESS GATES
default       datadog-agent-666fm                    1/1     Running   0          45h   10.244.1.5   k8s-minion2-1   <none>           <none>
```


You may also notice that your master's and minion's Internal-IP is the same. This creates an issue sending commands to the minion:

```
root@k8s-master-2:/etc/kubernetes# kubectl exec -it datadog-agent-666fm agent status
error: unable to upgrade connection: pod does not exist
```

The resolution I found is to append the file /var/lib/kubelet/kubeadm-flags.env with the master's and node's IP accordingly with `--node-ip=<node_IP>`:

```
root@k8s-master-2:/var/lib/kubelet# cat kubeadm-flags.env
KUBELET_KUBEADM_ARGS=--cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --node-ip=192.168.44.10


root@k8s-minion2-1:/var/lib/kubelet# cat kubeadm-flags.env
KUBELET_KUBEADM_ARGS=--cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --node-ip=192.168.44.20
```


You should now be able to connect to your dd-agent pod. It will have a fixable error and I'll leave it to you to resolve as practice.


## Master-isolation

You can execute the command below if you only want to use one VM and deploy the dd-agent in the master node. 

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```
 
Minikube would much easier to use in MacOS, though I haven't made it work in Vagrant VMs.
 
