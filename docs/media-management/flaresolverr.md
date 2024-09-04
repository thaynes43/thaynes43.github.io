---
title: FlareSolverr
permalink: /docs/flaresolverr/
---

This is needed for some indexers with Prowlarr, 1337x asked for it last time I set things up. It can be used in general so I will add it to the `cloudflare` namespace.

## CRDs

### Kustomization

Adding to the existing `kustomization-cloudflare.yaml`.

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: flaresolverr
  namespace: flux-system
spec:
  interval: 15m
  retryInterval: 1m
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./cloudflare/flaresolverr
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: flaresolverr
      namespace: cloudflare
```

### HelmRelease

Very straightforward service with no need for persistence or secrets.

`helmrelease-flaresolverr.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: flaresolverr
  namespace: cloudflare
spec:
  chart:
    spec:
      chart: app-template
      version: 3.3.2 # https://github.com/bjw-s/helm-charts/tree/main/charts/other/app-template
      sourceRef:
        kind: HelmRepository
        name: bjw-s
        namespace: flux-system
  interval: 15m
  timeout: 5m
  values:
    # Common https://github.com/bjw-s/helm-charts/blob/main/charts/library/common/values.yaml
    controllers:
      flaresolverr:
        containers:
          app:
            image:
              repository: ghcr.io/flaresolverr/flaresolverr # https://github.com/FlareSolverr/FlareSolverr
              tag: latest
              pullPolicy: IfNotPresent
    service:
      app:
        controller: flaresolverr
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.122
        ports:
          http:
            port: 8191
```