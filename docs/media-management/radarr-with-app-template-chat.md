---
title: App Template Chart
permalink: /docs/app-template-chart/
---

I'm like liking the complexity of truecharts. Fortunately I've found something that looks simple but extensible to give a try [here](https://bjw-s.github.io/helm-charts/docs/app-template/). Says tons of examples can be found [here](https://kubesearch.dev/).

## Using The Chart

Gonna try using the chart with linuxserver radarr. 

* Values: https://github.com/bjw-s/helm-charts/blob/main/charts/apps/k8s-ycl/values.yaml
* Common Values: https://github.com/bjw-s/helm-charts/blob/main/charts/library/common/values.yaml

### HelmRepository

`helmrepository-bjw-s.yaml`
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: bjw-s
  namespace: flux-system
spec:
  interval: 15m
  url: https://bjw-s.github.io/helm-charts
```

### Kustomization

This is going in the existing `` file:

`kustomization-media-management.yaml`
```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: radarr
  namespace: flux-system
spec:
  #dependsOn: 
  #  - name: TODO depends on download clients
  interval: 15m
  retryInterval: 1m  
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./media-management/radarr
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: radarr
      namespace: media-management
```

### HelmRelease for Radarr

For this I am following the [3.0.x sabnzbd example](https://bjw-s.github.io/helm-charts/docs/app-template/#upgrade-instructions) plus the [example flux helmrelease](https://github.com/bjw-s/helm-charts/blob/main/examples/flux/helmrelease.yaml).

I have a good idea of how large the volume needs to be, which is north of 10Gi, because this has already loaded my data on unRAID:

![radarr-folder-size]({{ site.url }}/images/windows/radarr-folder-size.png)

`helmrelease-radarr.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: radarr
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
      radarr:
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/radarr # https://fleet.linuxserver.io/image?name=linuxserver/radarr
              tag: 5.9.1
              pullPolicy: IfNotPresent
            probes:
              liveness:
                enabled: true
              readiness:
                enabled: true
              startup:
                enabled: true
                spec:
                  failureThreshold: 30
                  periodSeconds: 5
    service:
      app:
        controller: radarr
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.110
        ports:
          http:
            port: 7878
    persistence:
      config:
        type: persistentVolumeClaim
        storageClass: ceph-rbd
        accessMode: ReadWriteOnce
        size: 15Gi
        globalMounts:
          - path: /config
            readOnly: false
```

## Test It

The service starts but restarts after a few minutes:

```
2024-08-23 14:21:36	[ls.io-init] done.
2024-08-23 14:24:00	[Info] Microsoft.Hosting.Lifetime: Application is shutting down... 
2024-08-23 14:24:00	[Info] ConsoleApp: Exiting main. 
```

It also never goes ready:

```
thaynes@kubem01:~/workspace$ k get pods -n media-management 
NAME                                READY   STATUS    RESTARTS       AGE
radarr-847f76b785-4lbs5             0/1     Running   2 (114s ago)   7m6s
```

Maybe a probe thing:

[Docs on probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) may help or reviewing comments [here](https://github.com/bjw-s/helm-charts/blob/main/charts/library/common/values.yaml#L271).

```
  Type     Reason                  Age                    From                     Message
  ----     ------                  ----                   ----                     -------
  Warning  FailedScheduling        43m                    default-scheduler        0/8 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/8 nodes are available: 8 Preemption is not helpful for scheduling.
  Normal   Scheduled               43m                    default-scheduler        Successfully assigned media-management/radarr-847f76b785-4lbs5 to kubew03
  Normal   SuccessfulAttachVolume  43m                    attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-de9d2617-fe33-4838-ba46-4db22566dc48"
  Normal   Pulling                 43m                    kubelet                  Pulling image "lscr.io/linuxserver/radarr:5.9.1"
  Normal   Pulled                  43m                    kubelet                  Successfully pulled image "lscr.io/linuxserver/radarr:5.9.1" in 6.49s (6.49s including waiting). Image size: 86004114 bytes.
  Normal   Created                 43m                    kubelet                  Created container app
  Normal   Started                 43m                    kubelet                  Started container app
  Warning  Unhealthy               8m45s (x326 over 43m)  kubelet                  Startup probe failed: dial tcp 10.42.5.105:8080: connect: connection refused
  Warning  BackOff                 3m39s (x64 over 26m)   kubelet                  Back-off restarting failed container app in pod radarr-847f76b785-4lbs5_media-management(0296f2b5-df57-4e0f-80cf-b8609b511dcf)
```

Final issue was the wrong port. Once mapped we're in! Note that the [External](https://wiki.servarr.com/radarr/faq#forced-authentication) auth setting is needed in this config too if I want to disable the popup.

### Radarr-Exporter

The `truecharts` repo came with these and I am already running a dashboard in grafana on unRAID so I can easily migrate that over. I'll just need to pull it myself for this templated chart.

> **NOTE** I will make this into it's own thing, than update back here as I apply it per arr. 