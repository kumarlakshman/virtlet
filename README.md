# virtlet
For virtlet PoC with Rook Ceph

KubeVirt is a virtual machine management add-on for Kubernetes providing control of VMs as Kubernetes Custom Resources. 

Virtlet, on the other hand is a CRI (Container Runtime Interface) implementation, which means that Kubernetes sees VMs in the same way it sees Docker containers.

Steps:
------

    # sudo apt-get update && sudo apt-get install -y apt-transport-https curl
    # curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    # cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
      deb https://apt.kubernetes.io/ kubernetes-xenial main
      EOF
    # sudo apt-get update
    # sudo apt-get install -y kubelet=1.17.0-00 kubeadm=1.17.0-00 kubectl=1.17.0-00 docker=19.03.4
    # sudo apt-mark hold kubelet kubeadm kubectl docker
    
    
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
    sudo apt-get install docker-ce=18.06.3~ce~3-0~ubuntu  containerd.io


--> swapoff on all nodes

    sudo swapoff -a
    
    sudo kubeadm init--pod-network-cidr=10.244.0.0/16
    sudo mkdir -p $HOME/.kube/
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    kubectl get pods -o wide --all-namespaces
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
    kubectl taint nodes --all node-role.kubernetes.io/master-

if your deployment is single node deployment just untaint the master node
Deploy Kubernetes cluster
1 master and 3 workers
worker nodes attach 1 HHDs with 100G capacity to each worker, these will be used for CEPH OSD drives

    NAME           STATUS   ROLES    AGE    VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
    controlnode    Ready    master   132m   v1.17.0   172.16.1.151   <none>        Ubuntu 16.04.6 LTS   4.15.0-76-generic   docker://19.3.4
    workernode01   Ready    <none>   130m   v1.17.0   172.16.1.115   <none>        Ubuntu 16.04.6 LTS   4.15.0-76-generic   docker://19.3.4
    workernode02   Ready    <none>   130m   v1.17.0   172.16.1.126   <none>        Ubuntu 16.04.6 LTS   4.15.0-76-generic   docker://19.3.4
    workernode03   Ready    <none>   129m   v1.17.0   172.16.1.183   <none>        Ubuntu 16.04.6 LTS   4.15.0-76-generic   docker://19.3.4

make sure the disks you selecting for OSDs on worker node, does not contain any valid partions or signatures, ifso please make cleanup 
     
    wipefs --all /dev/xvdb
                (or)
    fdisk -l
    dmsetup remove /dev/mapper/ceph--
    
--> 
clone the rook-ceph 1.2 git repo
    
    git clone --single-branch --branch release-1.2 https://github.com/rook/rook.git

follow the instructions present @ https://rook.io/docs/rook/v1.2/ceph-quickstart.html and deploy your rook-ceph 
make the needed changes for your HDDs you use.

    cd rook/cluster/examples/kubernetes/ceph
    kubectl create -f common.yaml
    kubectl create -f operator.yaml
    kubectl create -f cluster.yaml (please refer to cluster.yaml from this repo for needed changes.)
    kubectl create -f toolbox.yaml <-- This will bring up the toolbox pod with all needed ceph clients for connecting to 
    ceph cluster.
    kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash <-- for connecting to the tools pod


on re-installation make sure the the config directory being deleted.
    sudo rm -rf /var/lib/rook/

once you have the all pods up, you can access the ceph dashboard, accessing the manager pod. access kubernetes node with port forwarding..

find the service with manager-dashboard, change the service type from ClusterIp to NodePort, on the port which it get maps, login to the host with port tunneling to localhost and access the dashboard from local system.
               ssh -i "ssh-key.pem" -L 32136:localhost:32136 ubuntu@13.233.251.211
               
virtlet Installation
====================

check the pre-reqisites for virtlet installation

    sestatus <-- SELinux Status should be disabled
    
    sudo groupadd docker
    sudo usermod -aG docker ubuntu

    make sure all k8s cluster nodes pingable with hostnames

--> use this document to install the virtlet https://docs.virtlet.cloud/user-guide/real-cluster/

-->
   download and install criproxy
   
    wget https://github.com/Mirantis/criproxy/releases/download/v0.14.0/criproxy_0.14.0_amd64.deb
    dpkg -i criproxy_0.14.0_amd64.deb
    systemctl status criproxy
    
-->
   on worker nodes stop the dockershim and criproxy and kubelet  
   
    sudo systemctl stop dockershim criproxy kubelet
-->

   edit /etc/systemd/system/dockershim.service and the contents of it look like
    
    [Unit]
    Description=kubelet: The Kubernetes Node Agent
    Documentation=https://kubernetes.io/docs/home/

    [Service]
    ExecStart=/usr/bin/kubelet --experimental-dockershim --port 11250
    Restart=always
    StartLimitInterval=0
    RestartSec=10

    [Install]
    WantedBy=multi-user.target
    RequiredBy=criproxy.service
   
   edit /lib/systemd/system/kubelet.service, after edit it will look something like
   
   
    [Unit]
    Description=kubelet: The Kubernetes Node Agent
    Documentation=https://kubernetes.io/docs/home/

    [Service]
    ExecStart=/usr/bin/kubelet --container-runtime=remote --container-runtime-endpoint=unix:///run/criproxy.sock --image-service-endpoint=unix:///run/criproxy.sock --enable-controller-attach-detach=false
    Restart=always
    StartLimitInterval=0
    RestartSec=10

    [Install]
    WantedBy=multi-user.target

--> 
  re-load system deamon 
  
    sudo systemctl daemon-reload
  
-->
   label the nodes 
   
    kubectl label node workernode01 extraRuntime=virtlet
    kubectl label node workernode02 extraRuntime=virtlet
    
    
-->
   accessing ceph dashboard password
   
    kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo

Troubleshooting failures
------------------------

systemctl status dockershim <-- incase criproxy installation fails check the status of dockrshim, get it started and then  start the criproxy again.

sudo systemctl enable criproxy dockershim <-- enable the criproxy/dockershim to start on boot

