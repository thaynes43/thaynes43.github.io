---
title: Zigbee2MQTT
permalink: /docs/smart-home/zwave-js/
---

[ZWave JS UI](https://zwave-js.github.io/zwave-js-ui/) seems to be the best bet for a Z2M experience with Z-Wave. To connect to devices I have a [tubeszb kit](https://tubeszb.com/product/z-wave-poe-kit/) with a [Zooz 800 GPIO](https://www.getzooz.com/zac93-gpio-module/).

## Flex Files

We will use the existing `iot-services` namespace for this service. The offical docs come close to a helm chart with a [kustomization](https://zwave-js.github.io/zwave-js-ui/#/getting-started/other-methods?id=kubernetes). However ArtifactHUB has a fairly recent chart [here](https://artifacthub.io/packages/helm/k8sonlab/zwave-js-ui).

```yaml
helm repo add k8sonlab https://charts.ar80.eu
helm install my-zwave-js-ui k8sonlab/zwave-js-ui --version 0.2.98
```

### HelmRepository

`helmrepository-k8sonlab.yaml`:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: k8sonlab
  namespace: flux-system
spec:
  interval: 15m
  url: https://charts.ar80.eu
```

### Bootstrap Kustomization

Adding to `kustomization-iot-services.yaml`:
```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: zwave-js-ui
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
  healthChecks:
    - apiVersion: apps/v1
      kind: HelmRelease
      name: zwave-js
      namespace: iot-services
  path: ./iot-services/zwave-js-ui
```

### HelmRelease

Source for chart is [here](https://github.com/k8sonlab/publiccharts/tree/main/charts/zwave-js-ui).

`helmrelease-zwave-js-ui.md`:
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: zwave-js-ui
  namespace: iot-services
spec:
  chart:
    spec:
      chart: zwave-js-ui
      version: 0.2.x
      sourceRef:
        kind: HelmRepository
        name: k8sonlab
        namespace: flux-system
  interval: 15m
  timeout: 5m
  releaseName: zwave-js-ui
  values: # https://github.com/k8sonlab/publiccharts/blob/main/charts/zwave-js-ui/values.yaml
    image:
      tag: "9.17.0" # https://github.com/zwave-js/zwave-js-ui/releases
    service:
      type: LoadBalancer
      annotations:
        metallb.universe.tf/loadBalancerIPs: 192.168.40.106
    persistence:
      enabled: true
      size: 2Gi
      existingClaim: ""
      accessMode: ReadWriteOnce
      storageClass: ceph-rbd
```

### IngressRoute w/ Auth Proxy

Since this is an internal service that I will be accessing frequently I want a nice URL as well as SSO and having it in my Authentik Application menu. This should be as easy as following the steps from the dashboard.

> **TODO** Clean the proxy provider part up and add to a tutorial, then link to this

`ingressroute-zwave-js-ui.yaml`

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: zwave-js-ui
  namespace: traefik
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`www.zwave.local.example.com`)
      kind: Rule
      services:
        - kind: Service
          name: zwave-js-ui
          namespace: iot-services
          port: 80
      middlewares:  
        - name: authentik-auth-proxy
          namespace: traefik 
    - kind: Rule
      match: Host(`zwave.local.example.com`) && PathPrefix(`/`)
      services:
        - kind: Service
          name: zwave-js-ui
          namespace: iot-services
          port: 80
      middlewares:  
        - name: authentik-auth-proxy
          namespace: traefik 
  tls:
    secretName: certificate-local.example.com
```

## Let It Rip!

> **NOTE** Findings below have been applied to the configuration examples above.

### Error 1

It didn't like how I defined my `accessModes`:

```
           * PersistentVolumeClaim "zwave-js-ui-store" is invalid: [spec.accessModes: Unsupported value: "": supported values: "ReadOnlyMany", "ReadWriteMany", "ReadWriteOnce", "ReadWriteOncePod", spec.resources[storage]: Invalid value: "0": must be greater than zero]
```

The chart is a bit different than others for [pvc template](https://github.com/k8sonlab/publiccharts/blob/main/charts/zwave-js-ui/templates/pvc.yaml) so I needed to tweak a few things.

### Error 2

```
  Warning  Failed                  2m43s (x4 over 3m32s)   kubelet                  Error: failed to create containerd task: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: error mounting "/run/k3s/containerd/io.containerd.runtime.v2.task/k8s.io/zwave-js-ui/tcp:/tubeszb-zwave01.example:6638" to rootfs at "/dev/ttyUSB0": stat /run/k3s/containerd/io.containerd.runtime.v2.task/k8s.io/zwave-js-ui/tcp:/tubeszb-zwave01.example:6638: no such file or directory: unknown
```

This was adapter configuration related which was all but certain. For this I simply removed the path to the usbdevice.

## Testing

First, I must trend lightly here:

![zwave-is-illegal]({{ site.url }}/images/haos/zwave/zwave-is-illegal.png)

Now that it's live I need to configure Z-Wave to use `tcp://tubeszb-zwave01.example:6638` and pair some devices. To start we will use a [Zooz ZSE41 open | close xs sensor](https://devices.zwave-js.io/?jumpTo=0x027a:0x7000:0xe001:0.0) which will pair well with my device.

Took me a while to find the button to add new devices but here it is:

![manage-nodes]({{ site.url }}/images/haos/zwave/manage-nodes.png)

But pairing from there was easy (since the default options all worked)!

![my-first-zwave]({{ site.url }}/images/haos/zwave/my-first-zwave.png)

## Home Assistant Integration

This was so easy there's really not much to say. I just went "Add Integration" -> Uncheck "Use the Z-Wave JS Supervisor add-on" -> Submit:

![add-zwave]({{ site.url }}/images/haos/add-zwave.png)

Then I just pointed at my instance of ZWave-JS-UI (which I mapped to a nice DNS entry `ws://zwave.example:3000`) and everything was instantly added!

![first-zwave-device-in-haos]({{ site.url }}/images/haos/first-zwave-device-in-haos.png)

## Appendix I - All Values

Values as of `0.2.98` for reference:

> **TODO** Collapse big blocks like this!

```yaml
# Default values for zwave-js-ui.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
replicaCount: 1

image:
  repository: zwavejs/zwave-js-ui
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: ""

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

# -- configure Probes
health:
  livenessProbe:
    path: /health
    initialDelaySeconds: 15
    periodSeconds: 30
    httpHeaders:
    - name: Accept
      value: "text/plain"
  readinessProbe:
    path: /health
    initialDelaySeconds: 5
    periodSeconds: 30
    httpHeaders:
    - name: Accept
      value: "text/plain"
  startupProbe:
    path: /health
    initialDelaySeconds: 5
    periodSeconds: 30
    httpHeaders:
    - name: Accept
      value: "text/plain"

# -- ui and websocet ports
ports:
  ui:
    name: http-ui
    containerPort: 8091
    servicePort: 80
    protocol: TCP
  websocket:
    name: http-websocket
    containerPort: 3000
    servicePort: 3000
    protocol: TCP

service:
  type: ClusterIP
  port: 8091  
  annotations: {}

# -- Setting default strategy, to avoid running 2 containers with one stick
strategy:
  type: Recreate

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

# -- add your env variables here, following standard syntax
env:
  - name: ZWAVE_JS_EXTERNAL_CONFIG
    value: /usr/src/app/store/.config-db

# -- you can add secrets and configmaps. this way you support external secrets for secure variables
envFrom: []

podAnnotations: {}

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

persistence:
  # -- enable persistent volume, otherwise use empty dir
  enabled: false
  # existingClaim: my-claim
  # -- change the path of store. Just in case you use different env variable.
  mountPath: /usr/src/app/store
  # -- Optionally specify a subPath
  # subPath: /path/to/config

# -- Support custom usb device
#usbDevice: /dev/ttyUSB0

ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

# -- Initial resources, based on a 40node network
resources:
  limits:
    cpu: 300m
    memory: 256Mi
  requests:
    cpu: 200m
    memory: 192Mi

nodeSelector: {}

tolerations: []

affinity: {}

# -- Support Prometheus ServiceMonitor
serviceMonitor:
  # -- enable Service Monitor
  enabled: false
  # -- add Custom labels, for prometheus Service Monitor
  labels: {}
  # -- interval
  interval: 30s
  # -- namespace selector
  namespaceSelector: {}
  # -- endpoint additions - add endpoint modifications
  endpointAdditions: {}
```