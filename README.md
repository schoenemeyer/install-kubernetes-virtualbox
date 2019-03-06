# install-kubernetes-virtualbox
install kubernetes on virtualbox


bridged adapter

sudo apt-get update
sudo apt-get install ssh

check status with
sudo service ssh status

ssh in machine with root

create root password:

sudo passwd root


sudo apt install git

git clone 
https://github.com/schoenemeyer/Kubernetes-GPU-Guide.git

cd ~/Kubernetes-GPU-Guide/scripts

sudo swapoff -a

modify  vi /etc/fstab

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

  



