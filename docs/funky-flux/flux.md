---
title: Flux
permalink: /docs/funky-flux/flux/
---

In my attempts to come up with a [backup-strategy]({{ site.url }}/docs/funky-flux/backup-strat/) I came across a [tutorial for running velero](https://geek-cookbook.funkypenguin.co.nz/kubernetes/backup/velero/) that looked great. However, I didn't quite realize the dependencies this required that were basically conforming you to how one guy decided to set up his homelab. He used a tool called [flux](https://fluxcd.io/) which I found perfect for my needs but modified their official patterns to fix his needs in this [funky tutorial](https://geek-cookbook.funkypenguin.co.nz/kubernetes/deployment/flux/).

This will be the front page of building out my cluster using the patterns I've now adopted from [funky penguin](https://geek-cookbook.funkypenguin.co.nz/). He clearly knows a lot more than you and I so it'll be fine. Plus all 

## Seems Half Baked

A lot like these notes the author of this site has moved on to another project. He seemed to have started with docker swarm and later switched to k8s. This [premix](https://github.com/geek-cookbook/premix) repo has a handful of both but mostly compose files. But, for example, the [unifi](https://github.com/geek-cookbook/premix/tree/main/unifi/kubernetes) folder has both. This may be useful later on but I hope to eventually be able to funky flux anything on my own.

## Some Useful Commands

```
flux logs
flux get kustomizations --watch
flux reconcile source git flux-system
flux reconcile helmrelease -n <namespace> <name of helmrelease>
flux reconcile kustomization <name of kustomization>
```