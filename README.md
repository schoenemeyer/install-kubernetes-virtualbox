# Install Kubernetes on a single node with Virtualbox

This recipe will lead you through the installation of Kubernetes on a single Workstation running VirtualBox.
I created two VMs (master and slave with Ubuntu 16.04.05.
The master was created with 2 cores and 2GB RAM, the slave with 1 core and 1 GB.
I used two network adapter (NAT and host-only)
For detailed instructions visit https://medium.com/@KevinHoffman/building-a-kubernetes-cluster-in-virtualbox-with-ubuntu-22cd338846dd

Start with 
```
sudo apt-get install ssh
```
check ssh status with
```
sudo service ssh status
```
ssh in the VM and create root password and do some basic installations:
```
sudo passwd root
sudo apt install git curl
```
ssh in the VM from the host and do some modifications, it is much more comfortable than inside Virtualbox terminal.
```
sudo swapoff -a
```
modify  vi /etc/fstab accordingly

```
sudo apt-get update
sudo apt-get upgrade
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

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
If you don’t want to preface the docker command with sudo, create a Unix group called docker and add users to it. When the Docker daemon starts, it creates a Unix socket accessible by members of the docker group.
```
sudo groupadd docker
sudo usermod -aG docker $USER
```
doublecheck whether docker daemon is running.
```   
sudo docker run hello-world
``` 
If this is fine, then continue with the Kubernetes installation. The first step is to grab the key for the Kubernetes install.
```  
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add
```  
and create a file called kubernetes.list with 
```  
touch /etc/apt/sources.list.d/kubernetes.list
```
and insert the following line 
```
deb http://apt.kubernetes.io/ kubernetes-xenial main    
```
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
Before we initialize Kubernetes, we should think of the network plugin to deploy later.
Options are Multus, Calico and others. 
Pick your network plugin before you run kubeadm init.
```
git clone https://github.com/intel/multus-cni.git
```



We can now initialize Kubernetes by running the initialization command and passing --pod-network-cidr which is required for Flannel to work correctly
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=xxx.xxx.xxx.xx --kubernetes-version stable-1.13

```
You must replace --apiserver-advertise-address with the IP of your host.
It is important to save the line with the kubeadm join command (especially the token)!

it is useful to add an separate account with superuser privileges
```
sudo useradd tom -G sudo -m -s /bin/bash
sudo passwd tom
```
Configure environmental variables as the new user and switch into the new user account 
```
su tom
cd $HOME
sudo whoami
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
echo "export KUBECONFIG=$HOME/admin.conf" | tee -a ~/.bashrc
```
Once Kubernetes has been initialized we then install the Flannel Pod Network by running
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
```
or with Multus CNI 
```

cat ./images/{multus-daemonset.yml,flannel-daemonset.yml} | kubectl apply -f -

```
Finally you run 
```
kubectl get nodes
tom@master:~$ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   41m   v1.13.4

```
Now you should see all the Pods with namespaces 
```
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
master   Ready   master   29m   v1.13.4
thomas@master:~$ 
```

Usually you have workernodes in your cluster, but if you want to run Pods on the master node you can do by untaint the master node with 
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```
Check the taint status
```
thomas@master:~$ kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
master   Ready      master   14d   v1.13.4
```
Now look into the details of the node called master
```
kubectl describe nodes master 
```
or just

```
thomas@master:~$ kubectl describe nodes master | grep -i taints
```

If you look for taints, you will see "<none>" if the master is untained and you can place jobs there.

```    
    Taints:             <none>
```    
if you see the "NoSchedule", then no jobs will be placed on the master node.    

```    
    Taints:             node-role.kubernetes.io/master:NoSchedule 
    
```


Now you can add any number of machines by running the following on each node as root
```
sudo kubeadm join xxx.xxx.xxx.xx:6443 --token r7uxxxxxxxxxx --discovery-token-ca-cert-hash sha256:xxxxxxx
```
You will get this:
```

tom@master:~$ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   75m   v1.13.4
worker   Ready    <none>   84s   v1.13.4
tom@master:~$ 
```
kubeadm join command do not explicitly label the role of node
You can add this yourself with kubectl label
```
kubectl get node --show-labels
kubectl label node worker node-role.kubernetes.io/worker=
tom@master:~$ kubectl get node 
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   92m   v1.13.4
worker   Ready    worker   18m   v1.13.4
tom@master:~$ 
```

Now you are ready to start a simple example
```

kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```
Display information about the Deployment:
```
kubectl describe deployment nginx-deployment
```
The output is similar to this:
```
tom@master:~$ kubectl describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Thu, 07 Mar 2019 10:39:43 +0100
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},"spec":{"replica...
Selector:               app=nginx
Replicas:               2 desired | 2 updated | 2 total | 0 available | 2 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.7.9
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-76bf4969df (2/2 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  106s  deployment-controller  Scaled up replica set nginx-deployment-76bf4969df to 2

```
List the pods created by the deployment:
```
kubectl get pods -l app=nginx
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-76bf4969df-pjmfq   0/1     ContainerCreating   0          3m13s
nginx-deployment-76bf4969df-sqq8g   0/1     ContainerCreating   0          3m13s

```
After a short time the Status will change into Running State.

```
tom@master:~$ kubectl apply -f https://k8s.io/examples/application/deployment-update.yaml
deployment.apps/nginx-deployment configured
tom@master:~$ 

```
Cleaning up pods:
```
kubectl delete pod nginx-deployment-76bf4969df-sqq8g
```

Lets look at the next Example. The job config below computes π to 2000 places and prints it out. It takes around 10s to complete
```
kubectl create -f https://k8s.io/examples/controllers/job.yaml
tom@master:~$ kubectl get pods
NAME                                READY   STATUS              RESTARTS   AGE
pi-dkj28                            0/1     ContainerCreating   0          74s
tom@master:~$ 
tom@master:~$ kubectl get pods
NAME         READY   STATUS      RESTARTS   AGE
pi-dkj28     0/1     Completed   0          25m
tom@master:~$ 

```
Check on the status of the Job with kubectl:
```
kubectl describe jobs/pi
```

View the standard output of one of the pods:
```
kubectl logs pi-dkj28

3.141592653589793238462643383279502884197169399375105820974944592307816......
```

If you ever have problems you can always delete your kubernetes install use kubeadm reset command. this will un-configure the kubernetes cluster.
```
kubeadm reset 
```
login as root and reset iptables
```
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```
to completely remove and clean Kubernetes (installed with "apt-get"):
```
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube*   
sudo apt-get autoremove  
sudo rm -rf ~/.kube
```

Access the Dashboard

Create Service Account with name admin-user in namespace kube-system first by creating the file dashboard-adminuser.yaml with the content below:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```
then we use kubectl apply -f dashboard-adminuser.yaml to create the service account

Now we create ClusterRoleBinding for our ServiceAccount.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```
you can now get the token for the dashboard by this command

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
Then we start 
```
kubectl proxy 
```
from the local workstation (e.g. master)

the dashboard will be available under http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/


Now we create a (hostPath) PersistentVolume, that be shared by all pods. Kubernetes supports hostPath for development and testing on a single-node cluster. It uses a file or directory on the Node to emulate network-attached storage

In your shell of any node in the cluster , create a /mnt/data directory and write a file

```
echo 'Hello from Kubernetes storage' > /mnt/data/index.html
```


In a production cluster, you would not use hostPath. Instead a cluster administrator would provision a network resource like a persistent disk on-prem or in the cloud. 

Get the config file 
https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/storage/pv-volume.yaml

and modify the configuration file for the hostPath PersistentVolume if necessary (e.g. size)

```
pods/storage/pv-volume.yaml 
kind: PersistentVolume
apiVersion: v1
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```
Now create the PersistentVolume:


```
kubectl apply -f pv-volume.yaml

```
View the PersistentVolume
```
kubectl get pv task-pv-volume
```
Now create the Volume Claim

wet https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/storage/pv-claim.yaml
Modify it to your needs and run apply

```
kubectl apply -f pv-claim.yaml

kubectl get pv task-pv-volume

```
you should something like this
```
kube@kubemaster:~$ kubectl get pv task-pv-volume
NAME             CAPACITY        ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
task-pv-volume   107374182400m   RWO            Retain           Bound    default/task-pv-claim   manual                  4m21s
kube@kubemaster:~$ 
```
or the dashboard view

<img src="https://github.com/schoenemeyer/install-kubernetes-virtualbox/blob/master/figures/Screenshot%20at%202019-05-08%2017-28-16.png" width="652">

Just in case you this error:

```
error: unable to recognize "./pv-volume.yaml": Get http://localhost:8080/api?timeout=32s: dial tcp 127.0.0.1:8080: connect: connection refused
```

curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops


https://github.com/kubernetes/kops.git

then
```
# Find your cluster name
kops get clusters
# set the clustername as a var
clustername=<clustername>
# export the KUBECONFIG variable, which kubectl uses to find the kubeconfig file
export KUBECONFIG=~/.kube/${clustername}
# download the kubeconfig file locally using kops
kops export kubecfg --name ${clustername} --config=~$KUBECONFIG
```    


Other Basic commands

kubectl config view
kubectl describe pods



priority of pods Pod Priority and Preemption
https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/

kubectl apply -f  priority.yaml
```
apiVersion: scheduling.k8s.io/v1beta1
kind: PriorityClass
metadata:
  name: hig-priority
value: 1000000
globalDefault: false
description: "Use this class for XYZ service pods only."
```
kubectl get priorityclasses
```
tom@master:~$ kubectl get priorityclasses
NAME                      VALUE        GLOBAL-DEFAULT   AGE
hig-priority              1000000      false            3s
system-cluster-critical   2000000000   false            15d
system-node-critical      2000001000   false            15d
tom@master:~$ 
```

Testing:
The higher the value, the higher the priority. 

## Assigning priority to your pods


Create Deployment


Delete all kube-system namespaces

kubectl delete --all pods --namespace=kube-system
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl get nodes -o wide

kubectl get pods --all-namespaces
kubectl get pods --all-namespaces
kubectl describe  po  pod1



Another way to deploy Kubernetes productive cluster

git clone kubespray

pip3 install ruamel.yaml

declare -a IPS= .....

from master
cd .ssh
ssh-keygen -t rsa -b 4096
ssh-copy-id user@serverip 




