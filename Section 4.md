#### Monitor Cluster Components
- Metrics Server gets metrics from nodes/pods and save them to memory
- Only one metrics server per cluster
- does not store monitoring on disk, only memory
kubelet has cAdvisor or Container Advisor that retrics metrics from pods and exposes them through the kubelet api and makes them available to the Metrics Server
- to install:
    - git clone metrics server repo
    - `kubectl create -f <folder_path>`
- to use:
    - `kubectl top node`    
    - `kubectl top pod`
___
#### Application Logs
- if theyre are multiple containers within a pod, you must specify the container name in the command
- ex: if container in pod is called event-simulator:
    - `kubectl logs -f event-simulator-pod event-simulator`