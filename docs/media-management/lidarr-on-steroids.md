---
title: Lidarr on Steroids
permalink: /docs/media-management/lidarr-on-steroids/
---

I have had good luck using [lidarr-on-steroids](https://github.com/youegraillot/lidarr-on-steroids) and want to keep it going. Ideally I'd split out Deemix as a download client but the whole project seems defunct now.

## CRDs

Namespace will be `media-management` and we will use the `app-template` for helm.

### HelmRelease

`helmrelease-lidarr-on-steroids.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: lidarr-on-steroids
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
      lidarr-on-steroids:
        containers:
          app:
            image:
              repository: youegraillot/lidarr-on-steroids # https://hub.docker.com/r/youegraillot/lidarr-on-steroids
              tag: 1.5
              pullPolicy: IfNotPresent
    service:
      app:
        controller: lidarr-on-steroids
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.114
        ports:
          http:
            port: 8686
          deemix:
            port: 6595
    persistence:
      config:
        type: persistentVolumeClaim
        storageClass: ceph-rbd
        accessMode: ReadWriteOnce
        size: 40Gi
        globalMounts:
          - path: /config
            readOnly: false
      config-deemix:
        type: persistentVolumeClaim
        storageClass: ceph-rbd
        accessMode: ReadWriteOnce
        size: 2Gi
        globalMounts:
          - path: /config_deemix
            readOnly: false
      tank:
        type: persistentVolumeClaim
        existingClaim: pvc-smb-tank-k8s-media-management
        globalMounts:
          - path: /tank
            readOnly: false
```

## Test It!

The setup seemed to easy. I did skip mounting `/downloads` and `/music` which may have been required. I can see in my docker I mounted those but never used them and ended up using a custom `/data` path that mapped the whole `TRaSH` filesystem.

First I had to fiddle with the ports to expose two but [app-template docs](https://bjw-s.github.io/helm-charts/docs/app-template/) seemed to have an example.

Then I learned you can't have an underscore in the name of your pvc:

```
  Warning  InstallFailed  85s  helm-controller  Helm install failed for release media-management/lidarr-on-steroids with chart app-template@3.3.2: 2 errors occurred:
           * PersistentVolumeClaim "lidarr-on-steroids-config_deemix" is invalid: metadata.name: Invalid value: "lidarr-on-steroids-config_deemix": a lowercase RFC 1123 subdomain must consist of lower case alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character (e.g. 'example.com', regex used for validation is '[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*')
           * Deployment.apps "lidarr-on-steroids" is invalid: [spec.template.spec.volumes[1].name: Invalid value: "config_deemix": a lowercase RFC 1123 label must consist of lower case alphanumeric characters or '-', and must start and end with an alphanumeric character (e.g. 'my-name',  or '123-abc', regex used for validation is '[a-z0-9]([-a-z0-9]*[a-z0-9])?'), spec.template.spec.containers[0].volumeMounts[1].name: Not found: "config_deemix"]
```

After fixing that I had the service running!

```
thaynes@kubem01:~$ k -n media-management get services
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                         AGE
lidarr-on-steroids   LoadBalancer   10.43.12.64    192.168.40.114   6595:32427/TCP,8686:32078/TCP   5m19s
```

Both came preconfigured with 