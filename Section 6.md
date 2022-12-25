#### OS Upgrades
- If a pod goes down for more than 5 minutes, the pod will be terminated from the node (this is called the time-eviction-timeout)
- you can set the time with: `kube-controller-manager --pod-eviction-timeout=5m0s
- if a pod is not part of a replicaSet, it will not be recreated on another node
- you can drain a node with `kubectl drain node-1` in order terminate services on that node and recreate them in another
- the node will be marked as unschedulable
- `kubectl uncordon node-1` will allow pods to be sheduled again
`kubectl cordon node-2` will make node unschedulable for further pods; will note delete current pods on node
___
#### Software Versions
- v1.11.3:
    - v1 is major version
    - .11 is minor version: features and functionalities
    - .3 is patch: bug fixes
#### Cluster Upgrade Process
- Refresh of components:
    - kube-apiserver
    - kubelet
    - controller-manager
    - etcd cluster
    - kubectl
    - Core DNS
    - kube-scheduler
    - kube-proxy
- components can have different versions
- none can have a version higher than the kube-apiserver, except for kubectl
- recommended to upgrade to one minor version at a time
- you can use tools like kubeadm to upgrade cluster:
    - `kubeadm upgrade plan`
    - `kubeadm upgrade apply`
- when upgrading, master and management functionalities go down first; worker nodes and services are still running
- for upgrading worker nodes, 3 strategies:
    - upgrade all at once
    - move workloards of one node to others, upgrade that node, then move workloads back
    - add new nodes with newer version
- kubeadm tool must be upgraded before upgradin cluster:
    - `apt-get upgrade -y kubeadm=1.12.0-00`
    - then `kubeadmn upgrade apply v1.12.0`
- `kubectl get node` will show version of kubelet and not version of kube-apiserver
- next is to upgrade kubelets:
    - `kubectl drain node-1`
    - `apt-get upgrade -y kubelet=1.12.0-00`
    - `kubeadm upgrade node config --kubelet version v1.12.0`
    - `systemctl restart kubelet`
    - `kubectl uncordon node-1`
___
#### Backup and Restore Methods
- Potential candidates to backup:
    - resource configuration (deployment, pods, services defintion files)
    - etcd cluster (where all cluster related information is stored)
    - persistent volumes
- if someone created an object the imperative way (using a command), you can query the kube-apiserver
- you can create a backup-scipt that can query the server and output as yaml files:
    - for namespaces: `kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml`
- tools like "Velero" can also achieve this
- etcd cluster is hosted on the master nodes
- you can take a snapshot of the etcd database:
    ```yaml
    ETCDCTL_API=3 etcdctl \
        snapshot save snapshot.db
    ```
- to specify a location:
    ```yaml
    ETCDCTL_API=3 etcdctl \
        snapshot save snapshot.db \

    ```
- `snapshot status snapshot.db`
- to restore:
    - `service kube-apiserver stop`
    - ```yaml
      ETCDCTL_API=3 etcdctl \
      snapshot restore snapshot.db \
      --data-dir /var/lib/etcd-from-backup
      ```
      - then configure etcd.service to use new data directory specified in restore
      - `systemctl daemon-reload`
      - `system etcd restart`
      - `service kube-apiserver start`