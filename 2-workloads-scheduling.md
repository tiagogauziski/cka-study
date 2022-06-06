# Workloads & Scheduling (15%)

## Table of contents
1. [Understand deployments and how to perform rolling upgrade and rollbacks](#understand-deployments-and-how-to-perform-rolling-upgrade-and-rollbacks)

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