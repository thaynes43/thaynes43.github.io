---
title: Readarr
permalink: /docs/media-management/readarr/
---

Since Bazarr was a breeze I'm feeling good for Readarr. Only hang up I can think of will be around needing two of these, but each can use the same port since I'll load balance them so the config should be identical. 

## CRDs

Namespace will be `media-management` and we will use the `app-template` chart. For this arr we need two, otherwise we can't select between audio and ebook. For some reason they designed this as if an audio book was a 4K version of an ebook.

### HelmRelease

Hopefully just following the same pattern as for the other arrs.

`helmrelease-readarr-ebooks.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: readarr-ebooks
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
      readarr-ebooks:
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/readarr # https://hub.docker.com/r/linuxserver/readarr
              tag: develop
              pullPolicy: IfNotPresent
    service:
      app:
        controller: readarr-ebooks
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.118
        ports:
          http:
            port: 8787
    persistence:
      config:
        type: persistentVolumeClaim
        storageClass: ceph-rbd
        accessMode: ReadWriteOnce
        size: 15Gi
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

`helmrelease-readarr-audiobooks.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: readarr-audiobooks
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
      readarr-audiobooks:
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/readarr # https://hub.docker.com/r/linuxserver/readarr
              tag: develop
              pullPolicy: IfNotPresent
    service:
      app:
        controller: readarr-audiobooks
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.117
        ports:
          http:
            port: 8787
    persistence:
      config:
        type: persistentVolumeClaim
        storageClass: ceph-rbd
        accessMode: ReadWriteOnce
        size: 15Gi
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

### Kustomization Change

Add this to the health checks for servarr:

```yaml
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: readarr-ebooks
      namespace: media-management
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: readarr-audiobooks
      namespace: media-management
```

## Let it rip!

Had an IP conflict, next time I'll grep now that the list is getting huge. Otherwise flawless victory!