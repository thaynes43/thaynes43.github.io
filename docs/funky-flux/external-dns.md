---
title: External DNS
permalink: /docs/funky-flux/external-dns/
---

[external-dns](https://github.com/kubernetes-sigs/external-dns) will let me manage DNS entries in Cloudflare via my flux repo which sounds most excellent. I assume later one we will add ingress to reverse proxy these like we are doing with NPM on unRAID right now.

## Adding to Flux-Repo

I am going to follow the [funky cookbook](https://geek-cookbook.funkypenguin.co.nz/kubernetes/external-dns/) pretty closely for this so expect a lot of copy pasta.

I took the chart from [here](https://github.com/bitnami/charts/blob/main/bitnami/external-dns/values.yaml) which was sha 773fcd7b74b7231483b12f25c65a291a52cc2e9c from 8/6/2024

One call out is we are putting it in a mode where all DNS entries will be defined in CRD but we could let services or other things add them later:

```
        sources:
          - crd
          # - service
          # - ingress
          # - contour-httpproxy
```

Now the annoying [sealed secret](https://geek-cookbook.funkypenguin.co.nz/kubernetes/sealed-secrets/):

```bash
  kubectl create secret generic cloudflare-api-token \
  --namespace external-dns \
  --dry-run=client \
  --from-literal=cloudflare_api_token=REDACTED -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  > sealedsecret-cloudflare-api-token.yaml
```

And off to the races:

```git
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   bootstrap/helmrepositories/helmrepository-bitnami.yaml
        new file:   bootstrap/kustomizations/kustomization-external-dns.yaml
        new file:   bootstrap/namespaces/namespace-external-dns.yaml
        new file:   external-dns/helmrelease-external-dns.yaml
        new file:   external-dns/sealedsecret-cloudflare-api-token.yaml
```

Helm did not like the chart version:

```
2024-08-08T00:51:16.939Z error HelmChart/external-dns-external-dns.flux-system - reconciliation stalled invalid chart reference: failed to get chart version for remote reference: no 'external-dns' chart with version matching '5.1.x' found
```

New version, `8.3.x` worked, but it didn't like this:

```
  Warning  InstallFailed        49s   helm-controller  Helm install failed for release external-dns/external-dns with chart external-dns@8.3.4: 1 error occurred:
           * CustomResourceDefinition.apiextensions.k8s.io "dnsendpoints.externaldns.k8s.io" is invalid: metadata.annotations[api-approved.kubernetes.io]: Required value: protected groups must have approval annotation "api-approved.kubernetes.io", see https://github.com/kubernetes/enhancements/pull/1111
```

And this is a real issue with external-dns. See [here](https://github.com/kubernetes-sigs/external-dns/issues/4657) for the guy asking for the release to be cut since the fix is there already! Next I tried `7.3.2` and it's much better:

```
thaynes@kubevip:~/workspace/external-dns$ helm list -a -n external-dns
NAME        	NAMESPACE   	REVISION	UPDATED                                	STATUS  	CHART             	APP VERSION
external-dns	external-dns	2       	2024-08-08 01:13:21.507616746 +0000 UTC	deployed	external-dns-7.3.2	0.14.1     
```

```
thaynes@kubevip:~/workspace/external-dns$ flux get kustomizations external-dns
NAME        	REVISION          	SUSPENDED	READY	MESSAGE                              
external-dns	main@sha1:3a10b46a	False    	True 	Applied revision: main@sha1:3a10b46a	
```

```
thaynes@kubevip:~/workspace/external-dns$ flux get helmreleases -n external-dns external-dns
NAME        	REVISION	SUSPENDED	READY	MESSAGE                                                                                       
external-dns	7.3.2   	False    	True 	Helm upgrade succeeded for release external-dns/external-dns.v2 with chart external-dns@7.3.2	
```

```
thaynes@kubevip:~/workspace/external-dns$ kubectl get pods -n external-dns -l app.kubernetes.io/name=external-dns
NAME                            READY   STATUS    RESTARTS   AGE
external-dns-6979678bd9-4ppck   1/1     Running   0          3m20s
```

All systems nominal!

### But Wait - It's Fucked

`external-dns` now crashes because it sees the entry it added. This was reported, fixed, and good to go in the version I can't use because of the helm chart angst so I will have to do some critical thinking. 

Deleting the records in cloudflare and letting it sync fixed that.

## Moving away from CRDs

Before I can do the UniFi stuff I need to move to adding entries based on the ingres class. Classes `traefik-external` and `traefik-internal` will tell which of the two needs to create the entry. [This doc](https://kubernetes-sigs.github.io/external-dns/v0.13.6/tutorials/traefik-proxy/#manifest-for-clusters-with-rbac-enabled) explains how adding an annotation like `external-dns.alpha.kubernetes.io/target: traefik.example.com` to the `IngressRoute` is all I'll need after this.

Other than the annotation I'll need to figure out how to how to configure the helm charts to use `source=traefik-proxy` and only look at one of the other ingress class.

> **BLOCKED** The ExtraArgs for this chart won't let me pass what I need 

I was able to work around it like this:

```yaml
    extraArgs:
      traefik-disable-legacy:
```

But it didn't quite work. It added what I excluded and only in www form so none of it worked.

### Use Official External DNS Chart

Maybe time for a new chart cause this back in time nonsense is exhausting. There is an [official chart]( https://github.com/kubernetes-sigs/external-dns/blob/master/charts/external-dns/) to try.

Just needs an updated helm release and repo.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-dns-cloudflare
  namespace: external-dns
spec:
  interval: 30m
  chart:
    spec:
      chart: external-dns # https://github.com/kubernetes-sigs/external-dns/blob/master/charts/external-dns/values.yaml
      version: 1.14.5
      sourceRef:
        kind: HelmRepository
        name: external-dns
        namespace: flux-system
  install:
    crds: CreateReplace
    remediation:
      retries: 3
  upgrade:
    cleanupOnFail: true
    crds: CreateReplace
    remediation:
      strategy: rollback
      retries: 3
  values:
    provider:
      name: cloudflare
    env:
      - name: CF_API_TOKEN
        valueFrom:
          secretKeyRef:
            name: cloudflare-api-token
            key: cloudflare_api_token
    extraArgs:
      - --cloudflare-proxied
      - --traefik-disable-legacy
    logLevel: debug
    policy: sync
    sources: ["crd", "traefik-proxy"]
    domainFilters: ["example.com"]
```

Note I did not add:

```yaml
    serviceMonitor:
      enabled: true
```

Or `--ignore-ingress-tls-spec` from the example but these may be worth looking into. The rest didn't seem needed, something was crashing my shit though not sure what I removed to fix it.

### Delete CRD And Use IngressRoute Annotation

After some confusion I found the following annotation would be sufficient to replace the CRD for external DNS entries:

```yaml
  annotations:
    external-dns.alpha.kubernetes.io/target: example.com
```

During the triage I came to realize that I should run a second instance of traefik exclusively for local routes, segregated from the external instance. This would let me annotate each `IngressRoute` with a different class `kubernetes.io/ingress.class: traefik-external` and each instance would be configured with the class it uses via `ingressClass: traefik-external` [see this sha](https://github.com/thaynes43/flux-repo/commit/615644414dd45524a6fbe41eec3b1fe36f37a7cb) as I removed it.

## Unifi Webhook

TODO https://github.com/kashalls/external-dns-unifi-webhook but first I gotta fix the shit hole of the other one

[Here](https://github.com/onedr0p/home-ops/tree/main/kubernetes/main/apps/network/external-dns) is an example of how somebody got this up and running with flux already!

The chart at `https://kubernetes-sigs.github.io/external-dns/` will work for this and the `external-dns` namespace will be used.

### Secret

Username: user
Password: REDACTED

```bash
kubectl create secret generic external-dns-unifi-secret \
--namespace external-dns \
--dry-run=client \
--from-literal username="user" \
--from-literal password="REDACTED" \
-o yaml > external-dns-unifi-secret.yaml
```

```bash
cat external-dns-unifi-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem --format yaml > sealedsecret-external-dns-unifi-secret.yaml
```

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: external-dns-unifi-secret
  namespace: external-dns
spec:
  encryptedData:
    password: REDACTED
    username: REDACTED
  template:
    metadata:
      creationTimestamp: null
      name: external-dns-unifi-secret
      namespace: external-dns
```

### HelmRelease

I am going to start [here](https://github.com/onedr0p/home-ops/blob/main/kubernetes/main/apps/network/external-dns/unifi/helmrelease.yaml) and work it into something I can use.

`helmrelease-external-dns-unifi.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-dns-unifi
  namespace: external-dns
spec:
  interval: 30m
  chart:
    spec:
      chart: external-dns
      version: 1.14.5 # https://github.com/kubernetes-sigs/external-dns/blob/master/charts/external-dns/values.yaml
      sourceRef:
        kind: HelmRepository
        name: external-dns
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
    fullnameOverride: external-dns-unifi
    logLevel: debug
    provider:
      name: webhook
      webhook:
        image:
          repository: ghcr.io/kashalls/external-dns-unifi-webhook
          tag: v0.2.2
        env:
          - name: UNIFI_HOST
            value: https://192.168.40.1
          - name: UNIFI_USER
            valueFrom:
              secretKeyRef:
                name: external-dns-unifi-secret
                key: username
          - name: UNIFI_PASS
            valueFrom:
              secretKeyRef:
                name: external-dns-unifi-secret
                key: password
          - name: LOG_LEVEL
            value: "debug"
        livenessProbe:
          httpGet:
            path: /healthz
            port: http-wh-metrics
          initialDelaySeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /readyz
            port: http-wh-metrics
          initialDelaySeconds: 10
          timeoutSeconds: 5
    extraArgs:
      - --traefik-disable-legacy
      - --ingress-class=traefik-internal
    policy: sync
    sources: ["crd", "traefik-proxy"]
```

## Test Unifi External DNS!

Hopefully a slam dunk. The person who made this webhook thanked me on discord for starting the repo so at least I know who to ask for help.

Had a false start due to a missing `namespace: external-dns` in the HelmRelease.

Well, it worked. It generated endpoints for all the external stuff... Sifting through the config it loaded I so see `IngressClassNames:[traefik-internal]`/

```
time="2024-09-02T02:37:56Z" level=info msg="config: {APIServerURL: KubeConfig: RequestTimeout:30s DefaultTargets:[] GlooNamespaces:[gloo-system] SkipperRouteGroupVersion:zalando.org/v1 Sources:[crd traefik-proxy] Namespace: AnnotationFilter: LabelFilter: IngressClassNames:[traefik-internal] FQDNTemplate: CombineFQDNAndAnnotation:false IgnoreHostnameAnnotation:false IgnoreIngressTLSSpec:false IgnoreIngressRulesSpec:false GatewayNamespace: GatewayLabelFilter: Compatibility: PublishInternal:false PublishHostIP:false AlwaysPublishNotReadyAddresses:false ConnectorSourceServer:localhost:8080 Provider:webhook GoogleProject: GoogleBatchChangeSize:1000 GoogleBatchChangeInterval:1s GoogleZoneVisibility: DomainFilter:[] ExcludeDomains:[] RegexDomainFilter: RegexDomainExclusion: ZoneNameFilter:[] ZoneIDFilter:[] TargetNetFilter:[] ExcludeTargetNets:[] AlibabaCloudConfigFile:/etc/kubernetes/alibaba-cloud.json AlibabaCloudZoneType: AWSZoneType: AWSZoneTagFilter:[] AWSAssumeRole: AWSAssumeRoleExternalID: AWSBatchChangeSize:1000 AWSBatchChangeSizeBytes:32000 AWSBatchChangeSizeValues:1000 AWSBatchChangeInterval:1s AWSEvaluateTargetHealth:true AWSAPIRetries:3 AWSPreferCNAME:false AWSZoneCacheDuration:0s AWSSDServiceCleanup:false AWSZoneMatchParent:false AWSDynamoDBRegion: AWSDynamoDBTable:external-dns AzureConfigFile:/etc/kubernetes/azure.json AzureResourceGroup: AzureSubscriptionID: AzureUserAssignedIdentityClientID: AzureActiveDirectoryAuthorityHost: BluecatDNSConfiguration: BluecatConfigFile:/etc/kubernetes/bluecat.json BluecatDNSView: BluecatGatewayHost: BluecatRootZone: BluecatDNSServerName: BluecatDNSDeployType:no-deploy BluecatSkipTLSVerify:false CloudflareProxied:false CloudflareDNSRecordsPerPage:100 CoreDNSPrefix:/skydns/ RcodezeroTXTEncrypt:false AkamaiServiceConsumerDomain: AkamaiClientToken: AkamaiClientSecret: AkamaiAccessToken: AkamaiEdgercPath: AkamaiEdgercSection: InfobloxGridHost: InfobloxWapiPort:443 InfobloxWapiUsername:admin InfobloxWapiPassword: InfobloxWapiVersion:2.3.1 InfobloxSSLVerify:true InfobloxView: InfobloxMaxResults:0 InfobloxFQDNRegEx: InfobloxNameRegEx: InfobloxCreatePTR:false InfobloxCacheDuration:0 DynCustomerName: DynUsername: DynPassword: DynMinTTLSeconds:0 OCIConfigFile:/etc/kubernetes/oci.yaml OCICompartmentOCID: OCIAuthInstancePrincipal:false OCIZoneScope:GLOBAL OCIZoneCacheDuration:0s InMemoryZones:[] OVHEndpoint:ovh-eu OVHApiRateLimit:20 PDNSServer:http://localhost:8081 PDNSAPIKey: PDNSSkipTLSVerify:false TLSCA: TLSClientCert: TLSClientCertKey: Policy:sync Registry:txt TXTOwnerID:default TXTPrefix: TXTSuffix: TXTEncryptEnabled:false TXTEncryptAESKey: Interval:1m0s MinEventSyncInterval:5s Once:false DryRun:false UpdateEvents:false LogFormat:text MetricsAddress::7979 LogLevel:debug TXTCacheInterval:0s TXTWildcardReplacement: ExoscaleEndpoint: ExoscaleAPIKey: ExoscaleAPISecret: ExoscaleAPIEnvironment:api ExoscaleAPIZone:ch-gva-2 CRDSourceAPIVersion:externaldns.k8s.io/v1alpha1 CRDSourceKind:DNSEndpoint ServiceTypeFilter:[] CFAPIEndpoint: CFUsername: CFPassword: ResolveServiceLoadBalancerHostname:false RFC2136Host: RFC2136Port:0 RFC2136Zone:[] RFC2136Insecure:false RFC2136GSSTSIG:false RFC2136KerberosRealm: RFC2136KerberosUsername: RFC2136KerberosPassword: RFC2136TSIGKeyName: RFC2136TSIGSecret: RFC2136TSIGSecretAlg: RFC2136TAXFR:false RFC2136MinTTL:0s RFC2136BatchChangeSize:50 RFC2136UseTLS:false RFC2136SkipTLSVerify:false NS1Endpoint: NS1IgnoreSSL:false NS1MinTTLSeconds:0 TransIPAccountName: TransIPPrivateKeyFile: DigitalOceanAPIPageSize:50 ManagedDNSRecordTypes:[A AAAA CNAME] ExcludeDNSRecordTypes:[] GoDaddyAPIKey: GoDaddySecretKey: GoDaddyTTL:0 GoDaddyOTE:false OCPRouterName: IBMCloudProxied:false IBMCloudConfigFile:/etc/kubernetes/ibmcloud.json TencentCloudConfigFile:/etc/kubernetes/tencent-cloud.json TencentCloudZoneType: PiholeServer: PiholePassword: PiholeTLSInsecureSkipVerify:false PluralCluster: PluralProvider: WebhookProviderURL:http://localhost:8888 WebhookProviderReadTimeout:5s WebhookProviderWriteTimeout:10s WebhookServer:false TraefikDisableLegacy:true TraefikDisableNew:false}"
```

And this filter works for Traefik as it's how each instance picks up what IngressRoute it owns. However, I'm going to take the easy way out on this and just `excludeDomains: ["example.com"]`.

### I need new certs!!! 

I have another domain that that I use just for these notes so I'll see if I can use that for certs and then filter the external on the domain I use for self hosting and internal to that which won't have any entries in cloudflare. I also picked up `haynesops.com` while I was in there which I think I'll use for a public facing site so the notes can continue to bear the entire truth of my endevor. 

> **NOTE** Add a github pages site for haynesops easily with [this](https://dash.cloudflare.com/1adbb78981186f1bd409cc11913b459a/workers-and-pages/create/pages).


Now that `certificate-examplelab` and `certificate-examplelab-staging` were issued I need to use that in my local IngressRoutes with that domain. I also need to update Authentik's `Auth Proxy Internal` outpost to sue this secret (which may not matter).

#### IngressRoute Test

I added one DNS entry which will point to traefik:

![traefik-internal]({{ site.url }}/images/unifi/traefik-internal.png)

I can then target this in the annotations and it should all work if the domain in the cert matches the host:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: sonarr
  namespace: traefik
  annotations:
    external-dns.alpha.kubernetes.io/target: traefik.internal
    kubernetes.io/ingress.class: traefik-internal
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`sonarr.examplelab.net`) && PathPrefix(`/`)
      services:
        - kind: Service
          name: sonarr
          namespace: media-management
          port: 8989
      middlewares:  
        - name: authentik-internal-auth-proxy
          namespace: traefik 
  tls:
    secretName: certificate-examplelab
```

And we are good for sonarr! However, `traefik.internal.examplelab.net` did not work, I assume because I did not have a wildcard cert for this far out like I did `*.local.example.com`.

