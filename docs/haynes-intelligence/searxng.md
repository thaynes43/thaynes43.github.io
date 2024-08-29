---
title: Searxng
permalink: /docs/haynes-intelligence/searxng/
---

[Searxng](https://docs.searxng.org/) acts as a web browser end point for Open WebUI. It also looks easy to run via helm so it's worth a shot. It integrates with Open WebUI which is documented [here](https://docs.openwebui.com/tutorial/web_search/).

## CRDs

Namespace will be `haynes-intelligence`. 

### HelmRepository

`helmrepository-searxng.yaml`
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: searxng
  namespace: flux-system
spec:
  interval: 15m
  url: https://charts.searxng.org
```

### Kustomization

I will group this with `Open WebUI` via `kustomization-haynes-intelligence.yaml`

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: searxng
  namespace: flux-system
spec:
  dependsOn:
    - name: sealed-secrets
    - name: haynes-intelligence-config
  interval: 15m
  retryInterval: 1m  
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: searxng
      namespace: haynes-intelligence
  path: ./haynes-intelligence/searxng
```

### Secret

```bash
  kubectl create secret generic searxng-secret \
  --namespace haynes-intelligence \
  --dry-run=client \
  --from-literal=secret="REDACTED" -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem --format yaml \
  > sealedsecret-searxng-secret.yaml
```

`sealedsecret-searxng-secret.yaml`
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: searxng-secret
  namespace: haynes-intelligence
spec:
  encryptedData:
    secret: REDACTED
  template:
    metadata:
      creationTimestamp: null
      name: searxng-secret
      namespace: haynes-intelligence
```

### HelmRelease

`helmrelease-searxng.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: searxng
  namespace: haynes-intelligence
spec:
  chart:
    spec:
      chart: searxng
      version: 1.0.x
      sourceRef:
        kind: HelmRepository
        name: searxng
        namespace: flux-system
  interval: 15m
  timeout: 5m
  releaseName: searxng
  values: 
    # config https://github.com/searxng/searxng/blob/master/searx/settings.yml
    # values https://github.com/searxng/searxng-helm-chart/blob/main/searxng/values.yaml
    # common https://github.com/k8s-at-home/library-charts/blob/main/charts/stable/common/values.yaml
    service:
      main:
        type: LoadBalancer
        annotations:
          metallb.universe.tf/loadBalancerIPs: 192.168.40.116
    env:
      INSTANCE_NAME: "haynes-intelligence-searxng"
      BASE_URL: "https://searxng.haynesnetwork.com/"
      AUTOCOMPLETE: "google"
      SEARXNG_SECRET: 
        valueFrom:
          secretKeyRef:
            name: searxng-secret
            key: secret
    persistence:
      limiter:
        enabled: true
        type: configMap
        name: configmap-searxng-limiter
        mountPath: /etc/searxng/limiter.toml
        subPath: limiter.toml
    searxng:
      config:
        use_default_settings: true
        server:
          secret_key: "$SEARXNG_SECRET"
          # See https://docs.searxng.org/admin/searx.limiter.html
          limiter: true
        search:
          formats:
            - html
            - json
        redis:
          url: redis://searxng-redis:6379
    # https://github.com/pascaliske/helm-charts/tree/main/charts/redis
    redis:
      enabled: true
      persistentVolumeClaim:
        storageClassName: ceph-rbd
```

## Try It!

First shot? 

```
  Warning  InstallFailed     2m13s (x10 over 8m36s)  helm-controller  Helm install failed for release haynes-intelligence/searxng with chart searxng@1.0.0: template: searxng/templates/common.yaml:17:3: executing "searxng/templates/common.yaml" at <include "common.all" .>: error calling include: template: searxng/charts/common/templates/_all.tpl:39:10: executing "common.all" at <include "common.deployment" .>: error calling include: template: searxng/charts/common/templates/_deployment.tpl:52:10: executing "common.deployment" at <include "common.controller.pod" .>: error calling include: template: searxng/charts/common/templates/lib/controller/_pod.tpl:69:12: executing "common.controller.pod" at <include "common.controller.volumes" .>: error calling include: template: searxng/charts/common/templates/lib/controller/_volumes.tpl:64:64: executing "common.controller.volumes" at <.Values.persistence.type>: nil pointer evaluating interface {}.persistence
```

Nope. I missed the [common values](https://github.com/k8s-at-home/library-charts/blob/main/charts/stable/common/values.yaml). 

Second shot, hell yeah:

![searxng-screen]({{ site.url }}/images/web/searxng-screen.png)

Just needed to adjust `persistence` a bit.

## Integrate with Open WebUI

We can point Open WebUI at `http://192.168.40.116:8080/search?q=<query>` and query this new service.

However, it throws a fit:

```
403 Client Error: FORBIDDEN for url: http://192.168.40.116:8080/search?q=WAWADSdad&format=json&pageno=1
```

Same thing with public instance `https://ooglester.com/search?q=<query>`:

```
403 Client Error: Forbidden for url: https://ooglester.com/search?q=What+does+SearXNG+do%3F+&format=json&pageno=1&safesearch=1&language=en-US&time_range=&categories=&theme=simple&image_proxy=0
```

Fix seems straightforward, just need to [enable json](https://github.com/open-webui/open-webui/issues/2824). Also noticed /config isn't used, they bind this in the [docker compose](https://github.com/searxng/searxng-docker/blob/master/docker-compose.yaml):

Add this for the config:

```yaml
    searxng:
      config:
        use_default_settings: true
        server:
          secret_key: "$SEARXNG_SECRET"
        search:
          formats:
            - html
            - json
```

And match this for my persistance:

__docker compose__
```yaml
   volumes:
      - ./searxng:/etc/searxng:rw
```

__helm values__
```yaml
    persistence:
      config:
        enabled: true
        mountPath: /etc/searxng
        storageClass: ceph-rbd
        accessMode: ReadWriteOnce
        size: 2Gi
```

## Limiter

Since we got redis working we should be able to enable the [limiter](https://docs.searxng.org/admin/searx.limiter.html) with ease. This protects against being banned or CAPACHAUD by the search engines because they think you are a bot. 

> **WARNING** This helm chart did not deploy a `limiter.toml` file so I had to jump through hoops to get shit to work.

After much head bashing I determined I needed to map a ConfigMap where the missing file was needed.

### Create ConfigMap

> **WARNING** This ConfigMap must be created FIRST or an empty directory will get mapped to the pod. I did this by creating a new Kustomization for the searxng Kustomization to depend on

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-searxng-limiter
  namespace: haynes-intelligence
data:
  limiter.toml: |
    # This configuration file updates the default configuration file
    # See https://github.com/searxng/searxng/blob/master/searx/limiter.toml
    [botdetection.ip_limit]
    link_token = true
```

### Create Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: haynes-intelligence-config
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
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: ConfigMap
      name: configmap-searxng-limiter
      namespace: haynes-intelligence
  path: ./haynes-intelligence/config
```

### Adjust Chart

The config map can be mounted with the `persistence` block in the chart:

```yaml
    persistence:
      limiter:
        enabled: true
        type: configMap
        name: configmap-searxng-limiter
        mountPath: /etc/searxng/limiter.toml
        subPath: limiter.toml
```

This took me a ton of fiddling around to figure out, the values package for the common chart said you could use a configMap but not how to use it.

### Verify 

```bash
/etc/searxng # ls
limiter.toml  settings.yml  uwsgi.ini
```

File finally there and not a directory!

```bash
/etc/searxng # cat limiter.toml 
# This configuration file updates the default configuration file
# See https://github.com/searxng/searxng/blob/master/searx/limiter.toml
[botdetection.ip_limit]
link_token = true/etc/searxng # 
```

Contents good!

```bash
/etc/searxng # cat settings.yml 
redis:
  url: redis://searxng-redis:6379
search:
  formats:
  - html
  - json
server:
  limiter: true
  secret_key: $SEARXNG_SECRET
```

Other file not screwed by all this!

## GO LIVE

Now we just need a `DNSEndpoint`, `IngressRoute`, and some Authentik fiddling to get this off the ground! Only risk is if Authentik blocks the API calls.

The process is well documented in my first tutorial located [here](https://hayneslab.net/tutorials/authentik-proxy-provider/).

### DNSEndpoint

`dnsendpoint-externaldns-searxng.yaml`
```yaml
apiVersion: externaldns.k8s.io/v1alpha1
kind: DNSEndpoint
metadata:
  name: "searxng.haynesnetwork.com"
  namespace: external-dns
spec:
  endpoints:
  - dnsName: "searxng.haynesnetwork.com"
    recordTTL: 180
    recordType: CNAME
    targets:
    - "haynesnetwork.com"
```

### Ingress Route

`ingressroute-searxng.yaml`
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: searxng
  namespace: traefik
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`www.searxng.haynesnetwork.com`)
      kind: Rule
      services:
        - kind: Service
          name: searxng
          port: 8080
          namespace: haynes-intelligence
      middlewares:
        - name: authentik-auth-proxy
          namespace: traefik
    - kind: Rule
      match: Host(`searxng.haynesnetwork.com`) && PathPrefix(`/`)
      services:
        - kind: Service
          name: searxng
          port: 8080
          namespace: haynes-intelligence
      middlewares:
        - name: authentik-auth-proxy
          namespace: traefik
  tls:
    secretName: certificate-haynesnetwork.com
```