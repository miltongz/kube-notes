# Application Life Cycle Management
___
#### Rolling updates and rollbacks
- when creating a deployment, a rollout is created and revision starts
- when container version is updated, a new rollout is made and revision is increased
- rollout can be viewed with: `kubectl rollout status deployment/myapp-deployment`
- to view history: `kubectl rollout history deployment/myapp-deployment`
- rolling update: take down replicas one by one and update
- updating could be update vesion of docker containers, update lables, replicas, etc.
- Recreate strategy will scale down a replica set to 0 (in order to update) then scales it back up
- Rolling update will scale up and down, one at at time
- You can 'rollback' to a previous version by running: `kubectl rollout undo deployment/myapp-deployment`
____
### Configure environment variables in applications
- EV are added under spec:
```yaml
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
      - containerPort: 8080
    env:
      - name: APP_COLOR
        value: pink
```
- can also use configmap and secrets:
    - for configmap:
    ```yaml
    env:
      - name: APP_COLOR
        valueFrom:
          configMapKeyRef:
    ```
    - for secrets:
    ```yaml
    env:
      - name: APP_COLOR
        valueFrom:
          secretKeyRef:
    ```
___
#### Configuring ConfigMaps
- `kubectl create configMap`
- or make a config-map.yaml:
    ```yaml
    apiVersion: v1
    kind: configMap
    metadata:
      name: app-config

    data:
      APP_COLOR: blue
      APP_MODE: prod
    ```
- then run: `kubectl create -f config-map.yaml`
- inject the configMap to a pod definition file:
    ```yaml
    envFrom:
      - configMapRef:
            name: app-config
    ```
- `kubectl get configmaps`
___
#### Secrets
- `kubectl create secret generic`
- create using a file:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata: 
      name: app-secret
    data: 
      DB_Host: mysql
      DB_User: root
      DB_Password: paswrd
    ```
- values in example above are in plain text and need to be converted to encoded form
- pipe each value to base64 utility:
    - `echo -n 'mysql' | base64` output is: `bXlzcWw=`
    - `echo -n 'root' | base64`
    - `echo -n 'paswrd' | base64`
- use `echo -n 'bXlzcWw=' | base64 --decode` to decode value.
- you can also create a secret by specifying a file:
    ```yaml
    kubectl create secret generic \
        app-secret --from-file=app_secret.properties
    ```
- `kubectl get secrets`
- `kubectl get secret app-secret -o yaml` will display values in yaml format
- you can inject a secret in a pod definition file by adding:
    ```yaml
    envFrom:
      - secretRef:
            name: app-config
    ```
    under `containers:`
- a single env could also be injected:
    ```yaml
    env:
      - name: DB_Password
        valueFrom:
          secretKeyRef:
            name: app-secret
            key: DB_Password
    ```
- or a whole secret as files in a volume:
    ```yaml
    volumes: 
    - name: app-secret-volume
      secret: 
        secretName: app-secret
    ```
- secrets are encoded, not ecrypted
- you should encrypt at rest
- you can use a encryption configuration file
- anyone that can create pods/deployments in the same namespace can access the secrets
- there a third party secret store providers (AWS Provider, Azure, etc.)
___
#### Multi Container PODs
- instead of running a pod for each container of microservices that work together, and there by possibly needing to deploy services, you can run multiple containers in a single pod
- add multiple containers under spec:
    ```yaml
    spec:
      containers:
      - name: simple-web-app
        image: simple-web-app
        ports:
          - containerPort: 8080
      - name: log-agent
        image: log-agent
    ```
___
#### initContainers
- containers under the same pod depend on each other to stay alive: if one of them shuts-down after completing a task or fails, the other container(s) will shut down.
- You can specify initContainers in a pod file:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: myapp-pod
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: busybox:1.28
        command: ['sh', '-c', 'echo The app is running! && sleep 3600']
      initContainers:
      - name: init-myservice
        image: busybox
        command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ; done;']
    ```
- you can configure multiple initContainers to run in sequential order:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: myapp-pod
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: busybox:1.28
        command: ['sh', '-c', 'echo The app is running! && sleep 3600']
      initContainers:
      - name: init-myservice
        image: busybox:1.28
        command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
      - name: init-mydb
        image: busybox:1.28
        command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
    ```
- if any fail, the pod will restart until they all suceed
___
#### Self Healing Applications
- Self healing applications can be achieved by using ReplicaSets and Replication Controllers
- Kubernetes will ensure that the pod is recreated in case of a crash and that is running at all times
- Kubernetes provides support for checking app health within the PODs through Liveness and Readiness Probes