---
title: Kubernetes Cheat Sheet
permalink: /cheatsheets/kube-commands/
---

```
flux logs
flux get kustomizations --watch
flux reconcile source git flux-system
flux reconcile helmrelease -n <namespace> <name of helmrelease>
flux reconcile kustomization <name of kustomization>
```

## Watch for updates

At first I was using:

```bash
flux get kustomizations --watch
```

But it got weird so I switched to:

```bash
watch -n 1 flux get kustomizations
watch --interval 1 --no-wrap flux get kustomizations
```