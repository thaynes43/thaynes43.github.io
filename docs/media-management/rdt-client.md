---
title: rdt-client
permalink: /docs/media-management/rdt-client/
---

Download Client for Real-Debrid.

## CRDs

I will use the already existing `download-clients` namespace and the `app-template` chart.

### Kustomization 

This just needs to be a health check in `helmrelease-download-clients.yaml`

```yaml
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: rdt-client
      namespace: download-clients
```

### HelmRelease

Looks about the same. There is no need for a `/config` but a `/data/db` instead.

`helmrelease-rdt-client.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: rdt-client
  namespace: download-clients
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
    defaultPodOptions:
      securityContext:
        fsGroup: 1000
    controllers:
      rdt-client:
        containers:
          app:
            image:
              repository: rogerfar/rdtclient # https://hub.docker.com/r/rogerfar/rdtclient
              tag: latest
              pullPolicy: IfNotPresent
    service:
      app:
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.121
        controller: rdt-client
        ports:
          http:
            port: 6500
    persistence:
      config:
        type: persistentVolumeClaim
        storageClass: ceph-rbd
        accessMode: ReadWriteOnce
        size: 4Gi
        globalMounts:
          - path: /data/db
            readOnly: false
      tank:
        type: persistentVolumeClaim
        existingClaim: pvc-smb-tank-k8s-download-clients
        globalMounts:
          - path: /tank
            readOnly: false
```

## GO LIVE!

One shot this one.