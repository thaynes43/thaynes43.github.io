---
title: Bazarr
permalink: /docs/media-management/bazarr/
---

Everyone in my family (the elders) love subtitles so this one is a must. Should be a breeze too.

I am going all linuxserver images where possible and it looks like I can do that [here](https://hub.docker.com/r/linuxserver/bazarr)!

## CRDs

Not much for this one, same namespace and chart as the others.

`helmrelease-bazarr.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: bazarr
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
      bazarr:
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/bazarr # https://hub.docker.com/r/linuxserver/bazarr/tags
              tag: 1.4.3
              pullPolicy: IfNotPresent
    service:
      app:
        controller: bazarr
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.115
        ports:
          http:
            port: 6767
    persistence:
      config:
        type: persistentVolumeClaim
        storageClass: ceph-rbd
        accessMode: ReadWriteOnce
        size: 4Gi
        globalMounts:
          - path: /config
            readOnly: false
      tank:
        type: persistentVolumeClaim
        existingClaim: pvc-smb-tank-k8s-media-management
        globalMounts:
          - path: /tank
            readOnly: false
```

Then I've just been adding a health check to the Kustomization:

```yaml
...
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: lidarr-on-steroids
      namespace: bazarr
```

## Let it Rip!

And rip it did! No problems here, think this was the first HelmRelease I shot in that just worked.