---
title: Overseerr
permalink: /docs/media-management/overseer/
---

Should be another easy one. This sits at the top of the stack as an end point for users to interact with what media is on the server. It's not really needed anymore since you can hook watchlists with sonarr and radarr but everyone's use to it already.

## CRDs

`media-management` namespace already exists and we will use the `app-template` for the chart.

### Kustomization

I want this to depend on it's downstream arr dependencies so I will make it it's own kustomization but in the same `kustomization-media-management.yaml` file.

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: request-management
  namespace: flux-system
spec:
  dependsOn:
    - name: servarr
  interval: 15m
  retryInterval: 1m  
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./media-management/request-management
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: overseerr
      namespace: media-management
```

I kept it general with `request-management` incase I wanted to add ombi or something later.

### HelmRelease

Hopefully another easy one

`helmrelease-overseerr.md`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: overseerr
  namespace: media-management
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
    defaultPodOptions:
      securityContext:
        fsGroup: 1000
    controllers:
      overseerr:
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/overseerr # https://hub.docker.com/r/linuxserver/overseerr
              tag: latest
              pullPolicy: IfNotPresent
    service:
      app:
        controller: overseerr
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.119
        ports:
          http:
            port: 5055
    persistence:
      config:
        type: persistentVolumeClaim
        storageClass: ceph-rbd
        accessMode: ReadWriteOnce
        size: 4Gi
        globalMounts:
          - path: /config
            readOnly: false
```

## GO LIVE!

ONE SHOT