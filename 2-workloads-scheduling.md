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

- To create a Deployment 
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

- To update a Deployment, there are several ways you can achieve that:
```bash

```
</details>