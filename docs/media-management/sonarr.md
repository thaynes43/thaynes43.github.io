---
title: Sonarr
permalink: /docs/sonarr/
---

This document goes through my journey setting up [Sonarr](https://sonarr.tv/#home) on k8s through a flux deployment process.

## Forward

I found [this video](https://www.youtube.com/watch?v=OBJa2G3Ef7o) for Sonarr w/ Authentik and Nginx so I figured I'd start here and work my way around the stack. 

After watching it I realized the Proxy Provider I have been using can pass http basic auth along. 

> **TODO** See if we can pass basic auth / tokens for Kube and Traefik dashboard the way it shows here

Another one where we have [k8s-at-home](https://github.com/k8s-at-home/charts/tree/master/charts/stable/sonarr), which is archived, or [truecharts](https://truecharts.org/charts/stable/sonarr/) who's chart is [here](https://github.com/truecharts/charts/tree/master/charts/stable/sonarr). 

`Truecharts` has at least been updated to `4.0.8` which is the latest from the [repository](https://github.com/onedr0p/containers/pkgs/container/sonarr) it pulls from. This is a good sign of the chart being up to date.

## Flux Files

Here comes the pain.

### Namespace

Since we are starting in the middle for my media "management" stack I'm adding a namespace:

`namespace-media-management.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: media-management
```

### HelmRepository

This was added before but is a weird one so I'll throw it here so people don't have to go searching.

`helmrepository-truecharts.yaml`
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
    name: truecharts
    namespace: flux-system
spec:
    type: oci
    interval: 5m
    url: oci://tccr.io/truecharts
```

### Kustomization

I plan to have one Kustomization for the arr stack and then another for it's dependencies (download clients etc).

> **NOTE** I will go back and add dependencies once the stack is up. Starting here for the auth tutorial I found.

`kustomization-media-management.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: sonarr
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
  path: ./media-management/sonarr
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: sonarr
      namespace: sonarr
```

### HelmRelease

`helmrelease-sonarr.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: sonarr
  namespace: media-management
spec:
  chart:
    spec:
      chart: sonarr
      version: 23.1.4 # https://truecharts.org/charts/stable/sonarr/ 
      sourceRef:
        kind: HelmRepository
        name: truecharts
        namespace: flux-system
  interval: 15m
  timeout: 5m
  values: 
    # Sonarr defaults found here https://github.com/truecharts/charts/blob/master/charts/stable/sonarr/values.yaml
    # Common defaults found here https://github.com/truecharts/library-charts/blob/main/library/common/values.yaml
    global:
      annotations:
        metallb.universe.tf/loadBalancerIPs: 192.168.40.113    
      fallbackDefaults:
        serviceType: LoadBalancer
        storageClass: cephfs # exporter and sonarr both access the volumes
        pvcSize: 2Gi
        vctSize: 2Gi
        accessModes:
          - ReadWriteMany
        vctAccessModes:
          - ReadWriteMany
    workload:
      main:
        podSpec:
          containers:
            main:
              env:
                SONARR__AUTHENTICATION_METHOD: "External"
```

### App-Template HelmRelease

Gonna move this one away from truecharts now.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: sonarr
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
      annotations: 
        backup.velero.io/backup-volumes-excludes: tank
      securityContext:
        fsGroup: 1000
    controllers:
      sonarr:
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/sonarr # https://fleet.linuxserver.io/image?name=linuxserver/sonarr
              tag: 4.0.9
              pullPolicy: IfNotPresent
            env:
              SONARR__AUTH__METHOD: External
              SONARR__AUTH__REQUIRED: DisabledForLocalAddresses
    service:
      app:
        controller: sonarr
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.113
        ports:
          http:
            port: 8989
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

## Let it Rip!

Only one snag this time! I needed `cephfs` w/ `ReadWriteMany` because the exporter was also reading the config file. Once I got this all set I fiddled with the auth settings a bit but landed on:

![sonarr-auth-setting]({{ site.url }}/images/web/sonarr-auth-setting.png)

Ideally I'd have set this to [External](https://wiki.servarr.com/sonarr/faq-v4#forced-authentication) but I do not know how to modify `config.xml` from the helm chart and the env var values.yaml provides does nothing.

> **WARNING** I just edited the file on the volume by exec'ing into the pod. Setting it via helm would be smarter, this might be a good candidate to write my own chart

## Rest of the Files

### DNSEndpoint

> **TODO** Since this is only for local access I manually did this in UniFi but, as stated before, ideally we have the sidecar that does it for us. 

### IngressRoute

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: sonarr
  namespace: traefik
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`www.sonarr.local.example.com`)
      kind: Rule
      services:
        - kind: Service
          name: sonarr
          namespace: media-management
          port: 8989
      middlewares:  
        - name: authentik-auth-proxy
          namespace: traefik 
    - kind: Rule
      match: Host(`sonarr.local.example.com`) && PathPrefix(`/`)
      services:
        - kind: Service
          name: sonarr
          namespace: media-management
          port: 8989
      middlewares:  
        - name: authentik-auth-proxy
          namespace: traefik 
  tls:
    secretName: certificate-local.example.com
```

## Authentik 

> **TODO** Need to flip services I've secured with Authetik from `LoadBalancer` to `ClusterIP` so they can only be acceded through the `IngressRoute`

For this I just followed the tutorial over at 

## Values Reference

```yaml
image:
  repository: ghcr.io/onedr0p/sonarr
  pullPolicy: IfNotPresent
  tag: 4.0.8.1874@sha256:3c8d3d5648f9d292d834252e98c34f459ea81a906ab88782bd53f405bb2c4b26
exportarrImage:
  repository: ghcr.io/onedr0p/exportarr
  pullPolicy: IfNotPresent
  tag: v2.0.1@sha256:727e7bc8f2f0934a2117978c59f4476b954018b849a010ea6cfb380bd6539644
service:
  main:
    ports:
      main:
        port: 8989
  metrics:
    enabled: true
    type: ClusterIP
    targetSelector: exportarr
    ports:
      metrics:
        enabled: true
        port: 8990
        targetSelector: exportarr
workload:
  main:
    podSpec:
      containers:
        main:
          probes:
            liveness:
              enabled: true
              type: http
              path: /ping
            readiness:
              enabled: true
              type: http
              path: /ping
            startup:
              enabled: true
              type: http
              path: /ping
          env:
            SONARR__PORT: "{{ .Values.service.main.ports.main.port }}"
            SONARR__AUTHENTICATION_METHOD: ""
  exportarr:
    enabled: true
    type: Deployment
    strategy: RollingUpdate
    replicas: 1
    podSpec:
      containers:
        exportarr:
          primary: true
          enabled: true
          imageSelector: exportarrImage
          args:
            - sonarr
          probes:
            liveness:
              enabled: true
              type: http
              path: /healthz
              port: "{{ .Values.service.metrics.ports.metrics.port }}"
            readiness:
              enabled: true
              type: http
              path: /healthz
              port: "{{ .Values.service.metrics.ports.metrics.port }}"
            startup:
              enabled: true
              type: http
              path: /healthz
              port: "{{ .Values.service.metrics.ports.metrics.port }}"
          env:
            INTERFACE: 0.0.0.0
            PORT: "{{ .Values.service.metrics.ports.metrics.port }}"
            URL: '{{ printf "http://%v:%v" (include "tc.v1.common.lib.chart.names.fullname" $) .Values.service.main.ports.main.port }}'
            # additional metrics (slow)
            # ENABLE_ADDITIONAL_METRICS: false
            # enable gathering unknown queue items
            # ENABLE_UNKNOWN_QUEUE_ITEMS: false
            CONFIG: "/config/config.xml"
persistence:
  config:
    enabled: true
    targetSelector:
      main:
        main:
          mountPath: /config
      exportarr:
        exportarr:
          mountPath: /config
          readOnly: true
metrics:
  main:
    enabled: true
    type: "servicemonitor"
    endpoints:
      - port: metrics
        path: /metrics
    targetSelector: metrics
    prometheusRule:
      enabled: false
portal:
  open:
    enabled: true
updated: true
```

## External Auto Config

I found [this chart](https://github.com/onedr0p/home-ops/blob/main/kubernetes/main/apps/default/sonarr/app/helmrelease.yaml) that shows how to configure External auth w/ envs.

> **TODO** we may never use this externally so we may not want this

## envs

Looks like I can set some of the values I couldn't edit in config from env. For far I've tried:

```yaml
            env:
              SONARR__AUTH__METHOD: External
              SONARR__AUTH__REQUIRED: DisabledForLocalAddresses
```

[This HelmRelease](https://github.com/onedr0p/home-ops/blob/main/kubernetes/main/apps/default/sonarr/app/helmrelease.yaml) has a bunch more:

```yaml
            env:
              SONARR__APP__INSTANCENAME: Sonarr
              SONARR__APP__THEME: dark
              SONARR__AUTH__METHOD: External
              SONARR__AUTH__REQUIRED: DisabledForLocalAddresses
              SONARR__LOG__DBENABLED: "False"
              SONARR__LOG__LEVEL: info
              SONARR__SERVER__PORT: &port 80
              SONARR__UPDATE__BRANCH: develop
              TZ: America/New_York
```

There are also a bunch of variables being set from external secretes [here](https://github.com/onedr0p/home-ops/blob/main/kubernetes/main/apps/default/sonarr/app/externalsecret.yaml).

This looks pretty cool, like a way to share these values across services since they are being pulled from `1password` using this image `docker.io/1password/connect-api`.

## System Upgrade

For upgrading k3s and debian it looks like we can use [System Upgrade Controller](https://github.com/rancher/system-upgrade-controller).

