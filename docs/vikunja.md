---
title: Waydroid
permalink: /docs/vikunka/
---

* Chart Page: https://truecharts.org/charts/stable/vikunja/
* Helm Repo: oci://tccr.io/truecharts/vikunja
* Values: https://github.com/truecharts/charts/blob/master/charts/stable/vikunja/values.yaml
* Common Values: https://github.com/truecharts/library-charts/blob/main/library/common/values.yaml

## Flux Files

> **ERROR** These are way stale

### Vikunja Namespace

`namespace-vikunja.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vikunja
```

### HelmRepository

`ocirepository-truecharts-vikunja.yaml`
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: truecharts-vikunja
  namespace: flux-system
spec:
  interval: 15m
  url: oci://tccr.io/truecharts/vikunja
```

### Vikunja Kustomization

`kustomization-vikunja.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: vikunja
  namespace: flux-system
spec:
  interval: 15m
  path: ./vikunja
  prune: true
  timeout: 2m
  sourceRef:
    kind: GitRepository
    name: flux-system
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: vikunja
      namespace: vikunja
```

### Vikunja HelmRelease

`helmrelease-vikunja.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: vikunja
  namespace: haynes-intelligence
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.40.107
spec:
  chart:
    spec:
      chart: vikunja
      version: 15.2.x # https://truecharts.org/charts/stable/vikunja/
      sourceRef:
        kind: OCIRepository
        name: truecharts-vikunja
        namespace: flux-system
  interval: 15m
  timeout: 5m
  releaseName: vikunja
  values:
    # Vikunja defaults found here https://github.com/truecharts/charts/blob/master/charts/stable/vikunja/values.yaml
    # Common defaults found here https://github.com/truecharts/library-charts/blob/main/library/common/values.yaml
    global:
      fallbackDefaults:
        serviceType: LoadBalancer
        storageClass: ceph-rbd
        pvcSize: 10Gi
        vctSize: 10Gi
    vikunja:
      service:
        motd: Welcome to the Haynes House Vikunja Instance!
      auth:
        local:
          enabled: false
        openid:
          enabled: true
          providers:
            - name: "Authentik"
              authurl: "https://authentik.example.com/application/o/vikunja/"
              logouturl: "https://authentik.example.com/application/o/vikunja/end-session/"
              clientid: "GETFROMAUTHENTIK"
              clientsecret: "GETFROMAUTHENTIK"
```

## Debug

First I had a hell of a time with it being from an OCI repository but that was mainly my namespace not having the `type` field and me thinking I needed an `OCIRepository` type over a `HelmRepository` type.

```
flux-system                             main@sha1:4572278f      False           False   kustomize build failed: accumulating resources: accumulation err='merging resources from './kustomizations/kustomization-vi
kunja.yaml': may not add resource with an already registered id: Kustomization.v1beta2.kustomize.toolkit.fluxcd.io/authentik.flux-system': must build at directory: '/tmp/kustomization-3833245915/bootstrap/kustom
izations/kustomization-vikunja.yaml': file is not directory
```

[This post](https://www.reddit.com/r/truecharts/comments/1ejfqdf/how_to_use_truecharts_oci_registry_with_flux/) was fresh on reddit and the `HelmRepository` posted did end up getting me futher. However, the next road block was major:

```
  Warning  InstallFailed     12s (x7 over 66s)  helm-controller  Helm install failed for release vikunja/vikunja with chart vikunja@15.2.8: unable to build kubernetes objects from release manifest: resource mapping not found for name: "vikunja-cnpg-main" namespace: "vikunja" from "": no matches for kind "Cluster" in version "postgresql.cnpg.io/v1"
ensure CRDs are installed first
```

Now this could turn into whole thing. These `truecharts` are quite impressive, and offer a lot of boilerplate while recommending it's all in flux. Way more content but less depth than the initial tutorials I followed. 

## Cloudnative-pg

[This chart](https://truecharts.org/charts/system/cloudnative-pg/) is for [this thing](https://cloudnative-pg.io/) that lets you run PostgreSQL "the kubernetes way". I've got Authentik using that already so who knows what way but I'll bit.

We will go with the [truechart](https://github.com/truecharts/charts/tree/master/charts/system/cloudnative-pg) for this since I'm trying to unblock another truechart.

### Flux CRDs

We will keep a single `HelmRepository` defined for all truechart charts but we'll need the rest for this.

#### Cloudnative-pg Namespace

I'm not really sure where this should live since it's kinda system wide related. For now it can get it's own.

`namespace-cloudnative-pg.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cloudnative-pg
```

#### Cloudnative-pg Kustomization

`kustomization-cloudnative-pg.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: cloudnative-pg
  namespace: cloudnative-pg
spec:
  interval: 15m
  retryInterval: 1m  
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./cloudnative-pg
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: cloudnative-pg
      namespace: cloudnative-pg
```

#### Cloudnative-pg HelmRelease

`helmrelease-cloudnative-pg.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: cloudnative-pg
  namespace: cloudnative-pg
spec:
  chart:
    spec:
      chart: cloudnative-pg
      version: 8.1.1 # https://github.com/truecharts/charts/blob/master/charts/system/cloudnative-pg/Chart.yaml
      sourceRef:
        kind: HelmRepository
        name: truecharts
        namespace: flux-system
  interval: 15m
  timeout: 5m
  #releaseName: cloudnative-pg
  values: values.yaml
```

Those got me in!

![vikunja-welcome]({{ site.url }}/images/web/vikunja-welcome.png)

It felt a lot like JIRALight where I could add mini stories (task) and link them in different ways. Might not catch on in my household but I'm still going to get it online!

## Getting it Online

### Authentik Config

For this application we wil use the wizard:

![authentik-wizard]({{ site.url }}/images/web/authentik-wizard.png)

First we name the application:

![authentik-wizard-new-app]({{ site.url }}/images/web/authentik-wizard-new-app.png)

Then we select ODIC:

![authentik-wizard-odic]({{ site.url }}/images/web/authentik-wizard-odic.png)

Next screen is a bit more involved:

> **WARNING** I forgot to select a Signing Key. I use my Cloudflare Origin Cert here to avoid weird errors at work.

![authentik-wizard-odic-provider-settings]({{ site.url }}/images/web/authentik-wizard-odic-provider-settings.png)

If you expand `Protocol Settings` you will see `Client ID` and `Client Secret` which will be needed soon for configuring the helm chart. 

Last time I set this up I did not need to specify a redirect URL but I also specified a configuration URL in `Open WebUI`'s config. This time I see it's just authurl and logouturl so I am going to manually set the redirect to "https://vikunja.example.com/oauth/auth/openid/" per [vikunja's docs](https://vikunja.io/docs/openid-example-configurations#authentik)

You can select explicit or implicit concent for the `Authorization flow`. I will likely do implicit but want to see what explicit does so I've selected that initially (my other apps use implicit).

### Helm Auth Config

#### DNSEndpoint

`dnsendpoint-externaldns-vikunja.yaml`
```yaml
apiVersion: externaldns.k8s.io/v1alpha1
kind: DNSEndpoint
metadata:
  name: "vikunja.example.com"
  namespace: external-dns
spec:
  endpoints:
  - dnsName: "vikunja.example.com"
    recordTTL: 180
    recordType: CNAME
    targets:
    - "example.com"
```

#### IngresRoute

`ingressroute-vikunja.yaml`
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: vikunja
  namespace: traefik
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`www.vikunja.example.com`)
      kind: Rule
      services:
        - kind: Service
          name: vikunja
          port: 10220
          namespace: vikunja
      middlewares:
        - name: default-headers
          namespace: traefik
    - kind: Rule
      match: Host(`vikunja.example.com`) && PathPrefix(`/`)
      services:
        - kind: Service
          name: vikunja
          port: 10220
          namespace: vikunja
      middlewares:
        - name: default-headers
          namespace: traefik
  tls:
    secretName: certificate-example.com
```

#### Sealed Secret

`vikunja-auth-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vikunja-provider-credentials
  namespace: vikunja

type: Opaque
stringData:
  id: "GETFROMAUTHENTIK"
  secret: "GETFROMAUTHENTIK"
```

Seal it up:

```bash
cat vikunja-auth-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-vikunja-auth-secret.yaml
```

#### Auth Values

> **TODO** I couldn't figure out how to pipe these as secrets since they can't be set to environment variables. [Here](https://community.vikunja.io/t/configure-openid-via-environment/628/13) is a post where people complain about it.

```yaml
      auth:
        local:
          enabled: false
        openid:
          enabled: true
          providers:
            - name: "Authentik"
              authurl: "https://authentik.example.com/application/o/vikunja/"
              logouturl: "https://authentik.example.com/application/o/vikunja/end-session/"
              clientid: "GETFROMAUTHENTIK"
              clientsecret: "GETFROMAUTHENTIK"
```

I missed some [authentik doc](https://docs.goauthentik.io/integrations/services/vikunja/) on how to set this shit up. They reminded me I missed Cloudflare to sign the tokens but the rest were misleading. The [vijunka](https://vikunja.io/docs/openid-example-configurations#authentik) ones were better but still off a bit, you don't need `redirecturl` cause it figures it out on it's own.

## Moving to Secrets

> **WARNING** You can't with this chart. At least not that I've found. This is just me trying and at least learning how to set env vars w/ truecharts.

Next I am trying to pass secrets for the openid secret but the only way I see this is possible is via environment variables. Few tries yields:

```
  Warning  UpgradeFailed  43s  helm-controller  Helm upgrade failed for release vikunja/vikunja with chart vikunja@15.2.8: template: vikunja/templates/common.yaml:8:3: executing "vikunja/templates/common.yaml" at <include "tc.v1.common.loader.apply" .>: error calling include: template: vikunja/charts/common/templates/loader/_apply.tpl:32:6: executing "tc.v1.common.loader.apply" at <include "tc.v1.common.spawner.workload" .>: error calling include: template: vikunja/charts/common/templates/spawner/_workload.tpl:49:12: executing "tc.v1.common.spawner.workload" at <include "tc.v1.common.class.deployment" (dict "rootCtx" $ "objectData" $objectData)>: error calling include: template: vikunja/charts/common/templates/class/_deployment.tpl:53:10: executing "tc.v1.common.class.deployment" at <include "tc.v1.common.lib.workload.pod" (dict "rootCtx" $rootCtx "objectData" $objectData)>: error calling include: template: vikunja/charts/common/templates/lib/workload/_pod.tpl:57:8: executing "tc.v1.common.lib.workload.pod" at <include "tc.v1.common.lib.pod.containerSpawner" (dict "rootCtx" $rootCtx "objectData" $objectData)>: error calling include: template: vikunja/charts/common/templates/lib/pod/_containerSpawner.tpl:33:10: executing "tc.v1.common.lib.pod.containerSpawner" at <include "tc.v1.common.lib.pod.container" (dict "rootCtx" $rootCtx "objectData" $container)>: error calling include: template: vikunja/charts/common/templates/lib/pod/_container.tpl:59:8: executing "tc.v1.common.lib.pod.container" at <include "tc.v1.common.lib.container.env" (dict "rootCtx" $rootCtx "objectData" $objectData)>: error calling include: template: vikunja/charts/common/templates/lib/container/_env.tpl:12:8: executing "tc.v1.common.lib.container.env" at <include "tc.v1.common.helper.container.envDupeCheck" (dict "rootCtx" $rootCtx "objectData" $objectData "source" "env" "key" $k)>: error calling include: template: vikunja/charts/common/templates/helpers/_envDupeCheck.tpl:15:43: executing "tc.v1.common.helper.container.envDupeCheck" at <$key>: wrong type for value; expected string; got int
```

But [TrueCharts env docs](https://truecharts.org/common/container/env/) point to my formatting being off where `key` is set.

Now this works:

```yaml
    workload:
      main:
        podSpec:
          containers:
            frontend:
              env:
                VIKUNJA_SERVICE_MOTD: "TEST ENV SET IN CHART"
```

But maybe not the right container because the motd is still wrong:

```
thaynes@kubem01:~/workspace/secret-sealing$ k exec -it -n vikunja pod/vikunja-54bf6d85-gzstd -- env
Defaulted container "vikunja-frontend" out of: vikunja-frontend, vikunja, vikunja-proxy, vikunja-system-cnpg-wait (init), vikunja-system-redis-wait (init)
VIKUNJA_SERVICE_MOTD=TEST ENV SET IN CHART
```

Not sure how to pick the container I `exec` in but google says `--container or -c`.

```
k exec -it -n vikunja pod/vikunja-58d5794fdd-fgs4n -c vikunja -- env
VIKUNJA_SERVICE_MOTD=TEST ENV SET IN CHART
```

Looks good!

![vikunja-env-works]({{ site.url }}/images/web/vikunja-env-works.png)

Now to try the secrets:

```yaml
    workload:
      main:
        podSpec:
          containers:
            main:
              env:
                VIKUNJA_AUTH_OPENID_CLIENTID:
                  secretKeyRef:
                    name: vikunja-provider-credentials
                    key: id
                VIKUNJA_AUTH_OPENID_CLIENTSECRET:
                  secretKeyRef:
                    name: vikunja-provider-credentials
                    key: secret
```

But no dice

```
  Warning  UpgradeFailed  67s  helm-controller  Helm upgrade failed for release vikunja/vikunja with chart vikunja@15.2.8: execution error at (vikunja/templates/common.yaml:8:3): Container - Expected in [env] the referenced Secret [vikunja-provider-credentials] to be defined
```

## Values Reference 

The chart specific values may be found [here](https://github.com/truecharts/charts/blob/master/charts/stable/vikunja/values.yaml) while common template values for all the truecharts are [here](https://github.com/truecharts/library-charts/blob/main/library/common/values.yaml).

```yaml
image:
  repository: vikunja/api
  tag: 0.22.1@sha256:c9415431e6235229302bb8f9ee6660b74c24859d1e8adbc4a3e25bd418604b57
  pullPolicy: IfNotPresent
frontendImage:
  repository: vikunja/frontend
  tag: 0.22.1@sha256:f0223d441997fe29c377d0b476dc4bb2fc091b44b9c24d76b1b88c213df520c5
  pullPolicy: IfNotPresent
nginxImage:
  repository: nginx
  tag: 1.27.1@sha256:1540e37eebb9abc5afa4256de1bade6542d50bf69b61b1dd855cb7804aaaf444
workload:
  main:
    podSpec:
      containers:
        main:
          probes:
            liveness:
              type: http
              port: 3456
              path: /health
            readiness:
              type: http
              port: 3456
              path: /health
            startup:
              type: http
              port: 3456
              path: /health
        frontend:
          enabled: true
          imageSelector: frontendImage
          probes:
            liveness:
              type: http
              port: 80
            readiness:
              type: http
              port: 80
            startup:
              type: http
              port: 80
        proxy:
          enabled: true
          imageSelector: nginxImage
          probes:
            liveness:
              type: http
              port: "{{ .Values.service.main.ports.main.port }}"
            readiness:
              type: http
              port: "{{ .Values.service.main.ports.main.port }}"
            startup:
              type: http
              port: "{{ .Values.service.main.ports.main.port }}"
vikunja:
  service:
    jwtttl: 259200
    jwtttllong: 2592000
    motd: Welcome to your new Vikunja instance
    frontendurl: http://localhost:10220
    maxitemsperpage: 50
    enablecaldav: true
    enablelinksharing: true
    enableregistration: true
    enabletaskattachments: true
    enabletaskcomments: true
    enabletotp: true
    enableemailreminders: true
    enableuserdeletion: true
    maxavatarsize: 1024
  cors:
    enabled: true
    origins: []
    maxage: 0
  ratelimit:
    enabled: false
    kind: user
    period: 60
    limit: 100
  files:
    maxsize: 20MB
  avatar:
    gravatarexpiration: 3600
  legal:
    imprinturl: ""
    privacyurl: ""
  mailer:
    enabled: false
    host: ""
    port: 587
    authtype: plain
    username: ""
    password: ""
    fromemail: ""
    skiptlsverify: false
    forcessl: true
    queuelength: 100
    queuetimeout: 30
  log:
    enabled: true
    path: /app/vikunja/logs
    standard: stdout
    level: INFO
    database: "off"
    databaselevel: WARNING
    http: stdout
    echo: "off"
    events: stdout
    eventslevel: INFO
  defaultsettings:
    avatar_provider: initials
    avatar_file_id: 0
    email_reminders_enabled: false
    discoverable_by_name: false
    discoverable_by_email: false
    overdue_tasks_reminders_enabled: true
    overdue_tasks_reminders_time: "9:00"
    default_list_id: 0
    week_start: 0
    language: ""
    timezone: ""
  backgrounds:
    enabled: true
    providers:
      upload:
        enabled: true
      unsplash:
        enabled: false
        accesstoken: ""
        applicationid: ""
  auth:
    local:
      enabled: true
    openid:
      enabled: false
      redirecturl: ""
      providers: []
      # - name: ""
      #   authurl: ""
      #   logouturl: ""
      #   clientid: ""
      #   clientsecret: ""
  migration:
    todoist:
      enable: false
      clientid: ""
      clientsecret: ""
      redirecturl: ""
    trello:
      enable: false
      key: ""
      redirecturl: ""
    microsofttodo:
      enable: false
      clientid: ""
      clientsecret: ""
      redirecturl: ""
service:
  main:
    ports:
      main:
        port: 10220
persistence:
  files:
    enabled: true
    mountPath: /app/vikunja/files
  vikunja-nginx:
    enabled: true
    noMount: true
    type: configmap
    objectName: nginx-config
    targetSelector:
      main:
        proxy:
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx-config
  vikunja-config:
    enabled: true
    type: secret
    objectName: config
    targetSelector:
      main:
        main:
          subPath: config.yaml
          mountPath: /etc/vikunja/config.yaml
metrics:
  main:
    enabled: true
    type: "servicemonitor"
    prometheusRule:
      enabled: false
cnpg:
  main:
    enabled: true
    user: vikunja
    database: vikunja
redis:
  enabled: true
  includeCommon: true
portal:
  open:
    enabled: true
securityContext:
  container:
    readOnlyRootFilesystem: false
    runAsNonRoot: false
    runAsUser: 0
    runAsGroup: 0
    capabilities:
      add:
        - NET_BIND_SERVICE
updated: true
```