---
title: Open WebUI
permalink: /docs/haynes-intelligence/open-webui/
---

Open WebUI was previously running as a beater deployment I could test before rebuilding the cluster. Now we are going to do it right, but still beat on it because I have no tested restoring from S3 yet.

## Prep The FLux

[This guide](https://blog.joshuastock.net/getting-started-with-authentik-or-keycloak-and-open-web-ui-a-step-by-step-guide) looks like a good start to get things going. 

### Small Blocker

But first, unresolved tech deb is calling. Before I get too beep I want to get ![HaynesIntelligence]({{ site.url }}/hardware/haynesintelligence/) over to the Hayneslab VLAN.

OK, that's never easy, but it's migrated to the VLAN. You can see how it went ![here]({{ site.url }}/hardware/haynesintelligence#haynesIntelligence-to-vlan)

> **NOTE** If that link is broke it's becuase I forgot to test it and have not used many inline links

### Back to The Flux

#### Namespace

I'm going to group AI/LLM related things update one namespace, specifically apps that depend on `HaynesIntelligence` as their backend for them GPUs.

`namespace-haynes-intelligence.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: haynes-intelligence
```

#### Helm Repository

This is straightforward based off of the [official docs](https://docs.openwebui.com/getting-started/installation) or the [chart readme](https://github.com/open-webui/helm-charts). I want the repo that is added when they say `helm repo add open-webui https://helm.openwebui.com/`.

`helmrepository-open-webui.yaml`:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: open-webui
  namespace: flux-system
spec:
  interval: 15m
  url: https://helm.openwebui.com/
```

#### Bootstrap Kustomization

I don't expect much here yet except for referencing a folder. We may want to add a dependency on `authentik` later but I think we only want that if the service can't come up without the dependency and in this case just SSO is degraded.

> **TODO** Should `authentik` depend on `traefik` since it doesn't do much of anything with out it? Or should it start up on it's own and just not do much. 

`kustomization-open-webui.yaml`:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: open-webui
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
      kind: HelmRelease
      name: open-webui
      namespace: haynes-intelligence
  path: ./open-webui
```

> **TODO** Should every bootstrap kustomization have `retryInterval: 1m` and `wait: true` ?

#### Helm Release

This should be straightforward except for the vales. Going off of their helm [releases](https://github.com/open-webui/helm-charts/releases) the latest is `3.1.6` so I will take all from `3.1.x`.

`helmrelease-open-webui.yaml`:
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: open-webui
  namespace: haynes-intelligence
spec:
  chart:
    spec:
      chart: open-webui
      version: 3.1.x
      sourceRef:
        kind: HelmRepository
        name: open-webui
        namespace: flux-system
  interval: 15m
  timeout: 5m
  releaseName: open-webui
  values: # found here https://github.com/open-webui/helm-charts/blob/main/charts/open-webui/values.yaml
  ```

##### Helm Values

I have older values I was using in Kubernetes before I switched to `flux`. It's mostly what I need except I don't use `NodePort` now that I have a loadbalancer. You may want to run Ollama with the UI but since I don't have GPUs in this cluster there's no way it'll perform well without the beefy server as it's Ollama instance.

```yaml
  values: # found here https://github.com/open-webui/helm-charts/blob/main/charts/open-webui/values.yaml
    # -- Automatically install Ollama Helm chart from https://otwld.github.io/ollama-helm/. Use [Helm Values](https://github.com/otwld/ollama-helm/#helm-values) to configure
    ollama:
      enabled: false

    # -- A list of Ollama API endpoints. These can be added in lieu of automatically installing the Ollama Helm chart, or in addition to it.
    ollamaUrls: [http://pop01.haynesnetwork:11434] # TODO traefik this 

    # If > 1 use cephfs w/ ReadWriteMany
    # TODO Keep at 1, seems to be a lot of problems with replication at the moment
    replicaCount: 1

    persistence:
      enabled: true
      size: 2Gi
      # -- Use existingClaim if you want to re-use an existing Open WebUI PVC instead of creating a new one
      existingClaim: ""
      # -- If using multiple replicas, you must update accessModes to ReadWriteMany - ReadWriteOnce is default.
      accessModes:
        - ReadWriteMany
      storageClass: cephfs
      selector: {}
      annotations: {}

    # -- Service values to expose Open WebUI pods to cluster
    service:
      type: LoadBalancer
      annotations:
        metallb.universe.tf/loadBalancerIPs: 192.168.40.103
      #loadBalancerIP: 192.168.40.103 # Can't do this and metallb.universe.tf/loadBalancerIPs

    extraEnvVars:
      # -- Default API key value for Pipelines. Should be updated in a production deployment, or be changed to the required API key if not using Pipelines
      - name: GLOBAL_LOG_LEVEL
        value: "DEBUG" # TODO set to "INFO" once stable
      - name: OPENAI_API_BASE_URLS
        value: "http://open-webui-pipelines.haynes-intelligence.svc.cluster.local:9099;https://api.openai.com/v1"
      - name: OPENAI_API_KEYS
        valueFrom:
          secretKeyRef:
            name: openai-open-webui-api-key
            key: apikeys
      - name: OPENID_PROVIDER_URL
        value: "https://authentik.haynesnetwork.com/application/o/open-webui/.well-known/openid-configuration"
      - name: OAUTH_CLIENT_ID
        valueFrom:
          secretKeyRef:
            name: open-webui-provider-credentials
            key: id
      - name: OAUTH_CLIENT_SECRET
        valueFrom:
          secretKeyRef:
            name: open-webui-provider-credentials
            key: secret
      - name: OAUTH_PROVIDER_NAME
        value: "authentik"
      - name: ENABLE_OAUTH_SIGNUP
        value: "true"
      - name: ENABLE_LOGIN_FORM
        value: "false"
      - name: DEFAULT_USER_ROLE
        value: "user"
```

I decided to use this opportunity to test a `cephfs` filesystem since what I have now are database / caches that are 1:1 with the pod using them. This UI allows me to specify replicates for the service that will all hit the same volume which I have not tested before.

#### Let It Rock

Before we layer anything else it we need to verify this all works. This usually means fixing a few typos too. Or spending three days troubleshooting. It's hard to tell. 

Few typos later and the app is up but it can't hit the ollama server:

```
2024-08-18 15:52:16	ERROR [apps.ollama.main] Connection error: Cannot connect to host pop01:80 ssl:default [Name or service not known]
2024-08-18 15:52:16	user-join UE5SS5wYg3VHbPcmAAAq {'auth': {'token': 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjhmMzAyMzBiLTg0NGItNDdkZS04NzA5LTg2MWQ0ZTUyNGJmOCJ9.KlfwpF8BIRU5YRWQyfMHFKRykU5zLH3tOhF1UqXXSfw'}}
```

Oddly I can hit http://pop01 in my windows PC but not on the kubernetes master node so I guess that's what needs fixing. Gonna try exposing straight ollama instead of nginx'ing it. 


First a fresh install `curl -fsSL https://ollama.com/install.sh | sh`. 

```
curl: (6) Could not resolve host: ollama.com
```

Well that's not good. Turns out `/etc/resolve.conf` is really screwed up here. Need to check that for the other VMs I migrated to new IPs.

```bash
systemctl edit ollama.service
```

Add at the top:
```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
```

And reboot ollama:
```bash
systemctl daemon-reload
systemctl restart ollama
```

Now try "http://pop01.example:11434" and boom!

```
thaynes@kubem01:~$ curl http://pop01.example:11434
Ollama is running
```

Now to see if updating the helm values to `ollamaUrls: [http://pop01.example:11434]` gets us going. But still, a blank page. Setting `replicaCount: 1` fixed it for now but I'd like to be able to use replicas. Might be something in the official [docs](https://docs.openwebui.com/getting-started/env-configuration/).

### Reverse Proxy

Going through the tutorial I realize I need to set up my DNSEndpoint and IngressRoute so that Authentik doesn't add the local IP to the 

#### external-dns DNSEndpoint

I already had this in cloudflare but to better GitOps I deleted it so this one wouldn't freak out.

```yaml
apiVersion: externaldns.k8s.io/v1alpha1
kind: DNSEndpoint
metadata:
  name: "ai.example.com"
  namespace: external-dns
spec:
  endpoints:
  - dnsName: "ai.example.com"
    recordTTL: 180
    recordType: CNAME
    targets:
    - "example.com"
```

#### Traefik IngressRoute

This I copied from when I exposed a few other external services. However, this time the namespace is different so hopefully all I have to do is provide it with the other `Service` properties.

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: open-webui
  namespace: traefik
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`www.ai.example.com`)
      kind: Rule
      services:
        - kind: Service
          name: open-webui
          port: 80
          namespace: haynes-intelligence
      middlewares:
        - name: default-headers
          namespace: traefik
    - kind: Rule
      match: Host(`ai.example.com`) && PathPrefix(`/`)
      services:
        - kind: Service
          name: open-webui
          port: 80
          namespace: haynes-intelligence
      middlewares:
        - name: default-headers
          namespace: traefik
  tls:
    secretName: certificate-example.com
```

## Starting the Tutorial

### Forward

First, I want to thank this guy:

![open-webui-author]({{ site.url }}/images/web/open-webui-author.png)

I am writing this on 8/18/2024, this shit is fresh and that is rare. I can guarentee you I can "read" this page in under seven minutes but we'll be here all day fiddling. He has also used `docker-compose` but that should be fine for translating into `funky-flex`. 

### OAuth 

Looks like Open-WebUI supports OAuth natively and you can [disable the form](https://docs.openwebui.com/getting-started/env-configuration/#enable_login_form) so no local accounts may be used. 

Setting up `Authentik` seemed straightforward and I even added a nice logo for my SSO portal:

![logo](https://cdn.jsdelivr.net/gh/walkxcode/dashboard-icons/png/open-webui-light.png)

However, this guy doesn't `funky-flux` and just raw text'd his secret. I'm going to need to seal up that shit before the GitOps Police come at me.

> **NOTE** At this point I flipped Authentik online, going from `authentik.local.example.com` to `authentik.example.com` which was quite easy just by searching both these notes and my flux-repo. I didn't want to mess around routing from one to another.

Create a temp secret to pass in:

> **NOTE** Quotes here may matter, it was freaking out before I started adding quotes to every string.

`nano open-webui-secret.yaml `:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: open-webui-provider-credentials
  namespace: haynes-intelligence

type: Opaque
stringData:
  id: "SMALLREDACTED"
  secret: "BIGREDACTED"
```

Seal it up:

```bash
cat open-webui-secret.yaml | \
kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
--format yaml > sealedsecret-open-webui-provider-credentials.yaml
```

And copy into the flux repo in the open-webui folder:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: open-webui-provider-credentials
  namespace: haynes-intelligence
spec:
  encryptedData:
    id: HUGEREDACTED
    secret: CRAZYHUGEREDACTED
  template:
    metadata:
      creationTimestamp: null
      name: open-webui-provider-credentials
      namespace: haynes-intelligence
    type: Opaque
```

Now we must configure a bunch of stuff in the helm values to use OAuth:

> **WARNING** It has come to my attention via trial by fire that EVERYTHING needs quotes

```yaml
extraEnvVars:
      - name: GLOBAL_LOG_LEVEL
        value: "DEBUG" # TODO set to "INFO" once stable
      - name: OPENID_PROVIDER_URL
        value: "https://authentik.example.com/application/o/open-webui/.well-known/openid-configuration"
      - name: OAUTH_CLIENT_ID
        valueFrom:
          secretKeyRef:
            name: open-webui-provider-credentials
            key: id
      - name: OAUTH_CLIENT_SECRET
        valueFrom:
          secretKeyRef:
            name: open-webui-provider-credentials
            key: secret
      - name: OAUTH_PROVIDER_NAME
        value: "authentik"
      - name: ENABLE_OAUTH_SIGNUP
        value: "true"
      - name: ENABLE_LOGIN_FORM
        value: "false"
```

> **TODO** I have not proxy'd authentik over the internet, this is the first service needing that access. Until the `.local...` is dropped from my URL this will only work locally. 

`ENABLE_LOGIN_FORM=false` only if you do not want local accounts to be allowed. I will leave this true to start so I can test it and give my other account admin.

## Velero Backup & Restore

Now that we have a less critical service up and running I can properly test velero. We will do this from the schedule since that is mission critical.

First see what's in S3 to we know if something new was created:

![s3-before-test]({{ site.url }}/images/web/s3-before-test.png)

### Test Backing Up to S3

Grab the schedule name:

```
thaynes@kubem01:~$ velero get schedules
NAME                   STATUS    CREATED                         SCHEDULE    BACKUP TTL   LAST BACKUP   SELECTOR   PAUSED
velero-daily-backups   Enabled   2024-08-12 23:16:24 -0400 EDT   0 0 * * *   240h0m0s     13m ago       <none>     false
```

Ask velero to make the backup:

```bash
velero backup create --from-schedule=velero-daily-backups
```

It kinda complained but seemed to work. I think I should have done a `--wait` so I could see righ when it finished. However, it tells me I can describe it and I assume that will have a status:

```
thaynes@kubem01:~$ velero backup create --from-schedule=velero-daily-backups
INFO[0000] No Schedule.template.metadata.labels set - using Schedule.labels for backup object  backup=velero/velero-daily-backups-20240819001742 labels="map[app.kubernetes.io/instance:velero app.kubernetes.io/managed-by:Helm app.kubernetes.io/name:velero helm.sh/chart:velero-5.1.7 helm.toolkit.fluxcd.io/name:velero helm.toolkit.fluxcd.io/namespace:velero name:my-first-backups]"
Creating backup from schedule, all other filters are ignored.
Backup request "velero-daily-backups-20240819001742" submitted successfully.
Run `velero backup describe velero-daily-backups-20240819001742` or `velero backup logs velero-daily-backups-20240819001742` for more details.
```

Yeah, looks like it's cooking:

```
Backup Volumes:
  Velero-Native Snapshots: <none included>

  CSI Snapshots: <none included or not detectable>

  Pod Volume Backups - restic (specify --details for more information):
    Completed:    38
    In Progress:  3
    New:          44
```

Now looks complete from the `describe` command:

```
Backup Volumes:
  Velero-Native Snapshots: <none included>

  CSI Snapshots: <none included>

  Pod Volume Backups - restic (specify --details for more information):
    Completed:  85

HooksAttempted:  0
HooksFailed:     0
```

And here it is in S3:

![s3-after-test]({{ site.url }}/images/web/s3-after-test.png)

### Test Restoring from S3

This is the scary part but there's nothing to lose anyway since all I did was add a user and set the default model.

```bash
k delete ns haynes-intelligence
```

That killed it good:

![deleted-hi-namespace]({{ site.url }}/images/web/deleted-hi-namespace.png)

This backup now has everything since I set the schedule to '*' for namespaces. This means I need to figuring how to filter the backup to just restore `haynes-intelligence` stuff. Fortunately, velero has good [docs](https://velero.io/docs/main/resource-filtering/).

```bash
velero create restore --from-backup velero-daily-backups-20240819001742 --include-namespaces haynes-intelligence --wait
```

This time I get a nice waiting screen:

```
Restore request "velero-daily-backups-20240819001742-20240818203704" submitted successfully.
Waiting for restore to complete. You may safely press ctrl-c to stop waiting - your restore will continue in the background.
.............................
Restore completed with status: Completed. You may check for more information using the commands `velero restore describe velero-daily-backups-20240819001742-20240818203704` and `velero restore logs velero-daily-backups-20240819001742-20240818203704`.
```

And it's alive again!

```
thaynes@kubem01:~$ k -n haynes-intelligence get pv,pvc,pods,services
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                         STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-0731b8c7-a1d1-4ad1-9682-f98978c55361   2Gi        RWX            Delete           Bound    haynes-intelligence/open-webui                                cephfs         <unset>                          41s
persistentvolume/pvc-e1c9f1d3-e6ce-4404-9ffc-9e8a80780ac3   2Gi        RWO            Delete           Bound    haynes-intelligence/open-webui-pipelines                      local-path     <unset>                          38s

NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/open-webui             Bound    pvc-0731b8c7-a1d1-4ad1-9682-f98978c55361   2Gi        RWX            cephfs         <unset>                 41s
persistentvolumeclaim/open-webui-pipelines   Bound    pvc-e1c9f1d3-e6ce-4404-9ffc-9e8a80780ac3   2Gi        RWO            local-path     <unset>                 41s

NAME                                        READY   STATUS    RESTARTS   AGE
pod/open-webui-0                            1/1     Running   0          41s
pod/open-webui-pipelines-69cccbdbbc-6zh9k   1/1     Running   0          41s

NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
service/open-webui             LoadBalancer   10.43.137.62   192.168.40.103   80:31405/TCP   41s
service/open-webui-pipelines   ClusterIP      10.43.66.239   <none>           9099/TCP       41s
```

## OpenAI Integration

Since I'm not sure how pipelines work I am going to stash that shit here for now:

`http://open-webui-pipelines.haynes-intelligence.svc.cluster.local:9099` must be the default URL for this.


While this is the default key:

```yaml
      - name: OPENAI_API_KEY
        value: "0p3n-w3bu!"
```

I can add that back in later but first I'm going to try just vanilla OpenAI via some secret sealing:

```bash
kubectl create secret generic openai-open-webui-api-key \
  --namespace haynes-intelligence \
  --dry-run=client \
  --from-literal=key="OPENAI_API_KEY" -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem --format yaml \
  > sealedsecret-openai-open-webui-api-key.yaml
```

Now I need to swap the pipeline URL for `https://api.openai.com/v1` and use the secret I just locked up. This can be done through the [well documented env vars](https://docs.openwebui.com/getting-started/env-configuration/#enable_login_form)

```yaml
      - name: OPENAI_API_BASE_URL
        value: "https://api.openai.com/v1"
      - name: OPENAI_API_KEY
        valueFrom:
          secretKeyRef:
            name: openai-open-webui-api-key
            key: key
```

From [this doc](https://docs.openwebui.com/tutorial/openai/) I believe I can add the pipeline back in with something like:

```bash
kubectl create secret generic openai-open-webui-api-key \
  --namespace haynes-intelligence \
  --dry-run=client \
  --from-literal=apikeys="0p3n-w3bu!;OPENAI_API_KEY" -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem --format yaml \
  > sealedsecret-openai-open-webui-api-key.yaml
```

Each key is mapped in the same order of the urls, so we'd add this to the config:

> **NOTE** from-literal was throwing an error so I switched to this method

`nano openai-open-webui-secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openai-open-webui-api-key
  namespace: haynes-intelligence

type: Opaque
stringData:
  apikeys: "0p3n-w3bu!;OPENAI_API_KEY"
```

```bash
cat openai-open-webui-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-openai-open-webui-api-key.yaml
```

> **NOTE** The env vars are now plural

```yaml
      - name: OPENAI_API_BASE_URLs
        value: "http://open-webui-pipelines.haynes-intelligence.svc.cluster.local:9099;https://api.openai.com/v1"
      - name: OPENAI_API_KEYS
        valueFrom:
          secretKeyRef:
            name: openai-open-webui-api-key
            key: apikeys # I changed the name of this
```

> **WARNING** These configs didn't update anything but I had already message around with manually changing settings so it could have been to avoid overwriting. I will leave these configured as shown so they'll be set if I re-deploy.