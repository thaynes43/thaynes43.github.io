---
title: External Secrets
permalink: /docs/external-secrets/
---

To use [external-secret](https://external-secrets.io/main/introduction/getting-started/) we first need a provider. It seems to support a ton but the cool kids use [1Password](https://1password.com/). To connect this up we need to use the [1Password Connect API](https://developer.1password.com/docs/connect/get-started/) which has a docker repo [here](https://hub.docker.com/r/1password/connect-api).

## 1Password

Instructions on the site were simple to get the "Connect server" setup on their end. Next I need to deploy a docker to host this server, image found [here](https://hub.docker.com/r/1password/connect-api).

### Secret CRDs

To do this I need to shove the connect server token and `1password-credentials.json` file in a secret. Instead of shoving the file in I can convert it to base64:

```bash
base64 1password-credentials.json -w 0 > 1password-credentials.b64
```

Now I can create `onepassword-connect-secret.yaml` which can then seal:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: onepassword-connect-secret
  namespace: external-secrets
type: Opaque
stringData:
  1password-credentials.json: "REDACTED"
  token: "REDACTED"
```

> **WARNING** I think sops is the way to go for this and is probably how people bootstrap new clusters with no fuss. I will circle back and explore sops later as I am first focusing on getting one cluster running with minor manual steps for bootstraping.

```bash
cat onepassword-connect-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-onepassword-connect-secret.yaml
```

And here it is:

`sealedsecret-onepassword-connect-secret.yaml`

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: onepassword-connect-secret
  namespace: external-secrets
spec:
  encryptedData:
    1password-credentials.json: REDACTED
    token: REDACTED
  template:
    metadata:
      creationTimestamp: null
      name: onepassword-connect-secret
      namespace: external-secrets
    type: Opaque
```

### Rest of the CRDs

### Flux Kustomization

Pretty straightforward:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: &app onepassword-connect
  namespace: flux-system
spec:
  targetNamespace: external-secrets
  commonMetadata:
    labels:
      app.kubernetes.io/name: *app
  interval: 15m
  retryInterval: 1m
  timeout: 2m
  prune: true
  wait: true
  dependsOn:
    - name: sealed-secrets
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./kubernetes/main/apps/system/secrets/external/onepassword-connect/app
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: onepassword-connect
      namespace: external-secrets
```

### HelmRelease

```yaml
---
# yaml-language-server: $schema=https://raw.githubusercontent.com/bjw-s/helm-charts/main/charts/other/app-template/schemas/helmrelease-helm-v2.schema.json
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: onepassword-connect
spec:
  interval: 30m
  chart:
    spec:
      chart: app-template
      version: 3.4.0
      sourceRef:
        kind: HelmRepository
        name: bjw-s
        namespace: flux-system
  install:
    remediation:
      retries: 3
  upgrade:
    cleanupOnFail: true
    remediation:
      strategy: rollback
      retries: 3
  values:
    controllers:
      onepassword-connect:
        strategy: RollingUpdate
        #annotations:
        #  reloader.stakater.com/auto: "true"
        containers:
          api:
            image:
              # https://hub.docker.com/r/1password/connect-api/tags
              repository: docker.io/1password/connect-api
              tag: 1.7.3
            env:
              XDG_DATA_HOME: &configDir /config
              OP_HTTP_PORT: &apiPort 80
              OP_BUS_PORT: 11220
              OP_BUS_PEERS: localhost:11221
              OP_SESSION:
                valueFrom:
                  secretKeyRef:
                    name: onepassword-connect-secret
                    key: 1password-credentials.json
            probes:
              liveness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /heartbeat
                    port: *apiPort
                  initialDelaySeconds: 15
                  periodSeconds: 30
                  failureThreshold: 3
              readiness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /health
                    port: *apiPort
                  initialDelaySeconds: 15
            securityContext: &securityContext
              allowPrivilegeEscalation: false
              readOnlyRootFilesystem: true
              capabilities: { drop: ["ALL"] }
            resources: &resources
              requests:
                cpu: 10m
              limits:
                memory: 256M
          sync:
            image:
              # https://hub.docker.com/r/1password/connect-sync/tags
              repository: docker.io/1password/connect-sync
              tag: 1.7.3
            env:
              XDG_DATA_HOME: *configDir
              OP_HTTP_PORT: &syncPort 8081
              OP_BUS_PORT: 11221
              OP_BUS_PEERS: localhost:11220
              OP_SESSION:
                valueFrom:
                  secretKeyRef:
                    name: onepassword-connect-secret
                    key: 1password-credentials.json
            probes:
              liveness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /heartbeat
                    port: *syncPort
                  initialDelaySeconds: 15
                  periodSeconds: 30
                  failureThreshold: 3
              readiness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /health
                    port: *syncPort
                  initialDelaySeconds: 15
            securityContext: *securityContext
            resources: *resources
    defaultPodOptions:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
        fsGroupChangePolicy: OnRootMismatch
        seccompProfile: { type: RuntimeDefault }
    service:
      app:
        controller: onepassword-connect
        ports:
          http:
            port: *apiPort
    persistence:
      config:
        type: emptyDir
        globalMounts:
          - path: *configDir
```

I grabbed this from a home-ops repo. It's more complicated than I'm use to so we'll see how it goes.

### GO LIVE

This is all happening from a hotel room at disney world so hopefully it's magical.

Seems OK but WSL is broken, the krew stuff I installed only works on the terminal I installed it into. Was simply because I forgot to update `.bashrc`.

## Reloader

Before I go further, since I'll likely need to tweak secrets to get this to work, setting up reloader seems like a smart move.

Easy enough to drop in, will revisit later when I actually test it.

## External Secrets

Now time for the payout! This went very smooth.

First I made two flux kustomizations, one for the external secret app and the other for the 1Password store:

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/kustomize.toolkit.fluxcd.io/kustomization_v1.json
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: &app external-secrets
  namespace: flux-system
spec:
  targetNamespace: external-secrets
  commonMetadata:
    labels:
      app.kubernetes.io/name: *app
  interval: 15m
  retryInterval: 1m
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./kubernetes/main/apps/system/secrets/external/external-secrets/app
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: external-secrets
      namespace: external-secrets
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/kustomize.toolkit.fluxcd.io/kustomization_v1.json
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: &app external-secrets-stores
  namespace: flux-system
spec:
  targetNamespace: external-secrets
  commonMetadata:
    labels:
      app.kubernetes.io/name: *app
  interval: 15m
  retryInterval: 1m
  timeout: 2m
  prune: true
  wait: true
  dependsOn:
    - name: external-secrets
    - name: onepassword-connect
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./kubernetes/main/apps/system/secrets/external/external-secrets/stores
```

The store is the only interesting thing here - it points to my 1Password integration and tells it to prioritize the `HaynesKube` vault first. If I needed to access other vaults I could add them in descending order of priority.

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/external-secrets.io/clustersecretstore_v1beta1.json
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: onepassword-connect
spec:
  provider:
    onepassword:
      connectHost: http://onepassword-connect.external-secrets.svc.cluster.local
      vaults:
        HaynesKube: 1
      auth:
        secretRef:
          connectTokenSecretRef:
            name: onepassword-connect-secret
            key: token
            namespace: external-secrets
```

The app itself has a simple HelmRelease with some serviceMonitors that I need to dig into more later:

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/helm.toolkit.fluxcd.io/helmrelease_v2.json
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-secrets
spec:
  interval: 30m
  chart:
    spec:
      chart: external-secrets
      version: 0.10.3
      sourceRef:
        kind: HelmRepository
        name: external-secrets
        namespace: flux-system
  install:
    remediation:
      retries: 3
  upgrade:
    cleanupOnFail: true
    remediation:
      strategy: rollback
      retries: 3
  dependsOn:
    - name: onepassword-connect
      namespace: external-secrets
  values:
    installCRDs: true
    serviceMonitor:
      enabled: true
      interval: 1m
    webhook:
      serviceMonitor:
        enabled: true
        interval: 1m
    certController:
      serviceMonitor:
        enabled: true
        interval: 1m
```

To test it I added a password in 1Password by following [this outdated guide](https://external-secrets.io/main/provider/1password-automation/) named `emqx` and added a secret using two key,value pairs I added under a "section" in the password while leaving the password itself blank (bit conveluted).

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/external-secrets.io/externalsecret_v1beta1.json
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: emqx
  namespace: iot-services
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: onepassword-connect
  target:
    name: emqx-secret
    template:
      engineVersion: v2
      data:
        EMQX_DASHBOARD__DEFAULT_USERNAME: "{{ .EMQX_DASHBOARD__DEFAULT_USERNAME }}"
        EMQX_DASHBOARD__DEFAULT_PASSWORD: "{{ .EMQX_DASHBOARD__DEFAULT_PASSWORD }}"
  dataFrom:
    - extract:
        key: emqx
```

This all seemed to work and wil be further tested once I set up emqx later:

```
thaynes@HaynesHyperion:~$ k describe secret -n iot-services emqx-secret
Name:         emqx-secret
Namespace:    iot-services
Labels:       reconcile.external-secrets.io/created-by=432761e62965f5dfb77cc49e7a6d7120
Annotations:  reconcile.external-secrets.io/data-hash: 25dfff24bc564e73bdc4589dd066e6e6

Type:  Opaque

Data
====
EMQX_DASHBOARD__DEFAULT_PASSWORD:  20 bytes
EMQX_DASHBOARD__DEFAULT_USERNAME:  8 bytes
```