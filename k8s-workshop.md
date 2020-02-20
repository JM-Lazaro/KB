
## Environment Installation

GCP Free Trial - https://cloud.google.com/free
Kubeadm - https://github.com/DataDog/se-docs/wiki/How-to-Create-a-Single-Node-Kubeadm-in-Vagrant-Ubuntu

## Agent Installation Details

Installation Page: https://app.datadoghq.com/account/settings#agent/kubernetes

## Run the installation commands:

** Download the files into your local machine so you can deploy/terminate Kubernetes objects easily.

https://github.com/DataDog/datadog-agent/tree/master/Dockerfiles/manifests

* `kubetcl create -f .
* `kubectl replace --force -f datadog-agent.yaml`


Common Commands:

```
kubectl get pods
kubectl get pods --all-namespaces
kubectl get pods -o wide
kubectl create -f datadog-agent.yaml
kubectl delete <object>
kubectl replace --force -f datadog.yaml
kubectl exec -it <podname> agent status
kubectl exec -it <podname> agent flare
kubectl logs <podname>
```

## Kubernetes Controllers

For testing Datadog functionalites, you can use the single-node environments. You can use your GKE trial to get a better grasp on multi-node clusters. 

#### Daemonsets

Deploy the agent on your GKE cluster.

* `kubectl create -f datadog-agent.yaml`

Check out the daemonset in the `default` namespace.

* `kubectl get ds -n default`
* `kubectl get ds <daemonset name> -n default -o yaml`
* `kubectl describe ds -n default`

Check out the pod created by the daemonset:

* `kubectl describe po <podname> -o yaml`

Delete an agent pod:

* `kubectl delete pod <podName>`

Notice that the pod will be replaced by a new one. This is because a DaemonSet controller ensures that there's one copy of its pod per host.

Delete the DaemonSet:

* `kubectl delete ds datadog-agent`

All pods created by the daemonset will be terminated as well.

* `kubectl delete ds datadog-agent`

#### Deployment

Change the `Kind:` parameter of the agent manifest from daemonSet to Deployment and ~deploy~ install.

Check out the Deployment in the `default` namespace.

* `kubectl get deploy -n default`
* `kubectl get deploy <daemonset name> -n default -o yaml`
* `kubectl describe deploy -n default`

Check out the pod created by the Deployment:

* `kubectl describe po <podname> -o yaml`

Delete an agent pod:

* `kubectl delete pod <podName>`

Notice that the pod will be replaced by a new one. This is because a DaemonSet controller ensures that there's one copy of its pod per host.

#### Networking


##### containerPort

Copy this manifest into your local directory and use it to create a simple nginx pod. 

* https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
* `kubectl create -f nginx_pod.yaml`

Get the pod IP:

* `kubectl get po -o wide`

 Get the Kubelet Host IP:
 
* `kubectl get no -o wide` # Internal IP

Check the connection from the agent from the same host using IPs:

* `kubectl exec -it <agent> curl <podIP>:80`
* `kubectl exec -it <agent> curl <hostIP>:80`


##### hostPort

Uncomment the `hostPort` section from the nginx manifest and re-deploy:

* `kubectl replace --force -f nginx_pod.yaml`

Check the connection from the agent from a different host.

* `kubectl exec -it <agent> curl <podIP>:80`
* `kubectl exec -it <agent> curl <hostIP>:80`

##### Services

Copy this Service manifest into your local directory and deploy:

* https://gist.github.com/JM-Lazaro/6961f15195ae8cb7a72db76426006492#file-2_service-yaml
* `kubectl create -f nginx_service.yaml`

Check the Service's details:

* `kubectl get svc`

You can connect to all pods with label `app: reverse-proxy` using the ClusterIP.

* `kubectl exec -it <agent> curl <clusterIP>:80`

#### AutoDiscovery


