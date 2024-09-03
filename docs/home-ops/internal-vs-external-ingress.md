---
title: Internal vs. External Ingress
permalink: /docs/home-ops/internal-vs-external-ingress/
---

## Clean up DNS w/ Two Traefiks 

We may want two instances of traefik, one for internal and one for external like described [here](https://www.reddit.com/r/Traefik/comments/10yzfr3/helm_traefik_how_can_we_create_an_internal_and/) but we may be able to just use external dns regex's for what is exposed. [here](https://community.traefik.io/t/multiple-traefik-deployment-in-k8s-via-helm/5141) is another post about `labelSelector`

### Adding traefik-external Ingress Class

I had been using just one ingress class but this will be the main decider for what is external and internal. Changing it was fairly easy:

* Define in traefik providers `ingressClass: traefik-external`
* Define in `IngressRoutes`: `kubernetes.io/ingress.class: traefik-external`
* Define in `ingress` helm values

After going for it Authentik is complaining about the outpost. Changing the ingress class doesn't do shit so I am going to remake one @ `http://ak-outpost-auth-proxy-external.authentik:9000/outpost.goauthentik.io/auth/traefik` with this config:

```yaml
log_level: info
docker_labels: null
authentik_host: https://authentik.example.com/
docker_network: null
container_image: null
docker_map_ports: true
refresh_interval: minutes=5
kubernetes_replicas: 3
kubernetes_namespace: authentik
authentik_host_browser: ""
object_naming_template: ak-outpost-%(name)s
authentik_host_insecure: false
kubernetes_json_patches: null
kubernetes_service_type: ClusterIP
kubernetes_image_pull_secrets: []
kubernetes_ingress_class_name: traefik-external
kubernetes_disabled_components: []
kubernetes_ingress_annotations: {}
kubernetes_ingress_secret_name: certificate-example.com
```

The new outpost fixed the errors, and I'll need a second for internal, but Authentik was not getting routed which blew so much else up. I just wrote an `IngressRoute` CRD instead of using the ingress in it's helm and it worked. Will restrict to those so no surprise ingresses can be routed simply by installing a chart helm. 

### Making The Current Instance Unique

It took me way to long to figure out I had to update the helm release for `releaseName: traefik-external` so it I could name the two separately. `fullnameOverride: traefik-external` at the root of the values also worked but I went with the higher level property.

Services so far:

```
thaynes@kubem01:~/workspace/secret-sealing$ k -n traefik get services
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP                 PORT(S)                                    AGE
haynestower        ExternalName   <none>          haynestower.example   <none>                                     18d
traefik-external   LoadBalancer   10.43.124.187   192.168.40.100              80:31651/TCP,443:30420/TCP,443:30420/UDP   57s
whoami             ClusterIP      10.43.197.214   <none>                      80/TCP                                     19d
```

Pods all unique:

```
thaynes@kubem01:~/workspace/secret-sealing$ k -n traefik get pods
NAME                               READY   STATUS    RESTARTS        AGE
traefik-external-59496c4f8-hc8pz   1/1     Running   0               54s
traefik-external-59496c4f8-kv9bj   1/1     Running   0               54s
traefik-external-59496c4f8-nbv7b   1/1     Running   0               54s
traefik-external-59496c4f8-pxvsq   1/1     Running   0               54s
traefik-external-59496c4f8-zp4nf   1/1     Running   0               54s
whoami                             1/1     Running   2 (7d21h ago)   19d
```

> **TODO** Clean up `whoami`

### traefik-internal HelmRelease

Not really sure what is needed other than things that need to be uniquely `traefik-internal` and the IP.

`helmrelease-traefik-internal.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: traefik-internal
  namespace: traefik
spec:
  chart:
    spec:
      chart: traefik
      version: 30.x 
      sourceRef:
        kind: HelmRepository
        name: traefik
        namespace: flux-system
  interval: 15m
  timeout: 5m
  releaseName: traefik-internal
  values: 
    # Values https://github.com/traefik/traefik-helm-chart/blob/master/traefik/values.yaml 
    globalArguments:
      - "--global.sendanonymoususage=false"
      - "--global.checknewversion=false"

    additionalArguments:
      - "--serversTransport.insecureSkipVerify=true"
      - "--log.level=DEBUG" # TODO Set to INFO later

    deployment:
      enabled: true
      replicas: 5
      annotations: {}
      podAnnotations: {}
      additionalContainers: []
      initContainers: []

    ports:
      web:
        redirectTo:
          port: websecure
          priority: 10
      websecure:
        http3:
          enabled: true
        advertisedPort: 4443
        tls:
          enabled: true
          
    ingressRoute:
      dashboard:
        enabled: false # will declare in a separate file
  
    providers:
      kubernetesCRD:
        enabled: true
        ingressClass: traefik-internal
        allowExternalNameServices: true
        allowCrossNamespace: true
      kubernetesIngress:
        enabled: false
        ingressClass: traefik-internal
        allowExternalNameServices: true
        publishedService:
          enabled: false

    rbac:
      enabled: true

    service:
      enabled: true
      type: LoadBalancer
      annotations:
        metallb.universe.tf/loadBalancerIPs: 192.168.40.200
      labels: {}
      loadBalancerSourceRanges: []
      externalIPs: []
```

I will also route an internal dashboard

`ingressroute-traefik-internal-dashboard.yaml`
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-internal-dashboard
  namespace: traefik
  annotations: 
    kubernetes.io/ingress.class: traefik-internal
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.internal.haynes`) # TODO setup external-dns to do UniFi entries for local stuff
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
      middlewares:
        - name: traefik-dashboard-basicauth #authentik-external-auth-proxy
          namespace: traefik
  tls:
    secretName: certificate-local.example.com
```

## Let it rip!

FAILED

```
ingress-routes                      18d     False   kustomize build failed: accumulating resources: accumulation err='merging resources from './internal-routes/ingressroute-traefik-internal-dashboard.yaml': may not add resource with an already registered id: IngressRoute.v1alpha1.traefik.io/traefik-dashboard.traefik': must build at directory: '/tmp/kustomization-1980296721/traefik/ingress-routes/internal-routes/ingressroute-traefik-internal-dashboard.yaml': file is not directory
```

Looks easy, I duped the name of the new `IngressRoute` - needs to be `name: traefik-internal-dashboard`. 

And it worked! https://traefik.internal.haynes/ brings up the dashboard but it's not secure which is weird. Might be the auth middleware using it's own self signed cert, I will wait until I port it to Authentik and see then.

## Migrating IngressRoutes

Now we have a second instance of traefik and nothing blew up. Even have two dashboards, neither on fire.

Now to switch sonarr's IngressRoute from:

```yaml
  annotations:
    kubernetes.io/ingress.class: traefik-external
```

To:

```yaml
  annotations:
    kubernetes.io/ingress.class: traefik-internal
```

AND we need a new outpost plus new middleware since Authentik also shares this "external" ingress class:

### Authentik

`Auth Proxy Internal` Outpost settings:
```yaml
log_level: info
docker_labels: null
authentik_host: https://authentik.example.com/
docker_network: null
container_image: null
docker_map_ports: true
refresh_interval: minutes=5
kubernetes_replicas: 3
kubernetes_namespace: authentik
authentik_host_browser: ""
object_naming_template: ak-outpost-%(name)s
authentik_host_insecure: false
kubernetes_json_patches: null
kubernetes_service_type: ClusterIP
kubernetes_image_pull_secrets: []
kubernetes_ingress_class_name: traefik-internal
kubernetes_disabled_components: []
kubernetes_ingress_annotations: {}
kubernetes_ingress_secret_name: certificate-local.example.com
```

#### Middleware:

`middleware-authentik-internal-auth-proxy.yaml`
```yaml
apiVersion: traefik.io/v1alpha1 # traefik.containo.us/v1alpha1 depreciated in Traefik v3
kind: Middleware
metadata:
  name: authentik-internal-auth-proxy
  namespace: traefik
spec:
  forwardAuth:
    # address of the identity provider (IdP)
    address: http://ak-outpost-auth-proxy-external.authentik:9000/outpost.goauthentik.io/auth/traefik
    trustForwardHeader: true
    # headers that are copied from the IdP response and set on forwarded request to the backend application
    authResponseHeaders:
    - X-authentik-username
    - X-authentik-groups
    - X-authentik-email
    - X-authentik-name
    - X-authentik-uid
    - X-authentik-jwt
    - X-authentik-meta-jwks
    - X-authentik-meta-outpost
    - X-authentik-meta-provider
    - X-authentik-meta-app
    - X-authentik-meta-version
```

#### Authentik Generated Reference:

Authentik generates middleware for the outposts in a different namespace which I could use but I'd rather have more control if I wanted to change it later.

```yaml
Spec:
  Forward Auth:
    Address:  http://ak-outpost-auth-proxy-internal.authentik:9000/outpost.goauthentik.io/auth/traefik
    Auth Response Headers:
      X-authentik-username
      X-authentik-groups
      X-authentik-email
      X-authentik-name
      X-authentik-uid
      X-authentik-jwt
      X-authentik-meta-jwks
      X-authentik-meta-outpost
      X-authentik-meta-provider
      X-authentik-meta-app
      X-authentik-meta-version
    Auth Response Headers Regex:  
    Trust Forward Header:         true
```

### Test It!

First test didn't work but that was because my DNS entry still pointed to the external traefik instance. Once I manually went in and switched that around we were good to go. Next up is external-dns for UniFI so I don't run into forgetting to checking how that is configured! 