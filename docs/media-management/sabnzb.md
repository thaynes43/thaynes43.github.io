---
title: SabNZB
permalink: /docs/sabnzb/
---

For SabNZB we already have an example over at the [app-template](https://bjw-s.github.io/helm-charts/docs/app-template/). We will go with the [linuxserver image](https://fleet.linuxserver.io/image?name=linuxserver/sabnzbd) to keep that consistent. 

## CRDs

### Namespace

Going to put downloader clients in a separate namespace from the "file management" services.

`namespace-download-clients.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: download-clients
```

### Kustomization 

This namespace can share a Kustomization. I don't really even see a name for seperate folders since it'll likely be two files with no secrets.

`kustomization-download-clients.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: sabnzb
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
  path: ./download-clients
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: sabnzb
      namespace: download-clients
```

### Helm Release

For this I wil run two instances of SabNZB as I like to route requests from Overseerr to a dedicated one incase the other is all backed up. 

Needed mappings can be found from [dockerhub](https://hub.docker.com/r/linuxserver/sabnzbd).

> **TODO** I just got rid of all the probe stuff but we may want to explore that later once we get more into monitoring

#### sabnzb-default

`helmrelease-sabnzb-default.yaml` 
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: sabnzb-default
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
    controllers:
      sabnzbd:
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/sabnzbd
              tag: latest
              pullPolicy: IfNotPresent
    service:
      app:
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.111        
        controller: sabnzb-default
        ports:
          http:
            port: 8080
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

#### sabnzb-mediarequests

`helmrelease-sabnzb-mediarequests.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: sabnzb-mediarequests
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
    controllers:
      sabnzbd:
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/sabnzbd
              tag: latest
              pullPolicy: IfNotPresent
    service:
      app:
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.112        
        controller: sabnzb-mediarequests
        ports:
          http:
            port: 8080
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

Indentation was super fucked:

```
thaynes@kubem01:~/workspace$ k get helmrelease -n download-clients 
NAME                   AGE   READY   STATUS
sabnzb-default         12m   False   Helm install failed for release download-clients/sabnzb-default with chart app-template@3.3.2: values don't meet the specifications of the schema(s) in the following chart(s):...
sabnzb-mediarequests   12m   False   Helm install failed for release download-clients/sabnzb-mediarequests with chart app-template@3.3.2: values don't meet the specifications of the schema(s) in the following chart(s):...
```

After this I could not get anything to reconcile. Was stuck. Will wait 15m and then plex dance it.

That wasn't stuck, it was just the same error so it didn't look different or update the time I guess. After that I hit:

```
  Warning  InstallFailed  3m57s (x2 over 5m27s)  helm-controller  Helm install failed for release download-clients/sabnzb-default with chart app-template@3.3.2: execution error at (app-template/templates/common.yaml:14:3): No enabled controller found with this identifier. (service: 'app', controller: 'sabnzb-default')
```

Which I realized I needed to match these two names:

```yaml
    controllers:
      sabnzb-mediarequests:
```

and

```yaml
    service:
      app:
        type: LoadBalancer
        controller: sabnzb-mediarequests
```

After that I was in!

```
thaynes@kubem01:~/workspace$ k get services,pods -n download-clients 
NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
service/sabnzb-default         LoadBalancer   10.43.43.27     192.168.40.111   8080:30538/TCP   46s
service/sabnzb-mediarequests   LoadBalancer   10.43.181.127   192.168.40.112   8080:30668/TCP   46s

NAME                                        READY   STATUS    RESTARTS   AGE
pod/sabnzb-default-7c4f8b6f97-gndc9         1/1     Running   0          46s
pod/sabnzb-mediarequests-66566b64cc-25k6v   1/1     Running   0          46s
```

But it wants me to do some quick start wizard before I can actually log into the app!

![sab-quick-start]({{ site.url }}/images/web/sab-quick-start.png)

## Exportarr

Exportarr supports SAB too. 