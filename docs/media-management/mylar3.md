---
title: Mylar3
permalink: /docs/media-management/mylar3/
---

Mylar3 is for comics. And hopefully it's as easy to slam into the cluster as the other stuff!

## CRDs

Namespace will share `media-management` with the others and the `app-template` chart will do.

### Kustomization

This will be added to the `servarr` kustomization as it's in the same realm though I don't think it shares a project. All we need here is the dependency:

```yaml
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: mylar3
      namespace: media-management
```

### HelmRelease

Just like the other `app-template` ones.

`helmrelease-mylar3.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: mylar3
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
      mylar3:
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/mylar3 # https://hub.docker.com/r/linuxserver/mylar3
              tag: latest
              pullPolicy: IfNotPresent
    service:
      app:
        controller: mylar3
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.120
        ports:
          http:
            port: 8090
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

## GO LIVE!

Seems good, complained that I didn't map it's default paths to directories but I could simply reconfigure them to use tank.