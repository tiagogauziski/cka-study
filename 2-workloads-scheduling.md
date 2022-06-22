# Workloads & Scheduling (15%)

## Table of contents
1. [Understand deployments and how to perform rolling upgrade and rollbacks](#understand-deployments-and-how-to-perform-rolling-upgrade-and-rollbacks)
1. [Use ConfigMaps and Secrets to configure applications](#use-configmaps-and-secrets-to-configure-applications)
1. [Know how to scale applications](#know-how-to-scale-applications)
1. [Bonus: Understanding the role of DaemonSets](#bonus-understanding-the-role-of-daemonsets)
1. [Bonus: Understanding the role of StatefulSets](#bonus-understanding-the-role-of-statefulsets)
1. [Understand the primitives used to create robust, self-healing, application deployments](#understand-the-primitives-used-to-create-robust-self-headling-application-deployments)
1. [Understand how resource limits can affect Pod scheduling](#understand-how-resource-limits-can-affect-pod-scheduling)

## Understand deployments and how to perform rolling upgrade and rollbacks
Reference: 
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

<details>
<summary>Solution</summary>

A Deployment provides declarative updates for Pods and ReplicaSets.

You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate, providing the ability to perform rolling upgrades and rollbacks. When you define a Deployment, it creates a new ReplicaSets. Changes to a deployment, it will create a new ReplicaSets and gradually phase out the old one.

- To create a Deployment (nginx-deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  # Create three replicated Pods
  replicas: 3 
  # Defines how the Deployment finds which Pods to manage.
  selector: 
    matchLabels: 
      app: nginx
  # Defines the Pods that will be created as part of this Deployment
  # It folllows the same template as `kind: Pod`
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

- To check the Deployment rollout status, run:
```bash
kubectl rollout status deployment/nginx-deployment

# Output should be similar to:
# Waiting for deployment "nginx-deployment" rollout to finish: 2 of 3 updated replicas are available...
# deployment "nginx-deployment" successfully rolled out
```

- Because every change in a Deployment creates a ReplicaSet, you can get the list of ReplicaSets by executing:
```bash
# Or kubectl get rs
kubectl get replicaset

# Output should look like:
# NAME                         DESIRED   CURRENT   READY   AGE
# nginx-deployment-9456bbbf9   3         3         3       2m6s
```

> Notice that the name of the ReplicaSet is always formatted as `[DeploymentName]-[Random-String]`. The random string is randomly generated and uses the `pod-template-hash` as seed.
> ```bash
> kubectl get pods --show-labels
> 
> # The output should look like:
> # NAME                               READY   STATUS    RESTARTS   AGE   LABELS
> # nginx-deployment-9456bbbf9-5q77q   1/1     Running   0          15m   app=nginx,pod-template-hash=9456bbbf9
> # nginx-deployment-9456bbbf9-99cs6   1/1     Running   0          15m   app=nginx,pod-template-hash=9456bbbf9
> # nginx-deployment-9456bbbf9-dh94m   1/1     Running   0          15m   app=nginx,pod-template-hash=9456bbbf9
> ``` 

### Update a Deployment

One of the advantages of using a Deployment is that it controls rolling upgrades and rolling back changes. When we perform changes to a Deployment, it will create a new ReplicaSet, and gradually increasing the number of Pods replicas on the new one and simultaneously decreasing the Pod replicas on older ReplicasSets.

- To update a Deployment, there are several ways we can achieve that:
```bash
# Imperative command to update the image
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1

# Imperative command to update the number of replicas
kubectl scale --relpicas 5 deployment/nginx-deployment

# We could also update `nginx-deployment.yaml` using vim and update the Deployment definition 
# Note that only changes to .spec.template will trigger a new ReplicaSet
vim nginx-deployment.yaml
kubectl apply -f nginx-deployment.yaml

# It's also possible to directly update the .spec.template using kubectl
kubectl edit deployment nginx-deployment.yaml
```

- Check the rollout status:
```bash
kubectl rollout status deployment/nginx-deployment

# Output similar to:
# deployment "nginx-deployment" successfully rolled out
```

- We can get additional details about the rollout:
```bash
kubectl get deployment nginx-deployment

# Output should look like:
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           18h

# We can check the ReplicaSet (it should have an old and a new)
kubectl get rs

# Output:
# NAME                         DESIRED   CURRENT   READY   AGE
# nginx-deployment-9456bbbf9   0         0         0       18h
# nginx-deployment-ff6655784   3         3         3       20s

# Notice the ReplicaSet random string into the Pods name.
kubectl get pods

# Output:
# NAME                               READY   STATUS    RESTARTS   AGE
# nginx-deployment-ff6655784-9mpt9   1/1     Running   0          7m1s
# nginx-deployment-ff6655784-mxtk8   1/1     Running   0          7m19s
# nginx-deployment-ff6655784-wkhft   1/1     Running   0          7m10s
```

- To get detailed view and history of actions performed to the Deployment:
```bash
kubectl describe deployment nginx-deployment

# Output:
# Name:                   nginx-deployment
# Namespace:              default
# CreationTimestamp:      Sun, 05 Jun 2022 04:59:33 +0000
# Labels:                 app=nginx
# Annotations:            deployment.kubernetes.io/revision: 2
# Selector:               app=nginx
# Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
# StrategyType:           RollingUpdate
# MinReadySeconds:        0
# RollingUpdateStrategy:  25% max unavailable, 25% max surge
# Pod Template:
#   Labels:  app=nginx
#   Containers:
#    nginx:
#     Image:        nginx:1.16.1
#     Port:         80/TCP
#     Host Port:    0/TCP
#     Environment:  <none>
#     Mounts:       <none>
#   Volumes:        <none>
# Conditions:
#   Type           Status  Reason
#   ----           ------  ------
#   Available      True    MinimumReplicasAvailable
#   Progressing    True    NewReplicaSetAvailable
# OldReplicaSets:  <none>
# NewReplicaSet:   nginx-deployment-ff6655784 (3/3 replicas created)
# Events:
#   Type    Reason             Age   From                   Message
#   ----    ------             ----  ----                   -------
#   Normal  ScalingReplicaSet  11m   deployment-controller  Scaled up replica set nginx-deployment-ff6655784 to 1
#   Normal  ScalingReplicaSet  11m   deployment-controller  Scaled down replica set nginx-deployment-9456bbbf9 to 2
#   Normal  ScalingReplicaSet  11m   deployment-controller  Scaled up replica set nginx-deployment-ff6655784 to 2
#   Normal  ScalingReplicaSet  10m   deployment-controller  Scaled down replica set nginx-deployment-9456bbbf9 to 1
#   Normal  ScalingReplicaSet  10m   deployment-controller  Scaled up replica set nginx-deployment-ff6655784 to 3
#   Normal  ScalingReplicaSet  10m   deployment-controller  Scaled down replica set nginx-deployment-9456bbbf9 to 0
```

### Rollback a Deployment

If the Deployment is not working as expected, you can perform a rollback of the Deployment to a previous revision.

- We can check the change history of the Deployment:
```bash
kubectl rollout history deployment/nginx-deployment

# Output
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```

> Note that the CHANGE-CAUSE field is set to `<none>`. 
> When performing changes to Deployment, it will only get recorded if using --record flag on the command, here are some examples:
> ```bash
> kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1 --record
> 
> kubectl apply -f nginx-deployment.yaml --record
> ```
> **Additional note**: the `--record` flag is being deprecated. The alternative is annotating as mentioned below. 
> 
> It's also possible to have a custom message by adding an annotation into the Deployment
> ```bash
> kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1"
> 
> # Output
> # REVISION  CHANGE-CAUSE
> # 1         <none>
> # 2         image updated to 1.16.1
> ```

- To get details from a specific version of Deployment rollout history:
```bash
kubectl rollout history deployment/nginx-deployment --revision=2

# Output
# deployment.apps/nginx-deployment with revision #2
# Pod Template:
#   Labels:       app=nginx
#         pod-template-hash=ff6655784
#   Annotations:  kubernetes.io/change-cause: image updated to 1.16.1
#   Containers:
#    nginx:
#     Image:      nginx:1.16.1
#     Port:       80/TCP
#     Host Port:  0/TCP
#     Environment:        <none>
#     Mounts:     <none>
#   Volumes:      <none>
```

- Suppose that you made a typo while updating the Deployment:
```bash
# Notice the version is incorrect.
kubectl set image deployment/nginx-deployment nginx=nginx:1.161 --record
```

- The rollout should get stuck. You can verify it by checking the rollout status:
```bash
kubectl rollout status deployment/nginx-deployment

# Output:
# Waiting for rollout to finish: 1 out of 3 new replicas have been updated...
```

- We can check the ReplicaSet status:
```bash
kubectl get rs

# Output
# NAME                          DESIRED   CURRENT   READY   AGE
# nginx-deployment-5b4685b9bd   1         1         0       12m
# nginx-deployment-9456bbbf9    0         0         0       19h
# nginx-deployment-ff6655784    3         3         3       56m

# We can check the Pods. Notice the STATUS of the Pods
kubectl get pods

# Output
# NAME                                READY   STATUS             RESTARTS   AGE
# nginx-deployment-5b4685b9bd-kdqtp   0/1     ImagePullBackOff   0          16m
# nginx-deployment-ff6655784-9mpt9    1/1     Running            0          59m
# nginx-deployment-ff6655784-mxtk8    1/1     Running            0          60m
# nginx-deployment-ff6655784-wkhft    1/1     Running            0          59m
```

- We can perform a rollback of the Deployment. First, let's check the history:
```bash
kubectl rollout history deployment/nginx-deployment

# Output
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         image updated to 1.16.1
# 3         kubectl set image deployment/nginx-deployment nginx=nginx:1.161 --record=true
```

- We can now rollback the changes to the previous Deployment revision. There are multiple ways of doing it:
```bash
# Rollback to the previous revision
kubectl rollout undo deployment/nginx-deployment

# Rollback by specifying the revision number. 
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Output:
# deployment.apps/nginx-deployment rolled back
```
</details>

## Use ConfigMaps and Secrets to configure applications
Reference: 
- https://kubernetes.io/docs/concepts/configuration/configmap/
- https://kubernetes.io/docs/concepts/configuration/secret/

<details>
<summary>Solution</summary>

The ConfigMap is an API object that lets you store configuration for other objects to use (such as Pod).  
Unlike most Kubernetes objects that have a `spec`, a ConfigMap has `data` and `binaryData` fields.

### Create a ConfigMap (sample-configmap.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-configmap
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true   
```
### Consume a ConfigMap inside a Pod definition

There are four different ways that we can use a ConfigMap to configure a container inside a Pod:
1. Inside a container command and args
1. Environment Variables for a container
1. Add a file in a read-only volume, for a application to read
1. Write code to run inside the Pod that uses the Kubernetes API to read a ConfigMap

The fourth method means additional work to add code into the application to consume Kubernetes API, but this technique allows you it would allow you to consume ConfigMap from different namespaces.

- Create a Pod and reference the `ConfigMap`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod-configmap
spec:
  containers:
    - name: demo
      image: alpine
      # command: ["sleep", "3600"]
      command: ['sh', '-c', 'while true; do echo "PLAYER_INITIAL_LIVES: $PLAYER_INITIAL_LIVES"; sleep 3600; done']
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES
          valueFrom:
            configMapKeyRef:
              name: sample-configmap 
              key: player_initial_lives
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: sample-configmap
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # You set volumes at the Pod level, then mount them into containers inside that Pod
    - name: config
      configMap:
        # Provide the name of the ConfigMap you want to mount.
        name: sample-configmap
        # An array of keys from the ConfigMap to create as files
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

- Once the Pod is running, we can verify the ConfigMap usage inside it:
```bash
# We can check the value of the environment variable by get the logs from the Pod.
# It should output the value for the environment variable PLAYER_INITIAL_LIVES
kubectl logs sample-pod-configmap

# Output: 
# PLAYER_INITIAL_LIVES: 3

# To verify the ConfigMap being mapped as a volume, we can open the container and 
# run a `ls` and check it's contents
kubectl exec sample-pod-configmap -- ls /config

# Output:
# game.properties
# user-interface.properties

kubectl exec sample-pod-configmap -- cat /config/game.properties

# Output:
# enemy.types=aliens,monsters
# player.maximum-lives=5
```

### Create a Secret (sample-secret.yaml)

Similar to `ConfigMap`, `Secret` also has two properties to store values: `data` and `stringData`. The difference is that values for `data` needs to be base64 encoded, and `stringData` accepts arbitrary strings as values. Internally, it all gets merged into `data`.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: sample-secret
type: Opaque
data:
  password: $(echo -n "test" | base64 -w0)
  username: $(echo -n "tiago" | base64 -w0)
stringData:
  foo: bar
EOF
```

- You can check the `Secret` contents:
```bash
kubectl get secret sample-secret -o yaml

# Output:
# apiVersion: v1
# data:
#   foo: YmFy
#   password: dGVzdA==
#   username: dGlhZ28=
# kind: Secret
# metadata:
# ...
```

> Note that the `stringData` key `foo` was merged into `data` and converted into base64.

- Create a Pod and reference the `Secret` (sample-pod-secret.yaml):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod-secret
spec:
  containers:
    - name: demo
      image: alpine
      command: ['sh', '-c', 'while true; do echo "SECRET_FOO: $SECRET_FOO"; sleep 3600; done']
      env:
      - name: SECRET_FOO
        valueFrom:
          secretKeyRef:
            name: sample-secret
            key: foo
            optional: false # This means that the secret MUST exists, and include the key named `foo`
      volumeMounts:
      - name: secrets
        mountPath: '/etc/secrets'
  volumes:
  - name: secrets
    secret:
      secretName: sample-secret
```

- Similar to ConfigMap, it's possible to check the Secret being used on the Pod:
```bash
# We can check the value of the environment variable by get the logs from the Pod.
kubectl logs sample-pod-secret

# Output: 
# SECRET_FOO: bar

# To verify the Secret being mapped as a volume, we can open the container and 
# run a `ls` and check it's contents
kubectl exec sample-pod-secret -- ls /etc/secrets

# Output:
# game.properties
# user-interface.properties

kubectl exec sample-pod-secret -- cat /etc/secrets/username

# Output:
# tiago

```
</details>

## Know how to scale applications
Reference: 
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment
- https://kubernetes.io/docs/tasks/run-application/scale-stateful-set/

<details>
<summary>Solution</summary>

To scale applications in Kubernetes, you just need to define how many replicas you need, and Kubernetes does the rest for you.

- You can scale it using declarative or imperative commands:
```bash
# Declarative scaling:
# Update the `replicas` field with the new value
vim nginx-deployment.yaml

# Apply the changes using kubectl
kubectl apply -f nginx-deployment.yaml
# Output:
# deployment.apps/nginx-deployment configured

# You can also edit the deployment directly using kubectl edit
kubectl edit deployment nginx-deployment
# Output:
# deployment.apps/nginx-deployment edited

# Imperative scaling:
kubectl scale deployment nginx-deployment --replicas 5
# Output:
# deployment.apps/nginx-deployment scaled
```

The same way we scale Deployments, it will also work for:
- StatefulSets
- ReplicaSets (not being controlled by a Deployment) 
</details>

## Bonus: Understanding the role of DaemonSets
Reference: 
- https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/

<details>
<summary>Solution</summary>

DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to the new nodes. When nodes are removed, the Pods are also removed.

Some typical uses of a DaemonSet are:
- Running a cluster storage daemon on every node.
- Running a log collection daemon on every node.
- Running a node monitoring daemon on every node.

> Note that if some CNI plugins also use DaemonSets to enable networking on all nodes of the cluster.
> If you are using the default flannel configuration, you should see a DaemonSet being used:
> ```bash
> kubectl get daemonset -n kube-system
> 
> # Output:
> # NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
> # kube-flannel-ds   5         5         5       5            5           <none>                   29d
> # kube-proxy        5         5         5       5            5           kubernetes.io/os=linux   29d
> ```

- Create a DaemonSet (sample-daemonset.yaml)
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

- Run `kubectl create` to apply create the resource:
```bash
kubectl create -f sample-daemonset.yaml

# Output:
# daemonset.apps/fluentd-elasticsearch created

# We can see the `fluentd-elasticsearch` DaemonSet running
kubectl get daemonset -n kube-system

# Output
# NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
# fluentd-elasticsearch   5         5         5       5            5           <none>                   84s
# kube-flannel-ds         5         5         5       5            5           <none>                   29d
# kube-proxy              5         5         5       5            5           kubernetes.io/os=linux   29d

# We can also check the Pods running for each node
kubectl get pods -n kube-system -o wide

# Output:
# NAME                                  READY   STATUS    RESTARTS       AGE     IP             NODE            NOMINATED NODE   READINESS GATES
# ...
# fluentd-elasticsearch-lvvt8             1/1     Running   0              2m48s   10.244.0.27    k8s-control     <none>           <none>
# fluentd-elasticsearch-6wmtd             1/1     Running   0              2m48s   10.244.1.3     k8s-control-2   <none>           <none>
# fluentd-elasticsearch-xd245             1/1     Running   0              2m48s   10.244.4.3     k8s-control-3   <none>           <none>
# fluentd-elasticsearch-xs67t             1/1     Running   0              2m48s   10.244.2.51    k8s-worker1     <none>           <none>
# fluentd-elasticsearch-hjm7g             1/1     Running   0              2m48s   10.244.3.52    k8s-worker2     <none>           <none>
# ...
```

</details>

## Bonus: Understanding the role of StatefulSets
Reference: 
- https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/

<details>
<summary>Solution</summary>

StatefulSets are useful to be able to scale stateful applications. 

> StatefulSets require a Headless Service to be responsible for the network identity of the Pods. The service needs to be created beforehand.

- Create a StatefulSet (sample-statefulset.yaml)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: statefulset-pv-1
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/pv-1"
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: statefulset-pv-2
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/pv-2"
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: statefulset-pv-3
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/pv-3"
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: statefulset-service
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sample-statefulset
  labels:
    app: statefulset
spec:
  replicas: 3
  serviceName: nginx-service
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 10
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
```

> The manifest above define 3 PersistentVolumes manually because our cluster is not configured to use dynamic provisioning. 
> Because the the StatefulSet define a `volumeClaimTemplate`, it requires a PersistentVolume to exist, otherwise it won't create the Pods.

- We can check the Kubernetes resources created as part of the StatefulSet
```bash
# Notice the Pod names are created have a ordinal index (1, 2, 3, N)
kubectl get pods

# Output:
# NAME                   READY   STATUS    RESTARTS   AGE
# sample-statefulset-0   1/1     Running   0          112s
# sample-statefulset-1   1/1     Running   0          98s
# sample-statefulset-2   1/1     Running   0          78s

# Each Pod also created it's own PersistentVolumeClaim
kubectl get pvc

# Output
# NAME                       STATUS   VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# www-sample-statefulset-0   Bound    statefulset-pv-1   100Mi      RWO                           20h
# www-sample-statefulset-1   Bound    statefulset-pv-3   100Mi      RWO                           20h
# www-sample-statefulset-2   Bound    statefulset-pv-2   100Mi      RWO                           20h
```

- StatefulSet also creates stable network identifiers
```bash
# Lets run a busybox container and run nslookup to check the DNS entries available
kubectl run -i --tty --image busybox:1.28 dns-test --restart=Never --rm

# Output:
#  # nslookup nginx-service
# Server:    10.96.0.10
# Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
# 
# Name:      nginx-service
# Address 1: 10.244.2.59 sample-statefulset-1.nginx-service.default.svc.cluster.local
# Address 2: 10.244.3.68 sample-statefulset-0.nginx-service.default.svc.cluster.local
# Address 3: 10.244.3.69 sample-statefulset-2.nginx-service.default.svc.cluster.local
```

</details>

## Understand the primitives used to create robust, self-healing, application deployments
Reference: 
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

<details>
<summary>Solution</summary>

Once you create a Deployment, StatefulSet or a DaemonSet, you want to make sure the Pods are resiliant in case of a failure on the application on other downstream components.

Pods allow us to define probes on running containers to assess their health:
- `livenessProbe`  
Liveness probes allow you to customise the default detection mechanism and make it more sophisticated.    
By default, Kubernetes will only consider a container to "down" and apply the restart policy if the container process stops.
> By default, Kubernetes will decide whether to restart the container based on the status of container's PID 1 process.  
> The first process to run on a container assumes PID 1. 

- `readinessProbe`
Indicates whether the container is ready to respond to requests. If the readiness probe fails, the Endpoint controller (related to Services) removes the Pod's IP address from the endpoints of all Services that match the Pod.
The default state of readiness before the initial delay is `Failure`. If a container does not provide a readiness probe, the default state is `Success`.

- `startupProbe`
Indicates whether the aplication within the container is started. All other probes are disabled if a startup probe is provided, until it succeeds. If the startup probe fails, the kubelet kill the container, and the container is subjected to it's restart policy. 
> Similar to `lievenessProbe`, however, while liveness probe run constantly, startup probes run at the container startup and stop running once it succeed.  
> Useful for legacy applications with long startup times.

### Investigate the default `livenessProbe` behavior

We can investigate the default liveness probe behavior by running a default nginx container, verify who is PID 1, kill the process and check what happens.

- Create a nginx Pod
```bash
# Lets use an imperative command to create the Pod
kubectl run nginx --image=nginx
```

- Check the Pod has been created
```bash
kubectl get pods

# Output:
# NAME    READY   STATUS    RESTARTS   AGE
# nginx   1/1     Running   0          31s

# Note the RESTARTS is set to 0.
```

- Lets run a bash command inside the container.
```bash
kubectl exec nginx -i -t -- bash

# List all process
ls -l /proc/*/exe

# Output:
# [...]
# lrwxrwxrwx 1 root  root  0 Jun 18 05:35 /proc/1/exe -> /usr/sbin/nginx
# lrwxrwxrwx 1 nginx nginx 0 Jun 18 05:35 /proc/31/exe
# [...]

# Notice that nginx process is PID 1
# Let's kill the process and check what happens
kill 1

# Output:
# root@nginx:/# command terminated with exit code 137
```

- Check again the Pod list
```bash
kubectl get pods

# Output:
# NAME    READY   STATUS    RESTARTS     AGE
# nginx   1/1     Running   1 (4s ago)   8m12s
```

### Configure a `livenessProbe` with `exec` command

- Create a Pod with a `livenessProbe` (sample-pod-livenessprobe.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: sample-pod-livenessprobe
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

The definition above describes a Pod that creates a file located at `/tmp/healthy` and sleeps for 30 seconds. After that, it removes the file and sleeps for 10 minutes.  
Because the `livenessProbe` has been configured to run `cat /tmp/healthy`, after 30 seconds, the file will be deleted and the probe should fail, because the file does not exits anymore.

- Within 35 seconds, we can check the Pod events, the probe has not failed yet:
```bash
kubectl describe pod sample-pod-livenessprobe

# Output:
# Events:
#   Type    Reason     Age   From               Message
#    ----    ------     ----  ----               -------
#   Normal  Scheduled  8s    default-scheduler  Successfully assigned default/sample-pod-livenessprobe to k8s-worker2
#   Normal  Pulling    7s    kubelet            Pulling image "k8s.gcr.io/busybox"
#   Normal  Pulled     5s    kubelet            Successfully pulled image "k8s.gcr.io/busybox" in 1.99976877s
#   Normal  Created    5s    kubelet            Created container liveness
#   Normal  Started    5s    kubelet            Started container liveness
```

- After 35 seconds, we should be able to see 
```bash

# Output:
# Events:
#   Type     Reason     Age                From               Message
#   ----     ------     ----               ----               -------
#   Normal   Scheduled  76s                default-scheduler  Successfully assigned default/sample-pod-livenessprobe to k8s-worker2
#   Normal   Pulled     73s                kubelet            Successfully pulled image "k8s.gcr.io/busybox" in 1.99976877s
#   Normal   Created    73s                kubelet            Created container liveness
#   Normal   Started    73s                kubelet            Started container liveness
#   Warning  Unhealthy  31s (x3 over 41s)  kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
#   Normal   Killing    31s                kubelet            Container liveness failed liveness probe, will be restarted
#   Normal   Pulling    1s (x2 over 75s)   kubelet            Pulling image "k8s.gcr.io/busybox"
```

> Note that the `livenessProbe` execute 3x before killing the container. That is because the default `failureThreshold` value is 3. [Source](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes)

### Configure a `startupProbe` with `exec` command

- Create a Pod with a `startupProbe` (sample-pod-startupprobe.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: startupprobe
  name: sample-pod-startupprobe
spec:
  containers:
  - name: startup
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      periodSeconds: 10
    startupProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      failureThreshold: 30
      periodSeconds: 10
```

Similar to the previous `livenessProbe` example, when using `startupProbe`, it allows slow applications to startup with different timeout/threshold requirements than the `livenessProbe`. Once `startupProbe` succeeds, `livenessProbe` takes over.

### Configure a `readinessProbe` with `exec` command

- Create a Pod with a `readinessProbe` (sample-pod-readinessProbe.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: readinessProbe
  name: sample-pod-readinessProbe
spec:
  containers:
  - name: startup
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - sleep 30; touch /tmp/healthy; sleep 1200
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      periodSeconds: 10
```
`readinessProbe` runs in parallel with `livenessProbe` during the entire lifecycle of the container.  
Main goal for `readinessProbe` is to ensure that traffic does not reach the container that is not ready for it. 

</details>

## Understand how resource limits can affect Pod scheduling
Reference: 
- https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/
- https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/


<details>
<summary>Solution</summary>

When creating a Pod, you can optionally specify how much of each resource a container needs.

There are two ways to specify resource usage:
- `request`
The `request` is used when the scheduler is deciding which Node the Pod should be allocated. If all available Nodes does not have enough resources (less than requested), the Pod does not get allocated, and remain with `Pending` status.  
The Pod is allowed to use more resources than initially requested, except when the `limit` is provided. 
- `limit`
When a `limit` is specified, the kubelet enforces those limits so that the running container is not alloed to use more of that resource than the limit set.  
Limits can be implemented either reactively (the sistem intervenes once it sees a violation) or by enforcement (the system prevents the container from ever exceeding the limit). Different runtimes can have different ways to implement the same restrictions. 

### Specify a CPU request

</details>