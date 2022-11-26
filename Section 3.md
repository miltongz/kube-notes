#### Scheduler
    - Finds what is the best node for new pods to run on.
    - If pod is not created yet (pod-definition.yaml file has not been run yet), you can manually set the node name in the file.
- *kubectl get pods -n kube-system -> list pods in the kube-system namespace*
- *kubectl replace --force -f nginx.yaml -> replaces pod
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