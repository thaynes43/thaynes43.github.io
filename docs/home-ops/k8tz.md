---
title: K8TZ
permalink: /docs/home-ops/k8tz/
---

[K8tz](https://github.com/k8tz/k8tz/tree/master) looks like the shit, it keeps everything local time so logs are easier to deal with. There is a [official chart](https://github.com/k8tz/k8tz/tree/master/charts/k8tz) too.

## CRDs

### HelmRepository

`helmrepository-k8tz.yaml`
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: k8tz
  namespace: flux-system
spec:
  interval: 15m
  url: https://k8tz.github.io/k8tz/
```

### Namespace

The documentation states that k8tz will not set the timezone for anything in it's namespace. Therefore it can have it's own namespace so everything gets set.

`namespace-k8tz.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: k8tz
```

### Kustomization

I have too many files in these 'bootstrap' folders but I'll carry on for now and do a massive re-org later.

`kustomization-k8tz.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: k8tz
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
  path: ./k8tz
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: k8tz
      namespace: k8tz
```

### HelmRelease

Following the pattern of grouping by namespace this fucker gets an entire folder.

`helmrelease-k8tz.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: k8tz
  namespace: k8tz
spec:
  chart:
    spec:
      chart: k8tz
      version: 0.16.2 # https://github.com/k8tz/k8tz/blob/master/charts/k8tz/Chart.yaml
      sourceRef:
        kind: HelmRepository
        name: k8tz
        namespace: flux-system
  interval: 15m
  timeout: 5m
  values:
    kind: DaemonSet
    timezone: America/New_York
    injectAll: true
    cronJobTimeZone: true
    webhook:
      certManager:
        enabled: true
        issuerRef:
          name: k8tz-webhook-ca
          kind: Issuer
```

### Cert Manager Config

Because k8tz is an administration controller it needs a certificate. You can see this in the values under `webhook` but it references an issuer which we need to set up. We can use cert-manager for this with the following CRDs:

> **NOTE** I put a dependancy on the certificate and issue being up before k8tz starts.

`issuer-k8tz-webhook-ca.yaml`
```yaml
# Create an Issuer that uses the above generated CA certificate to issue certs
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: k8tz-webhook-ca
spec:
  ca:
    secretName: k8tz-webhook-ca
```

This basically passes a long another cert issues via this `Issuer` in the cert-manager folder:

```yaml
# Create a selfsigned Issuer, in order to create a root CA certificate for
# signing webhook serving certificates
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: k8tz-webhook-selfsign
spec:
  selfSigned: {}
```

And this is the cert:

```yaml
# Generate a CA Certificate used to sign certificates for the webhook
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: k8tz-webhook-ca
spec:
  secretName: k8tz-webhook-ca
  duration: 43800h # 5y
  issuerRef:
    name: k8tz-webhook-selfsign
    kind: Issuer
  commonName: "ca.k8tz.cert-manager"
  isCA: true
  secretTemplate:
    annotations:
      reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
      reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "k8tz"
      reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
      reflector.v1.k8s.emberstack.com/reflection-auto-namespaces: "k8tz"
```

I reflected the secret over so the secret was in the `k8tz` namespace too. I could probably have had cert-manager create the secret in the namespace initially but this way I can centralize everything and reflect to what is consuming them.

## LET IT RIP!

Few things came up.

1. `Issuers` needed namespaces which was different from the `ClusterIssuers` I had been using
1. I had a `letsencrypt-wildcard-cert` as the final outcome of my `certificates` Kustomization which I changed to a more generic `issued-certs` since I added a self signed folder. However, I missed a few spots that were using the old Kustomization name.

```
k8tz   4m49s   False   Helm install failed for release k8tz/k8tz with chart k8tz@0.16.2: Unable to continue with install: Namespace "k8tz" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; label validation error: missing key "app.kubernetes.io/managed-by": must be set to "Helm"; annotation validation error: missing key "meta.helm.sh/release-name": must be set to "k8tz"; annotation validation error: missing key "meta.helm.sh/release-namespace": must be set to "k8tz"
```

I am fairly certain this is because I created the namespace and did not set `namespace: null` in the values so it is trying to re-create the same namespace. If that isn't the case I can just delete this and say what it actually was.

And finally I missed that I had the Issuer in the cert-manager namespace configured instead of `k8tz-webhook-ca` which was copied from [here](https://github.com/bjw-s/home-ops/blob/main/kubernetes/main/apps/system-controllers/k8tz/app/helmrelease.yaml) but didn't work in my split-namespace setup unlike here where it's all in k8tz. I don't know if I'm right in changing it around but it worked.

Unfortunately requires restarting pods, but after you can see sonarr mounted with the timezone info:

```
thaynes@kubem01:~$ k -n media-management describe pod sonarr-7f87b7b5bd-wqwdf 
Name:             sonarr-7f87b7b5bd-wqwdf
Namespace:        media-management
Priority:         0
Service Account:  default
Node:             kubew04/192.168.40.46
Start Time:       Tue, 03 Sep 2024 22:34:06 -0400
Labels:           app.kubernetes.io/component=sonarr
                  app.kubernetes.io/instance=sonarr
                  app.kubernetes.io/name=sonarr
                  pod-template-hash=7f87b7b5bd
Annotations:      backup.velero.io/backup-volumes-excludes: tank
                  k8tz.io/injected: true
                  k8tz.io/timezone: America/New_York
Status:           Running
IP:               10.42.6.191
IPs:
  IP:           10.42.6.191
Controlled By:  ReplicaSet/sonarr-7f87b7b5bd
Init Containers:
  k8tz:
    Container ID:    containerd://51dd89e40c1fc121dcc1290d3ca34eb5b6341b31f38c2ecc84ed21e18d80edc8
    Image:           quay.io/k8tz/k8tz:0.16.2
    Image ID:        quay.io/k8tz/k8tz@sha256:5ddf4a9d77edab97b3204ea04a6c7f75405a1f924de8fb28d6b53b635c47060d
    Port:            <none>
    Host Port:       <none>
    SeccompProfile:  RuntimeDefault
    Args:
      bootstrap
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Tue, 03 Sep 2024 22:34:17 -0400
      Finished:     Tue, 03 Sep 2024 22:34:18 -0400
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /mnt/zoneinfo from k8tz (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2z447 (ro)
Containers:
  app:
    Container ID:   containerd://73daa8216e67d5004847ae418a51f947568c7bf52fa426dc23b17ea4bdaa53ae
    Image:          lscr.io/linuxserver/sonarr:4.0.9
    Image ID:       lscr.io/linuxserver/sonarr@sha256:879f5f35b05566f71296bad0f3704709103568a6b4f42f5959543f5322728723
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 03 Sep 2024 22:34:23 -0400
    Ready:          True
    Restart Count:  0
    Environment:
      SONARR__AUTH__METHOD:    External
      SONARR__AUTH__REQUIRED:  DisabledForLocalAddresses
      TZ:                      America/New_York
    Mounts:
      /config from config (rw)
      /etc/localtime from k8tz (ro,path="America/New_York")
      /tank from tank (rw)
      /usr/share/zoneinfo from k8tz (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2z447 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  config:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  sonarr-config
    ReadOnly:   false
  tank:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-smb-tank-k8s-media-management
    ReadOnly:   false
  kube-api-access-2z447:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
  k8tz:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:      
    SizeLimit:   <unset>
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason                  Age   From                     Message
  ----     ------                  ----  ----                     -------
  Normal   Scheduled               97s   default-scheduler        Successfully assigned media-management/sonarr-7f87b7b5bd-wqwdf to kubew04
  Warning  FailedAttachVolume      97s   attachdetach-controller  Multi-Attach error for volume "pvc-a5cfb96c-95b4-4d59-8b4f-0994dc864524" Volume is already used by pod(s) sonarr-7f87b7b5bd-2zfvx
  Normal   SuccessfulAttachVolume  86s   attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-a5cfb96c-95b4-4d59-8b4f-0994dc864524"
  Normal   Pulled                  86s   kubelet                  Container image "quay.io/k8tz/k8tz:0.16.2" already present on machine
  Normal   Created                 86s   kubelet                  Created container k8tz
  Normal   Started                 86s   kubelet                  Started container k8tz
  Normal   Pulling                 84s   kubelet                  Pulling image "lscr.io/linuxserver/sonarr:4.0.9"
  Normal   Pulled                  80s   kubelet                  Successfully pulled image "lscr.io/linuxserver/sonarr:4.0.9" in 4.579s (4.579s including waiting). Image size: 85484179 bytes.
  Normal   Created                 80s   kubelet                  Created container app
  Normal   Started                 80s   kubelet                  Started container app
```

### Velero

I'm 99% sure pods need to be bounced before it works. I can't exec into any of those pods but radarr is still UTC and the volume isn't mounted so I'm gonna go assume it's so and see if I can get a proper midnight backup.

I was able to do a rolling restart of the node-agent daemonset but I had to restart it seperate from the deployment for some reason:

```bash
kubectl -n velero rollout restart daemonset node-agent
kubectl -n velero rollout restart deployment velero
```

After both the node agents and velero joined us on the east coast:

node-agent example:

```
Events:
Type    Reason     Age   From               Message
----    ------     ----  ----               -------
Normal  Scheduled  98s   default-scheduler  Successfully assigned velero/node-agent-h2xx2 to kubew02
Normal  Pulled     98s   kubelet            Container image "quay.io/k8tz/k8tz:0.16.2" already present on machine
Normal  Created    98s   kubelet            Created container k8tz
Normal  Started    98s   kubelet            Started container k8tz
Normal  Pulled     97s   kubelet            Container image "velero/velero:v1.14.1" already present on machine
Normal  Created    97s   kubelet            Created container node-agent
Normal  Started    97s   kubelet            Started container node-agent
```

velero:

```
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  6m42s  default-scheduler  Successfully assigned velero/velero-7c4556bdcb-xzcfp to kubew04
  Normal  Pulling    6m43s  kubelet            Pulling image "velero/velero-plugin-for-aws:v1.10.0"
  Normal  Pulled     6m41s  kubelet            Successfully pulled image "velero/velero-plugin-for-aws:v1.10.0" in 2.116s (2.116s including waiting). Image size: 35055239 bytes.
  Normal  Created    6m41s  kubelet            Created container velero-plugin-for-aws
  Normal  Started    6m40s  kubelet            Started container velero-plugin-for-aws
  Normal  Pulled     6m39s  kubelet            Container image "quay.io/k8tz/k8tz:0.16.2" already present on machine
  Normal  Created    6m39s  kubelet            Created container k8tz
  Normal  Started    6m39s  kubelet            Started container k8tz
  Normal  Pulled     6m38s  kubelet            Container image "velero/velero:v1.14.1" already present on machine
  Normal  Created    6m38s  kubelet            Created container velero
  Normal  Started    6m38s  kubelet            Started container velero
```

Now to see if we get another backup at midnight!

## Roll Out

This requires restarting every. fucking. pod. I have never done that so it's a good time to learn.

So far I've applied the change to:

- Sonarr
- Velero (entire namespace)

Says I can use [descheduler](https://github.com/kubernetes-sigs/descheduler) to hit em all. 

Also the following which may be crazy:

```bash
kubectl delete pod --all --all-namespaces
```

Maybe tone it down to:

```bash
kubectl -n media-management delete pod --all
kubectl -n download-clients delete pod --all
```

So far so good. Now that leaves:

| Namespace | Bounced? | 
| authentik | y |
| certificates | y |
| cloudflare | y |
| cloudnative-pg | y |
| external-dns | y |
| flux-system | y |
| haynes-intelligence | y |
| iot-services | y |
| kubernetes-dashboard | y |
| metallb-system | y |
| monitoring | y |
| podinfo | y |
| reflector | y |
| rook-ceph | y |
| sealed-secrets | y |
| traefik | y |
| vikunja | y |

Scary but do-able. And here's how it was done:

```bash
kubectl -n authentik delete pod --all
kubectl -n certificates delete pod --all
kubectl -n cloudflare delete pod --all
kubectl -n cloudnative-pg delete pod --all
kubectl -n external-dns delete pod --all
kubectl -n flux-system delete pod --all
kubectl -n haynes-intelligence delete pod --all
kubectl -n iot-services delete pod --all
kubectl -n kubernetes-dashboard delete pod --all
kubectl -n metallb-system delete pod --all
kubectl -n monitoring delete pod --all
kubectl -n podinfo delete pod --all
kubectl -n reflector delete pod --all
kubectl -n rook-ceph delete pod --all
kubectl -n sealed-secrets delete pod --all
kubectl -n traefik delete pod --all
kubectl -n vikunja delete pod --all
```

And sampling a few I see k8tz creeping in left and right!

```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  58s   default-scheduler  Successfully assigned traefik/traefik-internal-69fd846c8c-5flq6 to kubew02
  Normal  Pulled     59s   kubelet            Container image "quay.io/k8tz/k8tz:0.16.2" already present on machine
  Normal  Created    59s   kubelet            Created container k8tz
  Normal  Started    59s   kubelet            Started container k8tz
  Normal  Pulled     58s   kubelet            Container image "docker.io/traefik:v3.1.2" already present on machine
  Normal  Created    58s   kubelet            Created container traefik-internal
  Normal  Started    58s   kubelet            Started container traefik-internal
```

And nothing broke!!