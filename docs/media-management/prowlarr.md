---
title: Prowlarr
permalink: /docs/media-management/prowlarr/
---

Hopefully now a rinse and repeat - we are going to use [linuxserver's prowlarr](https://hub.docker.com/r/linuxserver/prowlarr) with a [app-template](https://bjw-s.github.io/helm-charts/docs/app-template/) chart.

## CRDs

Namespace will continue to be `media-management` and helm will use `https://bjw-s.github.io/helm-charts`. I have reworked the Kustomizations a bit where the core *arrs are alone and will depend on this.

### Kustomization 

`kustomization-media-management.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: prowlarr
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
  path: ./media-management/prowlarr
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: prowlarr
      namespace: media-management
```

### HelmRelease

`helmrelease-prowlarr.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: prowlarr
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
    controllers:
      prowlarr:
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/prowlarr # https://hub.docker.com/r/linuxserver/prowlarr
              tag: 1.21.2
              pullPolicy: IfNotPresent
    service:
      app:
        controller: prowlarr
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.109
        ports:
          http:
            port: 9696
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

## Let it Rip!

There's some risk as I've combined two `Kustomizations` into one with multiple `HelmRelease` but I'm feeling pretty good about this one.

It worked but I noticed the IPs were not set to what I had annotated. Adding `loadBalancerIP: 192.168.40.113` helped but I the `truechart` sonar uses didn't expose this in it's values.yaml files. I think to assign IPs like this I'll need to do a bit more tinkering but some things I do want a local, non-ingres'd direct route to the IP/Port (like MQTT) but I guess that isn't a strict requirement.

### IPs Not Assigned Right

At this point I realized I was annotating the HelmRelease instead of the Service resource so it was a FFA for IPs when I moved sonarr and radarr.

I have documented the resolution for this [here]({{ site.url }}/docs/vlan-migration/tech-debt#metallb-annotations-wrong).

And after the flux dance I was able to hit `http://192.168.40.109:9696/` and log in!

> **TODO** This is another one that I think we can set to `External` though [the documentation](https://wiki.servarr.com/prowlarr/settings) has not been updated. However, we may want to use cluster IPs if we go this route (negating all that IP assignment work I did...)