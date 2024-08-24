---
title: Exportarr
permalink: /docs/exportarr/
---

[Exportarr](https://github.com/onedr0p/exportarr) collects metrics from the arr stack and sends them along. We should be able to make a helm chart from [these CRDs](https://github.com/onedr0p/exportarr/blob/master/examples/kubernetes/sonarr-exporter.yaml) and then just re-use it for every exporter.

> **TODO** I can also just save time and use the manifest there, maybe circle back...

## CRDs

## Kustomization

This needs to be it's own Kustomization so it starts up after monitoring and the service it's exporting data from. They can all live in a folder under media management.

`kustomization-media-management.yaml`
```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: exportarr
  namespace: flux-system
spec:
  dependsOn:
    - name: sealed-secrets
    - name: monitoring-controllers # TODO should this depend on arrs? 
  interval: 15m
  retryInterval: 1m
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./media-management/exportarr
  healthChecks: # TODO will need a check per service
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: Service
      name: radarr-exporter
      namespace: media-management
```

## Radarr Exporter Manifest

Since Sonarr has one, but needs to be migrated later, I'll start with Radarr. The `truechart` threw the exporter in the same namespace as Sonarr but this here has me going with `monitoring` which does make sense. They can have their own Kustomization that is dependent on the monitoring stack

### Secret

Create a secret with the api key:

`radarr-api-key.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: radarr-api-key
  namespace: monitoring

type: Opaque
stringData:
  api-key: "RADARR_API_KEY"
```

Seal it up:

```bash
cat radarr-api-key.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-radarr-api-key.yaml
```

Drop it in:

`sealedsecret-radarr-api-key.yaml`
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: radarr-api-key
  namespace: monitoring
spec:
  encryptedData:
    api-key: HUGESECRET
  template:
    metadata:
      creationTimestamp: null
      name: radarr-api-key
      namespace: monitoring
    type: Opaque
```

### Manifest

`manifest-radarr-exporter.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: radarr-exporter
  namespace: monitoring
  labels:
    app.kubernetes.io/name: radarr-exporter
    app.kubernetes.io/instance: radarr-exporter
spec:
  clusterIP: None
  selector:
    app.kubernetes.io/name: radarr-exporter
    app.kubernetes.io/instance: radarr-exporter
  ports:
    - name: monitoring
      port: 9707
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: radarr-exporter
  namespace: monitoring
  labels:
    app.kubernetes.io/name: radarr-exporter
    app.kubernetes.io/instance: radarr-exporter
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: radarr-exporter
      app.kubernetes.io/instance: radarr-exporter
  endpoints:
    - port: monitoring
      interval: 4m
      scrapeTimeout: 90s
      path: /metrics
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: radarr-exporter
  namespace: monitoring
  labels:
    app.kubernetes.io/name: radarr-exporter
    app.kubernetes.io/instance: radarr-exporter
spec:
  replicas: 1
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: radarr-exporter
      app.kubernetes.io/instance: radarr-exporter
  template:
    metadata:
      labels:
        app.kubernetes.io/name: radarr-exporter
        app.kubernetes.io/instance: radarr-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "monitoring"
    spec:
      containers:
        - name: radarr-exporter
          image: ghcr.io/onedr0p/exportarr:v1.5.3
          imagePullPolicy: IfNotPresent
          args:
            - radarr
          env:
            - name: PORT
              value: "9707"
            - name: URL
              value: "http://radarr.media-management.svc.cluster.local:7878"
            - name: APIKEY
              valueFrom:
                secretKeyRef:
                  name: radarr-api-key
                  key: api-key
          ports:
            - name: monitoring
              containerPort: 9707
          livenessProbe:
            httpGet:
              path: /healthz
              port: monitoring
            failureThreshold: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz
              port: monitoring
            failureThreshold: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
```