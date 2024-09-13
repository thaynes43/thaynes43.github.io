---
title: Cloudflare-DDNS
permalink: /docs/cloudflare-ddns/
---

## Working it out

There are a whole bunch of these services on Github all named the same thing but I am running [timothymiller](https://github.com/timothymiller/cloudflare-ddns) on unRAID so will go with that. He seems to have it together and even pushes his own docker image.

I can also use the [app-template chart](https://bjw-s.github.io/helm-charts/docs/app-template/) and just pass environment variables from sealed secrets. No reason to write a chart __yet__. 

## Flux Files

### Namespace

More cloudflare stuff may come up so I am just going to drop the ddns part. Could also combine with `eternal-dns` into one namespace but changing namespaces kinda sucks.

`namespace-cloudflare.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cloudflare
```

### HelmRepository

Fortunately we can use `https://bjw-s.github.io/helm-charts` which is already created!

### Kustomization

`kustomization-cloudflare.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: cloudflare-ddns
  namespace: flux-system
spec:
  dependsOn:
    - name: sealed-secrets
  interval: 15m
  retryInterval: 1m
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./cloudflare
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: cloudflare-ddns
      namespace: cloudflare
```

### Deployment

No reason to use a chart for this one and the author has supplied a deployment.

`deployment-cloudflare-ddns.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflare-ddns
  namespace: cloudflare
spec:
  selector:
    matchLabels:
      app: cloudflare-ddns

  template:
    metadata:
      labels:
        app: cloudflare-ddns

    spec:
      containers:
        - name: cloudflare-ddns
          image: timothyjmiller/cloudflare-ddns:latest # https://hub.docker.com/r/timothyjmiller/cloudflare-ddns
          resources:
            limits:
              memory: '32Mi'
              cpu: '50m'
          env:
            - name: CONFIG_PATH
              value: '/etc/cloudflare-ddns/'
          volumeMounts:
            - mountPath: '/etc/cloudflare-ddns'
              name: config-cloudflare-ddns
              readOnly: true
      volumes:
        - name: config-cloudflare-ddns
          secret:
            secretName: config-cloudflare-ddns
```

### Sealed Secret

I can seal the entire config in a secret AND it's the same config I have on unRAID:

`config.json`
```json
{
    "cloudflare": [
      {
        "authentication": {
          "api_token": "REDACTED"
        },
        "zone_id": "REDACTED",
        "subdomains": [
          {
            "name": "",
            "proxied": true
          },
          {
            "name": "www",
            "proxied": true
          }
        ]
      }
    ],
    "a": true,
    "aaaa": false,
    "purgeUnknownRecords": false,
    "ttl": 300
}
```

Make that a secret:

```bash
kubectl create secret generic config-cloudflare-ddns --from-file=config.json --dry-run=client -oyaml -n cloudflare > config-cloudflare-ddns-secret.yaml
```

Seal it up:

```bash
cat config-cloudflare-ddns-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-config-cloudflare-ddns-secret.yaml
```

`sealedsecret-config-cloudflare-ddns.yaml`
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: config-cloudflare-ddns
  namespace: cloudflare
spec:
  encryptedData:
    config.json: HUGESECRET
  template:
    metadata:
      creationTimestamp: null
      name: config-cloudflare-ddns
      namespace: cloudflare
```

## Test It!

Things are looking good first try! Or maybe it just sleeps for five minutes before validating the config is right by reaching out to cloudflare...

```
2024-08-23 19:50:30	🕰️ Updating IPv4 (A) records every 300 seconds
```

We will check back later. unRAID had some errors in the logs but it was hard to tell over what span of time. The token isn't in use by anything else since external-dns is using the API key so I don't have a way to verify it's good.