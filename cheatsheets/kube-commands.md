---
title: Kubernetes Cheat Sheet
permalink: /cheatsheets/kube-commands/
---

Here are some commands for managing Kubernetes.

## kubectx & kubens

These are helper scripts that come from [here](https://github.com/ahmetb/kubectx).

### Show available contexts (clusters)

```
kubectx
```

### Switch cluster

```
kubectx <cluster name>
```

### Show available namespaces

```
kubens
```

### Switch namespaces

```
kubens <namespace>
```

## kubectl

This is the beast that drives operations around your k8s cluster. Most of the heavy lifting seems to be done in the manifest yaml or helm charts but debugging and/or setting up requires heavy use of `kubectl`.

### kubectl apply

```
kubectl apply -f <filename.yaml>
```

### kubectl get

To see what's running, installed, or anything else we use `kubectl get <thing>`

#### get nodes

```
kubectl get nodes
```

#### get namespaces

```
kubectl get namespaces
```

#### get pods

```
kubectl get pods --all-namespaces
kubectl -n <namespace> kubget pods 
```

#### get services

This is useful for figuring port mappings.

```
kubectl get svc --all-namespaces
```

#### get volumes

```
kubectl get pv,pvc -o wide
```
#### get volumesnapshots

```
kubectl get volumesnapshot
```

#### get volumesnapshotclass

This will show you what type of snapshots you can take.

```
kubectl get volumesnapshotclass
```

#### get ceph cluster

Since I used rook this tells me how the ceph cluster exposed to k8s is doing.

```
kubectl -n rook-ceph  get CephCluster
```

### kubectl delete

#### delete manifest

```
kubectl delete -f <manifest.yaml>
```

#### delete all your volumes (don't do this!)

```
kubectl delete pvc --all
```

#### delete pv

```
kubectl patch pvc -n namespace PVCNAME -p '{"metadata": {"finalizers": null}}'
kubectl patch pv PVNAME -p '{"metadata": {"finalizers": null}}'

k delete pvc PVCNAME --grace-period=0 --force
k delete pv PVNAME --grace-period=0 --force
```

### kubectl describe

Describe things in the cluster like pods and pvc's. This will give you some basic info to troubleshoot with before going to the logs (or if the logs are gone because the thing is crash looping.

```
kubectl describe <type> <name>
```

### kubectl logs

For an app:

```
kubectl logs -n velero -l app.kubernetes.io/name=velero
```

For a pod:

```
kubectl logs <podname>
```

### kubectl exec

Like bashing into a docker container but for a pod. Note if you are not kubens'd into the namespace you will need `-n <namespace>`

```
kubectl exec -it -n <namespace> <pod_name> -- env
```

Bash in:

```bash
k exec -it -n <namespace>  <pod_name> -- sh
k exec -it  -n <namespace>  <pod_name> -- /bin/bash
```

### See what capabilities you have

This will show resources. Here we are checking to see if the cluster can make volumesnapshots or at least is eligible to be setup for volumesnapshots.

```
kubectl api-resources | grep volumesnapshots
```

## Velero

### Create Backup

```
velero backup create hellothere --include-namespaces=default --wait
```

#### TODO FROM DEBUGGING NEEDS ORGANIZING

```
kubectl get events -A
ceph -s
ceph osd lspools
kubectl logs -f velero-764d58dfd9-k47sh
```

