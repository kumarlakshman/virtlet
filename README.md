# virtlet
For virtlet PoC with Rook Ceph

KubeVirt is a virtual machine management add-on for Kubernetes providing control of VMs as Kubernetes Custom Resources. 

Virtlet, on the other hand is a CRI (Container Runtime Interface) implementation, which means that Kubernetes sees VMs in the same way it sees Docker containers.

Steps:
------
Deploy Kubernetes cluster
1 master and 3 workers
worker nodes attach 1 HHDs with 100G capacity to each worker, these will be used for CEPH OSD drives 

clone the rook-ceph 1.2 git repo
    git clone --single-branch --branch release-1.2 https://github.com/rook/rook.git

follow the instructions present @ https://rook.io/docs/rook/v1.2/ceph-quickstart.html and deploy your rook-ceph 
make the needed changes for your HDDs you use.

--> cd rook/cluster/examples/kubernetes/ceph

--> kubectl create -f common.yaml

--> kubectl create -f operator.yaml

--> kubectl create -f cluster.yaml (please refer to cluster.yaml from this repo for needed changes.)

--> kubectl create -f toolbox.yaml <-- This will bring up the toolbox pod with all needed ceph clients for connecting to 
ceph cluster.

--> kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash <-- for connecting to the tools pod
