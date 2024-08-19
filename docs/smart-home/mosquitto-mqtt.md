---
title: Mosquitto MQTT
permalink: /docs/smart-home/mosquitto-mqtt/
---

Many many things we will soon be configuring first require `mosquitto-mqtt` to be up and running. There is no official helm chart and much is stale but I also don't think mich is changing with this service. To start I will use [this chart](https://sintef.github.io/mosquitto-helm-chart`).

## Flux Files

### Namespace

Since this is the first of the smart home services it gets to define the namespace. I am going to put everything `Home Assistant` will integrate with under this namespace.

`namespace-iot-services.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: iot-services
```

### Helm Repository

Next I need the `mosquitto-helm-chart` chart repo @ `https://sintef.github.io/mosquitto-helm-chart`:

`helmrepository-mosquitto-helm-chart.yaml`:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: mosquitto-helm-chart
  namespace: flux-system
spec:
  interval: 15m
  url: https://sintef.github.io/mosquitto-helm-chart
```

### Bootstrap Kustomization

Finally I need a kustomization for the release. I am going to group these as well:

`kustomization-iot-services.yaml`:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: mosquitto
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
      name: mosquitto
      namespace: iot-services
  path: ./iot-services/mosquitto
```

### Helm Release

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: mosquitto
  namespace: iot-services
spec:
  chart:
    spec:
      chart: mosquitto
      version: 0.1.0
      sourceRef:
        kind: HelmRepository
        name: mosquitto-helm-chart
        namespace: flux-system
  interval: 15m
  timeout: 5m
  releaseName: mosquitto
  values: # found here https://github.com/SINTEF/mosquitto-helm-chart/blob/main/charts/mosquitto/values.yaml
    image:
      repository: eclipse-mosquitto
      pullPolicy: IfNotPresent
      # Overrides the image tag whose default is the chart appVersion.
      tag: "2.0.18" # https://hub.docker.com/_/eclipse-mosquitto/
    auth:
      enabled: false
    service:
      type: LoadBalancer
      mqttPort: 1883
      mqttOverWebsocketPort: 9001 
```

###  Complete Values

For reference here all the values available in this chart:

```yaml
# Default values for mosquitto.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

image:
  repository: eclipse-mosquitto
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: ""

mqttOverWebsocket: true

general:
  maxInflightMessages: 20
  maxQueuedMessages: 1000

auth:
  enabled: true
  users:
    - username: admin
      # "admin" password in sha512-pbkdf2 format, generated with mosquitto_passwd
      password: $7$101$T1RBXD5MGIImHq8g$hSCHVAyZtAif0qN9Fhuam9mVCd0xLomREHIwzdreGjjADCQewz9VpfhK6AumcZyVFHFpHd2EhZdqU+Lq7393Xw==
      acl:
        - topic: "#"
          access: readwrite

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext:
  fsGroup: 1883

securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1883

service:
  type: ClusterIP
  mqttPort: 1883
  mqttOverWebsocketPort: 9001 

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

resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

nodeSelector: {}

tolerations: []

affinity: {}
```

## Drop Shipping It In! 

Well the chart seems broken due to these common values:

```
    Message:                      Helm install failed for release iot-services/mosquitto with chart mosquitto@4.8.2: template: mosquitto/templates/common.yaml:17:3: executing "mosquitto/templates/common.yaml" at <include "common.all" .>: error calling include: template: mosquitto/charts/common/templates/_all.tpl:39:10: executing "common.all" at <include "common.deployment" .>: error calling include: template: mosquitto/charts/common/templates/_deployment.tpl:52:10: executing "common.deployment" at <include "common.controller.pod" .>: error calling include: template: mosquitto/charts/common/templates/lib/controller/_pod.tpl:64:6: executing "common.controller.pod" at <include "common.controller.mainContainer" .>: error calling include: template: mosquitto/charts/common/templates/lib/controller/_container.tpl:52:6: executing "common.controller.mainContainer" at <include "common.controller.ports" .>: error calling include: template: mosquitto/charts/common/templates/lib/controller/_ports.tpl:7:11: executing "common.controller.ports" at <.enabled>: can't evaluate field enabled in type interface {}
```

Gonna try [this one](https://github.com/SINTEF/mosquitto-helm-chart/tree/main/charts/mosquitto)...

That worked much better, everything above has been updated!

```
thaynes@kubem01:~/workspace$ k get pods,services -n iot-services 
NAME                             READY   STATUS    RESTARTS   AGE
pod/mosquitto-849b875478-frmnm   1/1     Running   0          85s

NAME                TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                         AGE
service/mosquitto   LoadBalancer   10.43.131.2   192.168.40.104   1883:32361/TCP,9001:30549/TCP   85s
```

Now we can see it in action by ![setting up Zigbee2MQTT]({{ site.url }}/docs/smart-home/zigbee2mqtt/).