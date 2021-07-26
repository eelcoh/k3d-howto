
# Install Kubernetes on a single Fedora 31 host

There are many reasons why you should want to run Kubernetes across a couple of (virtual) machines. Nonetheless, there are still reasons for just having a simple single host kubernetes setup; for dev or test, for one of those side projects you once stared but never finished. Or just to learn the concepts.

In this document, i explain how to creae a simple cluster based on K3D and K3S, and using Contour as an Ingress Controller. I assume that you have a clean Fedora (31) install.  

## Install docker
### Revert cgoups
Kubernetes doesn't play well with the cgroup v2 that Fedora uses. You'll have to revert back to v1:
```
$ sudo dnf install -y grubby
$ sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
$ sudo reboot
```

### Install docker
We'll have to add the docker repo, then install docker-ce, and enable the service
```
$  sudo dnf config-manager --add-repo=https://download.docker.com/linux/fedora/docker-ce.repo
$  sudo dnf install docker-ce
$  sudo systemctl enable --now docker
$  systemctl status docker
```

Enable regular user to perform Docker administration tasks. First, create the docker group:
```
$  sudo groupadd docker
$  sudo usermod -aG docker eelco
```

## Install Kubernetes tools
We need to install the kubernetes tools first. Add the file `kubernetes.repo` to `/etc/yum.repos.d/`, and paste this as contents:
```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
```

Complete the repo addition with
```
$ sudo dnf upgrade -y --disableexcludes=kubernetes
```

And install `kubectl`:
```
$ sudo dnf install kubectl --disableexcludes=kubernetes
```

## Install Kubernetes
We are going to use K3D and K3S for kubernetes. K3D creates the K3S clusters.
We will use the description from https://www.sokube.ch/post/k3s-k3d-k8s-a-new-perfect-match-for-dev-and-test but instead of using traefik or nginx as an ingress controller, we will be using contour.

K3D can be installed in multiple ways. Please check []. For now, we will install it with the script:
```
$ wget -q -O - https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```

That is all. Now let's create a cluster named play. While we do, we will tell K3D not to install traefik. 
```
$  k3d cluster create play --servers 3 --k3s-server-arg '--no-deploy=traefik' 
```

K3D will now create the cluster for us, with three server nodes. Check whether everything went well with:
```
$  kubectl get all --all-namespaces
```

We are going to use the famous kuard application as the hello world application. We will create a pod, a deployment, a service and an ingress, all in one file. Also here, you can get it at the `https://projectcontour.io/examples/kuard.yaml`, but again there is a slight error in it, as it uses an outdated version of the ingress controller. So here is the yaml, store it as `kuard.yaml:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:1
        name: kuard
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: kuard
  sessionAffinity: None
  type: ClusterIP
```
This installs the pods and the service. 
```
$ kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
kuard   3/3     3            3           25h

$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
kuard-798585497b-2grn2   1/1     Running   0          25h
kuard-798585497b-bprvb   1/1     Running   0          25h
kuard-798585497b-xvzvd   1/1     Running   0          25h

$ kubectl get services
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kuard        ClusterIP   10.43.138.6   <none>        80/TCP    25h
kubernetes   ClusterIP   10.43.0.1     <none>        443/TCP   25h
```

That kaurd servcie is the one we would like to use. However, it doesn't have an external IP, and because of that, we can't reach it from outside the cluster. Let's make sure we can reach it through the Contour.

## Install Contour for Ingress
Now let's install Contour. We will first install the Contour operator, a helper application for actually installing Contour itself.
```
$  kubectl apply -f https://projectcontour.io/quickstart/operator.yaml
```

Once that is done we can install Contour itself. The Contour quickstart guide tells us to do a `kubectl apply -f https://projectcontour.io/quickstart/contour-custom-resource.yaml`, but unfortunately, at the time of writing there is an error in there as it specifies Contour to be a NodePortService, instead of a LoadBalancerService. So we will fix that. Fire up your vi and create this file and save it as `contour-custom-resource.yaml`
``` 
apiVersion: operator.projectcontour.io/v1alpha1
kind: Contour
metadata:
  name: contour-sample
spec:
  networkPublishing:
    envoy:
      type: LoadBalancerService
```

Then send it to the operator:
```
$ kubectl apply -f contour-custom-resource.yaml
```


We will use the Ingress resource for this. The Ingress is a standard Kubernetes resource. The main purpose of Ingress is to route the traffic from outside the cluster to the right service, based on a set of rules. Read about the set of rules here: https://kubernetes.io/docs/concepts/services-networking/ingress/ , but for here we will use a very simple one, with only a defaultBackend.

Save this file as kuard-ingress.yaml

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kuard
spec:
  defaultBackend:
    service:
      name: kuard
      port:
        number: 80

```

Run it:
```
$ kubectl apply -f kuard-ingress.yaml
```

> Although - or perhaps because - the Ingress is a standard Kubernetes resource, it is rather limited. Contour implements the Ingress, but next to that it also implements its own custom HTTPProxy resource, which is much richer and comes with a comprehensive set of optiions. Read more about it here: https://projectcontour.io/docs/main/config/fundamentals/


Now we will get an ingress of type loadbalancer. In Fedora, this will work straight away (on a Mac you would have to do some port forwarding). We will need to find the external-ip address of the load balancer:
```
$ kubectl get -n projectcontour service envoy -o wide        
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
envoy   LoadBalancer   10.43.155.243   172.21.0.2    80:31346/TCP,443:32620/TCP   65m   app=envoy
```

Now fire up your browser on fedora, or curl to http://172.21.0.2/ . If you use your browser, go to the SERVER ENV tab and find the hostname. Now reload a couple of times and see that you are actually ending up on a different instance when you do so. Or at least, most of the time. We can also query the API instead:
``` 
$ curl http://172.21.0.2/env/api | jq
{
  "commandLine": [
    "/kuard"
  ],
  "env": {
    "HOME": "/",
    "HOSTNAME": "kuard-798585497b-bprvb",
    "KUARD_PORT": "tcp://10.43.138.6:80",
    "KUARD_PORT_80_TCP": "tcp://10.43.138.6:80",
    "KUARD_PORT_80_TCP_ADDR": "10.43.138.6",
    "KUARD_PORT_80_TCP_PORT": "80",
    "KUARD_PORT_80_TCP_PROTO": "tcp",
    "KUARD_SERVICE_HOST": "10.43.138.6",
    "KUARD_SERVICE_PORT": "80",
    "KUBERNETES_PORT": "tcp://10.43.0.1:443",
    "KUBERNETES_PORT_443_TCP": "tcp://10.43.0.1:443",
    "KUBERNETES_PORT_443_TCP_ADDR": "10.43.0.1",
    "KUBERNETES_PORT_443_TCP_PORT": "443",
    "KUBERNETES_PORT_443_TCP_PROTO": "tcp",
    "KUBERNETES_SERVICE_HOST": "10.43.0.1",
    "KUBERNETES_SERVICE_PORT": "443",
    "KUBERNETES_SERVICE_PORT_HTTPS": "443",
    "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
  }
}
```

## Expose Contour to the WAN
The 172.21.0.2 address is internal to the Fedora box. The box itself has ip address 192.168.178.67. So what we want to do is make sure that any request to port 80 on 192.168.178.67 is relayed to the same port on 172.21.0.2. This requires some changes on firewalld-cmd.First let's check the zone that is active.

```
$ sudo firewall-cmd --list-all 
FedoraServer (active)
  target: default
  icmp-block-inversion: no
  ...
```

So the zone is `FedoraServer`. Now let's make relay the traffic. All incoming tcp traffic on port 80 on the host is being relayed to port 80 on 172.21.0.2:
```
$ sudo firewall-cmd --zone=FedoraServer --add-forward-port=port=80:toport=80:proto=tcp:toaddr=172.21.0.2
```
Now you can go and call the API from anywhere in your network.

``` 
$ curl http://192.168.178.67/env/api | jq
{
  "commandLine": [
    "/kuard"
  ],
  "env": {
    "HOME": "/",
    "HOSTNAME": "kuard-798585497b-bprvb",
    "KUARD_PORT": "tcp://10.43.138.6:80",
    "KUARD_PORT_80_TCP": "tcp://10.43.138.6:80",
  ...
```

## Accessing services on the host
If you want to have your pods access a database that is running directly on your host, you may be in for a nasty surprise. It may not be that trivial to reach it. `localhost` does not mean a thing to k3d. There is two things you need to know:
1. Instead of `localhost`, you should use `host.k3d.internal`.
2. Your database needs to listen to `0.0.0.0` instead of `127.0.0.1`. 

## Accessing services on other hosts in the network
This one i still have to figure out - did not spend too much time on it.

The interesting thing is that k3d uses a different dns than you might expect: `8.8.8.8`. This is the google dns. But that one won't resolve the names of the hosts you have in your network. So if your database is actually running on big-database-machine.your-network, it won't find it.


## Clean up
That is it. Delete your cluster to clean up:
```
$ k3d delete cluster play
```
## What's next
Now you've seen how to create a Kubernetes cluster, you may want to dive into that other big thing, the Service Mesh. Try to get Istio running on this cluster, and inspect it with Kalio..

## Attribution
I did not make this all up. I used multiple sources, and combined these into a sinle story. Most notably:
* [How to install Kubernetes on CentOS 8](https://upcloud.com/community/tutorials/install-kubernetes-cluster-centos-8/)
* [K3S + K3D = K8S : a new perfect match for dev and test](https://www.sokube.ch/post/k3s-k3d-k8s-a-new-perfect-match-for-dev-and-test)
* [Getting started with Contour](https://projectcontour.io/getting-started/)

