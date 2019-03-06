# install-kubernetes-virtualbox
This recipe will lead you through the installation of Kubernetes on a single Workstation running VirtualBox.
I created two VMs (master and slave with Ubuntu 16.04.05.
The master was created with 2 cores and 2GB RAM, the slave with 1 core and 1 GB.
I used bridged adapter for the Network.
Start with 
```
sudo apt-get install ssh
```

ssh in the VM from remote host and do some modifications.
```
sudo swapoff -a
```
modify  vi /etc/fstab accordingly

```
sudo apt-get update
sudo apt-get upgrade
```

check ssh status with
```
sudo service ssh status
```
ssh in the VM and create root password abd do some basic installations:
```
sudo passwd root
sudo apt install git curl
```
Now install docker (taken from https://docs.docker.com/install/linux/docker-ce/ubuntu/)    
```
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```   
doublecheck whether docker daemon is running
```   
sudo docker run hello-world
```   

su
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add
touch /etc/apt/sources.list.d/kubernetes.list
```
insert the following line 
deb http://apt.kubernetes.io/ kubernetes-xenial main
Now move on as regular user
```
sudo apt-get update
```
Kubernetes requires a Pod Network for the pods to communicate. For this guide we will use Flannel although there are several other Pod Networks available.
```
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
sudo modprobe br_netfilter
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```
We can now initialize Kubernetes by running the initialization command and passing --pod-network-cidr which is required for Flannel to work correctly
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf

```
Once Kubernetes has been initialized we then install the Flannel Pod Network by running
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
```
Now you should see all the Pods with 

kubectl get pods --all-namespaces
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
default       nginx-5d66c8bcb9-74cvd           0/1     Pending   0          2m3s
kube-system   coredns-86c58d9df4-qqzhj         0/1     Pending   0          12m
kube-system   coredns-86c58d9df4-thnt9         0/1     Pending   0          12m
kube-system   etcd-master                      1/1     Running   0          12m
kube-system   kube-apiserver-master            1/1     Running   0          12m
kube-system   kube-controller-manager-master   1/1     Running   0          12m
kube-system   kube-proxy-tgcbv                 1/1     Running   0          12m
kube-system   kube-scheduler-master            1/1     Running   0          12m
```
You can also see the nodes that are joining the Kubernetes cluster
```
thomas@master:~$ kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
master   NotReady   master   29m   v1.13.4
thomas@master:~$ 
```

Usually you have workernodes in your cluster, but if yu want to run Pods on the master node you can do by untaint the master node with 
```
kubectl taint nodes --all node-role.kubernetes.io/master-

```

git clone 
https://github.com/schoenemeyer/Kubernetes-GPU-Guide.git

cd ~/Kubernetes-GPU-Guide/scripts



thomas@master:~$ kubectl run my-nginx --image=nginx --replicas=2 --port=80 --expose --service-overrides='{ "spec": { "type": "LoadBalancer" }}'
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
service/my-nginx created
deployment.apps/my-nginx created
thomas@master:~$ kubectl get po
NAME                        READY   STATUS              RESTARTS   AGE
guids-68898f7dc9-nhdfv      1/1     Running             1          12h
my-nginx-64fc468bd4-mhb8p   0/1     ContainerCreating   0          3s
my-nginx-64fc468bd4-s2xdp   0/1     ContainerCreating   0          4s
thomas@master:~$ kubectl get service/my-nginx
NAME       TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
my-nginx   LoadBalancer   10.111.141.110   <pending>     80:31535/TCP   54s
thomas@master:~$ kubectl delete deployment,service my-nginx
deployment.extensions "my-nginx" deleted
service "my-nginx" deleted
thomas@master:~$ kubectl get po
NAME                        READY   STATUS        RESTARTS   AGE
guids-68898f7dc9-nhdfv      1/1     Running       1          12h
my-nginx-64fc468bd4-mhb8p   1/1     Terminating   0          117s
my-nginx-64fc468bd4-s2xdp   1/1     Terminating   0          118s
thomas@master:~$ 
  
thomas@master:~$ kubectl get namespaces
NAME          STATUS   AGE
default       Active   14h
kube-public   Active   14h
kube-system   Active   14h
thomas@master:~$ 

kubectl config view
kubectl config current-context
kubectl get namespaces
kubectl delete namespaces


example
kubectl create -f https://k8s.io/examples/admin/namespace-dev.json
kubectl create -f https://k8s.io/examples/admin/namespace-prod.json
kubectl get namespaces --show-labels

NAME          STATUS   AGE    LABELS
default       Active   14h    <none>
development   Active   103s   name=development
kube-public   Active   14h    <none>
kube-system   Active   14h    <none>
production    Active   14s    name=production

master:~$ kubectl config current-context

kubernetes-admin@kubernetes

thomas@master:~$ kubectl config set-context prod --namespace=production --cluster=kubernetes-admin@kubernetes
Context "prod" created.

thomas@master:~$ kubectl config use-context dev
Switched to context "dev".
thomas@master:~$ kubectl config current-context
dev



sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
sudo apt install nvidia-387

  



