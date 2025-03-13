---
title: "Get a specific apiVersion manifest from k8s"
datePublished: Tue Mar 19 2024 16:47:27 GMT+0000 (Coordinated Universal Time)
cuid: cltyly4dv000409ib69trbav3
slug: get-a-specific-apiversion-manifest-from-k8s
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1710882241617/465b34ab-c33d-48d6-8688-ef8300773072.png
tags: kubernetes, devops, k8s

---

## TL;DR

> To get a manifest from k8s in any supported API version you can use an extended `kubectl get` notation like `kubectl get deployments.v1beta1.extensions mydeploy -o yaml`

## Briefly and with examples

Everybody knows how to get a resource manifest from Kubernetes. But do you know that you can put a manifest with one `apiVersion` set and get the same resource manifest with another `apiVersion` back?

### Imagine

1. You have a `deployment` in a cluster, that uses `api-version extensions/v1beta1` from a git repo.
    
2. You've updated the cluster (1.15 =&gt; 1.16). Old deployment works fine, but you can't deploy nothing new with the old manifest because in k8s 1.16 `api-version extensions/v1beta1` is absent.
    
3. You have two options:
    
    1. Rewrite it manually or
        
    2. Get it from the cluster in updated spec format (`apps/v1`)
        

To get manifest from k8s you can:

> below there is an example for v1.15, where extensions/v1beta1 is not removed yet

```bash
# from extensions
kubectl get deployments.extensions ext -o yaml
# from extensions, but for v1beta1
kubectl get deployments.v1beta1.extensions ext -o yaml
# from apps
kubectl get deployments.apps ext -o yaml
# from apps, but v1
kubectl get deployments.v1.apps ext -o yaml
```

Notice that if you do just `kubectl get deployments ext -o yaml`, you will get a manifest from `extensions/v1beta1` nevertheless you've even applied `apps/v1` before. Details [here](https://github.com/kubernetes/kubernetes/issues/58131#issuecomment-356823588).

## Prove it

1. Start 1.15
    
    ```bash
    minikube start --kubernetes-version=v1.15.12
    ```
    
2. Make 2 manifests for `deployments` with different versions. Note the absence of `spec.selector` in `extensions/v1beta1`
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        run: apps
      name: apps
    spec:
      replicas: 1
      selector:
        matchLabels:
          run: apps
      template:
        metadata:
          creationTimestamp: null
          labels:
            run: apps
        spec:
          containers:
          - image: nginx
            name: apps
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      labels:
        run: ext
      name: ext
    spec:
      replicas: 1
      template:
        metadata:
          creationTimestamp: null
          labels:
            run: ext
        spec:
          containers:
          - image: nginx
            name: ext
    ```
    
3. Apply
    
    ```bash
    ❯ minikube kubectl -- apply -f .
    deployment.apps/apps created
    deployment.extensions/ext created
    ```
    
4. Check that resources `apps` and `ext` are in `extensions/v1beta1` and in `apps/v1` too
    
    * The hard way - curl:
        
    
    ```bash
    ❯ minikube kubectl -- proxy
    Starting to serve on 127.0.0.1:8001
    ❯ curl -s 127.0.0.1:8001/apis/extensions/v1beta1/namespaces/default/deployments/apps | yq . --yaml-output
    kind: Deployment
    apiVersion: extensions/v1beta1
    # ...
    ❯ curl -s 127.0.0.1:8001/apis/extensions/v1beta1/namespaces/default/deployments/ext | yq . --yaml-output
    kind: Deployment
    apiVersion: extensions/v1beta1
    # ...
    ❯ curl -s 127.0.0.1:8001/apis/apps/v1/namespaces/default/deployments/apps | yq . --yaml-output
    kind: Deployment
    apiVersion: apps/v1
    # ...
    ❯ curl -s 127.0.0.1:8001/apis/apps/v1/namespaces/default/deployments/ext | yq . --yaml-output
    kind: Deployment
    apiVersion: apps/v1
    # ...
    ```
    
    * Or kubectl:
        
    
    ```bash
    # from extensions
    kubectl get deployments.extensions ext -o yaml
    # from extensions, but v1beta1
    kubectl get deployments.v1beta1.extensions ext -o yaml
    # from apps
    kubectl get deployments.apps ext -o yaml
    # from apps, but v1
    kubectl get deployments.v1.apps ext -o yaml
    ```
    

## Bonus: kubectl explain

If you do `kubectl explain deployment` than (surprise!) you'll get a description for `extensions/v1beta1`. Because `kubectl explain` [works the same way](https://github.com/kubernetes/kubernetes/issues/73062), just like `kubectl get`:

If you want a specific version, use an `--api-version` flag :

```bash
minikube kubectl -- explain deployment.spec --api-version apps/v1
minikube kubectl -- explain deployment.spec --api-version extensions/v1beta1
```