#### Scheduler
    - Finds what is the best node for new pods to run on.
    - If pod is not created yet (pod-definition.yaml file has not been run yet), you can manually set the node name in the file.
- *kubectl get pods -n kube-system -> list pods in the kube-system namespace*
- *kubectl replace --force -f nginx.yaml -> replaces pod*
- *kubectl  get pods --watch -> watch the output change*
___
#### Labels
- Labels are added in a key-value format under metadata:
```yaml
metadata:
    name: simple-webapp
    labels:
        app: App1
        function: Front-end
```
    - kubectl get pods --selector app=App1
    - kubectl get pod --selector env=prod,bu=finance,tier=frontend
___
#### Taints & Tolerations
      - are used to set restrictions on what pods get sheduled on the nodes
- *kubectl taint nodes node-name key-value:taint-effect*
    - 3 taint-effects
      - NoSchedule: No pods will be schedule on the node 
      - PreferNoSchedule: Scheduler will try not to schedule pods
      - NoExecute: Any existing pods will be evicted if they can't tolerate the new taint
  - *kubectl taint nodes node1 app=blue:NoSchedule*
    - Tolerations can be added pod definition files:
```yaml
spec:
    containers:
    - name: nginx-container
      image: nginx
    tolerations:
    - key: "app"
      operator: "Equal"
      value: "Blue"
      effect: "NoSchedule"
```
        - kubectl describe node kubemaster | grep Taint
        - master node by default has a NoSchedule taint
        - remove taint with "-" at the end
          - ex: kubectl taint node controlplane node-role.kubernetes.io/control-plane:NoSchedule-
___
#### Node Selectors
- you can label nodes to then specify in pods what nodes they can go on
- in node: *kubectl label nodes <node-name> <label-key=label-value*
- ex: *kubectl label nodes node01 size=Large*
- then in pod under spec add:
```yaml
spec:
    nodeSelector:
        size: Large
```

#### Node Affinity
- similar to Node Selector, but allows more refinement
- can specify more than one value compared to Node selectors (by adding more than one under `values`):
```yaml
spec:
 affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
       - key: size
         operator: In
         values:
         - Large
         - Medium
```
or
```yaml
- key: size
  operator: NotIn
  values:
  - Small
```
- the `exists` operator if the label `size` on the nodes
- there are more operators
- 2 types of node affinity:            
    - `requiredDuringSchedulingIgnoredDuringExecution:`
    - `preferredDuring...`
- Scheduling is the state where a pod does not exists and is created for the first time
- Execution is when a pod already exists and change has been made in the environment it resides
- if `IgnoredDuringExecution` is expressed, the pod will not be impacted, however `Required` will and affected pod to be evicted`
___
Ex:
- Add a label to a node:
    - `kubectl label node node01 color=blue`
- Check for taints:
    - `kubectl describe node node01 | grep Taints`
- Edit a deployment:
    - `kubectl edit deployment blue`
___
Ex:
1. Create a deployment using `kubectl` and output to yaml file:
    - `kubectl create deployment red --image=nginx --replicas=2 --dry-run=client -o yaml > red.yaml`
2. Add the label `node-role.kubernetes.io/master` to the previous deployment:
```yaml
spec:
  affinity:
   nodeAffinity:
     requiredDuringSchedulingIgnoredDuringExecution:
       nodeSelectorTerms:
       - matchExpressions:
        - key: node-role.kubernetes.io/master
         operator: Exists
```
3. Then `kubectl create -f red.yaml`
____
#### Node Affinity vs Taints & Tolerations
- Taints & Tolerations does not guarantee that the pods will only preferred the nodes that are desired (there is not guarantee that a pod with "red" toleration will not end in a node without "red" taint).
- node selector ties the pods to the correct nodes, but does not guarantee that other pods without node affinity will end up on undesired nodes.
- we can use a combination of taints & tolerations with node affinity
____
#### Resources Requirements and Limits
- Default requirement 0.5 CPU and 256 Mi 
- If pod needs more than the default, resources can be specified under in `spec` under each container:
```yaml
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
      - containerPort: 8080
    resources:
      requests:
        memory: "1Gi"
        cpu: 1
```
- In docker, containers have no limit to the resources it can consume on a node.
- You can set default limits by creating a namespace (ex: `kubectl create namespace default-mem-example`) and `limitRange` file:
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```
- Then apply the `limitRange` file to the specified namespace:
`kubectl apply -f memory-defaults.yaml --namespace=default-mem-example`
- If you know a container may need more, you need to specify more resource usage, as well as specifying limits:
```yaml
resources:
      requests:
        memory: "1Gi"
        cpu: 1
      limits:
        memory: "2Gi"
        cpu: 2
```
- If a container attempts to use more cpu, kubernetes will throttle the cpu.
- If a container attempts to use more memory constantly, kubernetes will terminate the pod.
- `OOMkilled` status means that pod does not have enough memory.
___
#### Editing pods and deployments
- You cannot edit a running pod; the pod will eventually have to be deleted and recreated. You can still edit the active definition file, which will throw an craete a new temporary file, edit that file, and then run the `kubectl replace force -f <filename>` to recreate the pod.
- You can easily edit any deployment and in turn will make changes to any child definition (pod specifications for example). A change to a deployment will delete and recreate a pod if need be.
___
#### DaemonSets
- DaemonSets are the same as ReplicaSets, however they ensure that one copy of a pod exists in each node. If a new node is added, a copy of the pod will automatically be added to the new node.
- DaemonSets can be useful for pods that need to run on every pod (networking pods for example)
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      name: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
      - name: monitoring-agent
        image: monitoring-agent
```
- to get information, run `kubectl get daemonsets`
- daemonsets uses the default scheduler and node affinity rules to schedule pods on nodes.
- `kubectl get daemonsets -A` can specify all daemonsets in all namespaces
- or specify a daemonset and namespace:
`kubectl get daemontset <daemonset> -n <namespace>`
- there is no `kubectl` cli command to create a daemonset, but you can create a deployment dry-run and edit it appropiately:
`kubectl create deployment elastisearch -n kube-system --image=k8s.gcr.io/elastisearch --dry-run=client -o yaml > elasti.yaml`
- in new file:
    - change kind to 'DaemonSet'
    - remove `replicas: 1`, `strategy:`, and `status: {}`
- then run: `kubectl create -f elasti.yaml`
___
#### Static Pods
- kubelet on every node and relies on the kube api-server on what to load; which were decided by the kube scheduler
- kube scheduler is stored in the etcd cluster (data store) 
- kubelet can manage node independently
- pods created under these conditions with solely the kubelet are know as static pods.
- pod defintion files can be stored under `/etc/kubernetes/manifests` for example and the kubelet can be made to watch this directory
- can edit the `kubeconfig.yaml` file to specify the `staticPodPath:` (can check the `kubelet.service` file as well)
- you can use static pods to deploy componenets of the controlplane itself. Create definition files for each component and store in the preferred manifest folder path:
    - controller-manager.yaml
    - apiserver.yaml
    - etcd.yaml
- by deploying the components as pods, you don't have to worry about downloading binaries or the services crashing, since as static pods they will automatically be created whene necessary.
- kube admin tool sets up the components this. `kubectl get pods -n kube-system` will show the components being deployed as pods
- when `kubectl get pods -A`, static pods will generally their node name appended to the end (ex: `kube-apiserver-controlplane` or `etcd-controlplane`)
- you can also check the yaml file of the pod; information located under "ownerReferences":
```yaml
ownerReferences:
- apiVersion: v1
  controller: true
  kind: Node
  name: controlplane
```
- static pod path under kubelet config path: `/var/lib/kubelet/config.yaml` under `staticPodPath`
#### Multiple Schedulers
- kubernetes can be instructed to have a pod scheduled by a specific scheduler
- you can have multiple scheduler config files:
`myscheduler-config.yaml`
```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: kubeSchedulerConfiguration
profiles:
- schedulerName: my-scheduler
```
- a config can be specified to select a scheduler "leader" since only scheduler can be active a time (look into 'leaderElection' for more info)
- in a pod definition file, you can speciy a custom scheduler under `spec:` and add a new item called `schedulerName:`
- running `kubectl get events -o wide` will let you view the "source" of an event. In this case "my-custom-scheduler" can be the source.
- could also view the logs of the scheduler with `kubectl    log my-custom-scheduler --name-space=kube-system`
___
#### Scheduler Profiles
- pods have priorities specified under `spec`:
```yaml
spec:
  priorityClassName: high-priority
```
- A priority class first needs to be made:
```yaml
apiVersion: v1
kind: PriorityClass
metadata:
  name: high-priority
value: 10000000
globalDefault: false
description: "This priority class should be used for xyz service pods only."
```
- this allow pods with high-priority got to the beginning of the scheduler queue
- after this scheduling phase, the pod enters the filter phase:
    - nodes that don't have enought resources for the pods are filtered out
- after comes the scoring phase; nodes are given different weights:
    - the node with more resources left after scheduling a pod will get a higher score and chosen for the node to be scheduled
- finally, in the binding phase, a pod is bound to the node with the highest score
- scheduling plugins are used to achieve each phase
- custom plugins can be made and plugged into the "extension points" of each phase
- instead creating multiple schedulers which can cause an issue that one scheduler may not know that another one is scheduling, you can create multiple profiles in a single scheduler config