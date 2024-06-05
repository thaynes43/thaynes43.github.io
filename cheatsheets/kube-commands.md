---
title: Kubernetes Cheat Sheet
permalink: /cheatsheets/kube-commands/
---

Here are some commands I'm going to be looking for...

## kubectl

### Apply manifest

```
kubectl apply -f <filename.yaml>
```

### Delete manifest

```
kubectl delete -f <filename.yaml>
```

### Get volumes

```
kubectl get pv,pvc -o wide
```

### Don't wipe them all!

```
kubectl delete pvc --all
```

