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

Now time for the payout! 

