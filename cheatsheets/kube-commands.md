---
title: Kubernetes Cheat Sheet
permalink: /cheatsheets/kube-commands/
---

Here are some commands for managing Kubernetes.

## kubectx & kubens

These are helper scripts that come from [here](https://github.com/ahmetb/kubectx).

### Show available contexts (clusters)

```bash
kubectx
```

### Switch cluster

```bash
kubectx <cluster name>
```

### Show available namespaces

```bash
kubens
```

### Switch namespaces

```bash
kubens <namespace>
```

## kubectl

This is the beast that drives operations around your k8s cluster. Most of the heavy lifting seems to be done in the manifest yaml or helm charts but debugging and/or setting up requires heavy use of `kubectl`.

### kubectl apply

```bash
kubectl apply -f <filename.yaml>
```

### kubectl get

To see what's running, installed, or anything else we use `kubectl get <thing>`

#### get nodes

```bash
kubectl get nodes
```

#### get namespaces

```bash
kubectl get namespaces
```

#### get pods

```bash
kubectl get pods --all-namespaces
kubectl -n <namespace> kubget pods 
```

#### get services

This is useful for figuring port mappings.

```bash
kubectl get svc --all-namespaces
```

#### get volumes

```bash
kubectl get pv,pvc -o wide
```
#### get volumesnapshots

```bash
kubectl get volumesnapshot
```

#### get volumesnapshotclass

This will show you what type of snapshots you can take.

```bash
kubectl get volumesnapshotclass
```

#### get ceph cluster

Since I used rook this tells me how the ceph cluster exposed to k8s is doing.

```bash
kubectl -n rook-ceph  get CephCluster
```

### kubectl delete

#### delete manifest

```bash
kubectl delete -f <manifest.yaml>
```

#### delete all your volumes (don't do this!)

```bash
kubectl delete pvc --all
```

#### delete pv

```bash
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

```bash
kubectl logs -n velero -l app.kubernetes.io/name=velero
```

For a pod:

```bash
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

### Restart

```
https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_restart/

k rollout restart deployment/prowlarr -n media-management
```

### See what capabilities you have

This will show resources. Here we are checking to see if the cluster can make volumesnapshots or at least is eligible to be setup for volumesnapshots.

```bash
kubectl api-resources | grep volumesnapshots
```

## Velero

### Create Backup

```bash
velero backup create hellothere --include-namespaces=default --wait
```

```bash
 velero backup create --from-schedule=velero-daily-backups
 ```

#### TODO FROM DEBUGGING NEEDS ORGANIZING

```bash
kubectl get events -A
ceph -s
ceph osd lspools
kubectl logs -f velero-764d58dfd9-k47sh
```

## Mass Delete

### Jobs

```bash
kubectl get pod  -n velero --field-selector=status.phase==Succeeded

kubectl delete pod -n velero  --field-selector=status.phase==Succeeded

kubectl delete pod -n velero  --field-selector=status.phase==Failed
```

```bash
kubectl get jobs  -n velero --field-selector status.successful=1

kubectl delete jobs -n velero --field-selector status.successful=1
```

Pass ``--all-namespaces` for a good time!

```bash
velero delete backup move-data-test -n velero
velero delete backup velero-daily-backups-20240827000047 -n velero
velero delete backup velero-daily-backups-20240826000046 -n velero
```

TANK CHECK:

```bash
velero describe backup move-data-test-smb --details
```

```bash
kubectl -n velero rollout restart daemonset node-agent
kubectl -n velero rollout restart deployment velero
```

## Stuck stuff

```bash
kubectl get volumeattachment
```

See if it's attached, remove finalizers:

```bash
kubectl patch pvc {PVC_NAME} -p '{"metadata":{"finalizers":null}}'
```