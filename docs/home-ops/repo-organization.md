---
title: Repo Organization
permalink: /docs/home-ops/repo-organization/
---

Funky man did not follow the same patters as the wise k8s-at-home founders. Now I need to move everything around. [This post](https://github.com/fluxcd/flux2/discussions/1517#discussioncomment-867558) says you can't just move the damn folder so I gotta jump through hoops.

## Moving the Damn Folder

### Edits in the Repo

> **TODO** Nice folder structure diagrams

### Getting Flux Happy

`flux uninstall`

```bash
GITHUB_TOKEN=REDACTED \
flux bootstrap github \
  --owner=thaynes43 \
  --repository=flux-repo \
  --personal \
  --path "./kubernetes/main/flux" \
```

Spits this shit out

```
► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/example/flux-repo.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ component manifests are up to date
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
✔ public key: REDACTED
✔ configured deploy key "flux-system-main-flux-system-./kubernetes/main/flux" for "https://github.com/example/flux-repo"
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("d1be94775efd7848514f64bdb56b4277bbf17766")
► pushing sync manifests to "https://github.com/example/flux-repo.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for GitRepository "flux-system/flux-system" to be reconciled
✔ GitRepository reconciled successfully
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```
  
### Debugging

At first it looked good but the smb shares got stuck. `kubectl patch pvc {PVC_NAME} -p '{"metadata":{"finalizers":null}}'` took care of the pvc's and then the pv's disappeared despite being set to retain. However, no data was lost and I could just `flux reconcile kustomization smb-storage -n flux-system` them back to life.

Then Traefik lost it's helm installations and could not upgrade! Fortunately, it has no persistence, otherwise I'd have to jump through some major hoops but for now I'm going to try and flux dance it.

Still didn't work, some fucked up shit in there. 

```bash
thaynes@kubem01:~/workspace$ helm uninstall --namespace traefik traefik-external
release "traefik-external" uninstalled

thaynes@kubem01:~/workspace$ helm uninstall --namespace traefik traefik-internal
release "traefik-internal" uninstalled
```

Closer but stuck on a token I rolled back that hung around until I blasted this shit:

```
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  2m10s                 default-scheduler  Successfully assigned traefik/traefik-external-59496c4f8-9wl88 to kubew01
  Normal   Pulled     2m10s                 kubelet            Container image "quay.io/k8tz/k8tz:0.16.2" already present on machine
  Normal   Created    2m10s                 kubelet            Created container k8tz
  Normal   Started    2m10s                 kubelet            Started container k8tz
  Normal   Pulled     10s (x10 over 2m10s)  kubelet            Container image "docker.io/traefik:v3.1.2" already present on machine
  Warning  Failed     10s (x10 over 2m10s)  kubelet            Error: configmap "kubedash-auth-token" not found
```

That was a real issue though and was missed when I removed the secret. After deleting the env reference to it we're back in business!

Except Vikunja... Vikunja is fucked too but has persistence!

```bash
thaynes@kubem01:~/workspace$ k -n vikunja get helmrelease
NAME      AGE   READY   STATUS
vikunja   34m   False   Helm upgrade failed for release vikunja/vikunja with chart vikunja@15.2.8: "vikunja" has no deployed releases
```

Same gig as Traefik (what's up with these guys!)

```bash
thaynes@kubem01:~/workspace$ helm uninstall --namespace vikunja vikunja
release "vikunja" uninstalled
```

Now maybe we can Velero the release back?

```bash
velero create restore --from-backup velero-hayneslab-backups-20240904000019 --include-namespaces vikunja --wait
```

Seems kinda stuck. Maybe I can remove it, then re-add it so it starts clean, but then restore it to get the volumes. Gonna nuke it.

Gonna just let it time out and deal with this later, not sure why this one is so stuck but it's not essential.

Can't restore it but almost there from scratch - it hates the IPs:

```
2024-09-05 07:24:10	{"caller":"service.go:143","error":"IPFamilyForAddresses: same address family [\"192.168.40.107\" \"192.168.40.108\"]","level":"error","msg":"IP allocation failed","op":"allocateIPs","ts":"2024-09-05T07:24:10-04:00"}
2024-09-05 07:24:10	{"caller":"service_controller.go:112","controller":"ServiceReconciler","endpoints":"[]","event":"failed to handle service","level":"error","name":"vikunja/vikunja","service":"{\"kind\":\"Service\",\"apiVersion\":\"v1\",\"metadata\":{\"name\":\"vikunja\",\"namespace\":\"vikunja\",\"uid\":\"9e134557-4510-4ad1-9eb2-73ff2d914301\",\"resourceVersion\":\"12724805\",\"creationTimestamp\":\"2024-09-05T11:24:10Z\",\"labels\":{\"app\":\"vikunja-15.2.8\",\"app.kubernetes.io/instance\":\"vikunja\",\"app.kubernetes.io/managed-by\":\"Helm\",\"app.kubernetes.io/name\":\"vikunja\",\"app.kubernetes.io/version\":\"0.24.2\",\"helm-revision\":\"1\",\"helm.sh/chart\":\"vikunja-15.2.8\",\"helm.toolkit.fluxcd.io/name\":\"vikunja\",\"helm.toolkit.fluxcd.io/namespace\":\"vikunja\",\"release\":\"vikunja\",\"service.name\":\"main\"},\"annotations\":{\"meta.helm.sh/release-name\":\"vikunja\",\"meta.helm.sh/release-namespace\":\"vikunja\",\"metallb.universe.tf/allow-shared-ip\":\"vikunja\",\"metallb.universe.tf/loadBalancerIPs\":\"192.168.40.107,192.168.40.108\"},\"managedFields\":[{\"manager\":\"helm-controller\",\"operation\":\"Update\",\"apiVersion\":\"v1\",\"time\":\"2024-09-05T11:24:10Z\",\"fieldsType\":\"FieldsV1\",\"fieldsV1\":{\"f:metadata\":{\"f:annotations\":{\".\":{},\"f:meta.helm.sh/release-name\":{},\"f:meta.helm.sh/release-namespace\":{},\"f:metallb.universe.tf/allow-shared-ip\":{},\"f:metallb.universe.tf/loadBalancerIPs\":{}},\"f:labels\":{\".\":{},\"f:app\":{},\"f:app.kubernetes.io/instance\":{},\"f:app.kubernetes.io/managed-by\":{},\"f:app.kubernetes.io/name\":{},\"f:app.kubernetes.io/version\":{},\"f:helm-revision\":{},\"f:helm.sh/chart\":{},\"f:helm.toolkit.fluxcd.io/name\":{},\"f:helm.toolkit.fluxcd.io/namespace\":{},\"f:release\":{},\"f:service.name\":{}}},\"f:spec\":{\"f:allocateLoadBalancerNodePorts\":{},\"f:externalTrafficPolicy\":{},\"f:internalTrafficPolicy\":{},\"f:ports\":{\".\":{},\"k:{\\\"port\\\":10220,\\\"protocol\\\":\\\"TCP\\\"}\":{\".\":{},\"f:name\":{},\"f:port\":{},\"f:protocol\":{},\"f:targetPort\":{}}},\"f:selector\":{},\"f:sessionAffinity\":{},\"f:type\":{}}}}]},\"spec\":{\"ports\":[{\"name\":\"main\",\"protocol\":\"TCP\",\"port\":10220,\"targetPort\":10220}],\"selector\":{\"app.kubernetes.io/instance\":\"vikunja\",\"app.kubernetes.io/name\":\"vikunja\",\"pod.name\":\"main\"},\"clusterIP\":\"10.43.83.73\",\"clusterIPs\":[\"10.43.83.73\"],\"type\":\"LoadBalancer\",\"sessionAffinity\":\"None\",\"externalTrafficPolicy\":\"Cluster\",\"ipFamilies\":[\"IPv4\"],\"ipFamilyPolicy\":\"SingleStack\",\"allocateLoadBalancerNodePorts\":false,\"internalTrafficPolicy\":\"Cluster\"},\"status\":{\"loadBalancer\":{}}}","ts":"2024-09-05T07:24:10-04:00"}
```

> **WARNING** NEVER CHANGE THE PATH

## Adding aps Folder

Now to clean up the big chunk of files in the flux folder by spreading them across namespace then apps folder. I started with the structure kubernetes\main\apps and will then have a folder per namespace.

### IoT-Services

First I added a folder for the `iot-services` namespace stuff.

I added a kustomization in the flux repo to point to the apps folder:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cluster-apps
  namespace: flux-system
spec:
  interval: 10m
  path: ./kubernetes/main/apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

First try did not work:

```
cluster-apps                 13s   False   kustomize build failed: accumulating resources: accumulation err='accumulating resources from './iot-services': read /tmp/kustomization-107764831/kubernetes/main/apps/iot-services: is a directory': recursed accumulation of path '/tmp/kustomization-107764831/kubernetes/main/apps/iot-services': accumulating resources: accumulation err='accumulating resources from './mosquito/ks.yaml': open /tmp/kustomization-107764831/kubernetes/main/apps/iot-services/mosquito/ks.yaml: no such file or directory': must build at directory: not a valid directory: evalsymlink failure on '/tmp/kustomization-107764831/kubernetes/main/apps/iot-services/mosquito/ks.yaml' : lstat /tmp/kustomization-107764831/kubernetes/main/apps/iot-services/mosquito: no such file or directory
```

This one was an easy typo, `mosquito` vs `mosquitto`. Second try was a charm. Kustomizations deployed as if new:

```
cluster-apps                 8m      True    Applied revision: main@sha1:084ddf05d3423486d4c564e1ec5d51205c80430c
...
mosquitto                    4m57s   True    Applied revision: main@sha1:084ddf05d3423486d4c564e1ec5d51205c80430c
zigbee2mqtt                  4m57s   True    Applied revision: main@sha1:084ddf05d3423486d4c564e1ec5d51205c80430c
zwave-js-ui                  4m57s   True    Applied revision: main@sha1:084ddf05d3423486d4c564e1ec5d51205c80430c
```

But this recreated the volumes which wasn't ideal but there's not much to lose yet:

```
pvc-11310270-108d-40e0-96ea-4b88b12db78b   2Gi        RWO            Delete           Bound    iot-services/zwave-js-ui-store                       ceph-rbd       <unset>                          7m27s
```

Velero to the rescue!!!!

```bash
velero create restore --from-backup velero-hayneslab-backups-20240906000021 --include-namespaces iot-services --wait
```

This time it worked, unlike Vikunja's database, so we are one for two now not counting the initial tests w/ Open WebUI. However, if I'm gonna change all these folders I will need a safe way to do so without losing all the data. Since Vikunja is already fucked I can add some stuff to that, turn off prune, and see if sticks around. 

### IoT-Services PVCs

To follow the convention I need to also make seperate pvcs and then reference them as existing claims. I do not think I wil be able to save my data, at least not in a way that was worth the time since very little is set up.



### Vikunja

Before moving Vikunja I tried setting `prune: false` in hopes that my volumes would stick around this time.

Actually, changed my mind. I'm going to wipe out the vikunja namespace and re-name it to one that represents all productivity tools / services as I want to add some more specifically around quick / sticky notes.

Few mis-steps in changing the namespace, like not changing it everywhere, but that worked (with data loss).

#### Databases

Since I did Vikunja I'm going to quickly get `cloudnative-pg` over but also make it a new namespace called `databases`. This way I can throw other databases and database stuff in there that isn't tied to an app. I need a MySQL soon anyway to hold the worlds knowledge.

I had to recreate it's CRDs since they were namespaced but that was the only surprise. 

But now Vikunja is stuck:

```
thaynes@kubem01:~$ helm history vikunja --namespace productivity
REVISION	UPDATED                 	STATUS      	CHART         	APP VERSION	DESCRIPTION                              
1       	Fri Sep  6 10:57:16 2024	uninstalling	vikunja-15.2.8	0.24.2     	Deletion in progress (or silently failed)
```

Fix it up with `helm uninstall vikunja --namespace productivity`.

Now:

```bash
thaynes@kubem01:~$ helm history vikunja --namespace productivity
Error: release: not found
```

Moved the IngressRoute into Vikunja too but named the file `kustomize.yaml` instead of `kustomization.yaml` and it blew up so file name seems to matter here!

### k8tz

This one was super easy. I made a `admission-controller` directory for it but did not change any namespaces.

### Media Management

Now for one with PV's to lose. Gonna disable "prune" and see if that keeps them around.

This is a huge one:

```bash
$ git status
On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        deleted:    kubernetes/main/flux/kustomizations/kustomization-media-management.yaml
        deleted:    kubernetes/main/flux/namespaces/namespace-media-management.yaml
        deleted:    kubernetes/main/media-management/exportarr/manifest-radarr-exporter.yaml
        deleted:    kubernetes/main/media-management/exportarr/sealedsecret-radarr-api-key.yaml
        deleted:    kubernetes/main/media-management/prowlarr/helmrelease-prowlarr.yaml
        deleted:    kubernetes/main/media-management/request-management/helmrelease-overseerr.yaml
        deleted:    kubernetes/main/media-management/servarr/helmrelease-bazarr.yaml
        deleted:    kubernetes/main/media-management/servarr/helmrelease-lidarr-on-steroids.yaml
        deleted:    kubernetes/main/media-management/servarr/helmrelease-mylar3.yaml
        deleted:    kubernetes/main/media-management/servarr/helmrelease-radarr.yaml
        deleted:    kubernetes/main/media-management/servarr/helmrelease-readarr-audiobooks.yaml
        deleted:    kubernetes/main/media-management/servarr/helmrelease-readarr-ebooks.yaml
        deleted:    kubernetes/main/media-management/servarr/helmrelease-sonarr.yaml

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        kubernetes/main/apps/media-management/
```

Still need Sonarr's ingress route but I can worry about that later.

Insta fail:

```
cluster-apps                 7h21m   False   kustomize build failed: accumulating resources: accumulation err='accumulating resources from './media-management': read /tmp/kustomization-2096575925/kubernetes/main/apps/media-management: is a directory': recursed accumulation of path '/tmp/kustomization-2096575925/kubernetes/main/apps/media-management': accumulating resources: accumulation err='accumulating resources from './servarr/ks.yaml': open /tmp/kustomization-2096575925/kubernetes/main/apps/media-management/servarr/ks.yaml: no such file or directory': must build at directory: not a valid directory: evalsymlink failure on '/tmp/kustomization-2096575925/kubernetes/main/apps/media-management/servarr/ks.yaml' : lstat /tmp/kustomization-2096575925/kubernetes/main/apps/media-management/servarr: no such file or directory
```

Took my eyes a while to see but `sevarr` was the name of the directory while `servarr` was configured. Fixed it up and tried again. 

The volumes are all gone, not having much luck at all there. I think I need to add this before moving the file with the namespace:

```yaml
  labels:
    kustomize.toolkit.fluxcd.io/prune: disabled
```

But first - new problem:

```
bjw-s                  https://bjw-s.github.io/helm-charts                                             45h   False   failed to fetch Helm repository index: failed to cache index to temporary file: failed to fetch https://bjw-s.github.io/helm-charts/index.yaml : 404 Not Found
```

This was fixed with a temporary change suggested in [this issue](https://github.com/bernd-schorgers/helm-charts/issues/355)

Last problem:

```
exportarr                    5h11m   False   health check failed after 16.899088ms: failed early due to stalled resources: [Deployment/media-management/radarr-exporter status: 'Failed']
```

Which I think is because I slapped the example one I did in the radarr namespace. Will need to revisit these later but I was overriding all namespaces in the kustomization to media-management.

#### Media Management PVCs

I see a lot of `VOLSYNC_CLAIM` specified everywhere. It looks like the dark magic I seek to piece this all together. Unfortunately I believe it means a lot of work for me, these values are using existing claims that are created before the HelmRelease or even pulled from volsync if they are missing. 

First I am going to try making a claim for sonarr's configs:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarr-config
  namespace: media-management
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 15Gi
  storageClassName: ceph-rbd
```

Then just using that instead of in the values:

```yaml
    persistence:
      config:
        type: persistentVolumeClaim
        existingClaim: sonarr-config
        globalMounts:
          - path: /config
            readOnly: false
```

Old PVC got stuck, needs a patch:

```bash
k -n media-management patch pvc sonarr-config -p '{"metadata":{"finalizers":null}}'
```

Now that worked but we got two pv's!

```
thaynes@kubem01:~$ k get pv | grep sonarr
pvc-46932b91-7d0f-48fb-b4a7-a5b20bed8a90   15Gi       RWO            Delete           Released   media-management/sonarr-config                       ceph-rbd       <unset>                          145m
pvc-bbcef621-1ba2-4c69-982f-aa32c64264ef   15Gi       RWO            Delete           Bound      media-management/sonarr-config                       ceph-rbd       <unset>                          87s
```

```bash
kubectl patch pv pvc-46932b91-7d0f-48fb-b4a7-a5b20bed8a90 -p '{"spec":{"claimRef": null}}'
```

From there I will need something like this to back it up with volosync but we will need to get that all set up first which may take the weekend...

`volosync.yaml`
```yaml
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: sonarr-restic
  namespace: media-management
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: onepassword-connect
  target:
    name: sonarr-restic-secret
    creationPolicy: Owner
    template:
      engineVersion: v2
      data:
        RESTIC_REPOSITORY: "{{ .REPOSITORY_TEMPLATE }}/sonarr"
        RESTIC_PASSWORD: "{{ .RESTIC_PASSWORD }}"
        AWS_ACCESS_KEY_ID: "{{ .AWS_ACCESS_KEY_ID }}"
        AWS_SECRET_ACCESS_KEY: "{{ .AWS_SECRET_ACCESS_KEY }}"
  dataFrom:
    - extract:
        key: volsync-restic-template
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: sonarr
  namespace: media-management
spec:
  sourcePVC: sonarr-config
  trigger:
    schedule: "0 */12 * * *"
  restic:
    copyMethod: Snapshot
    pruneIntervalDays: 14
    repository: readarr-restic-secret
    #moverSecurityContext:
    #  runAsUser: 568
    #  runAsGroup: 568
    #  fsGroup: 568
    retain:
      daily: 14
    volumeSnapshotClassName: csi-ceph-blockpool
    storageClassName: ceph-rbd
    cacheStorageClassName: local-hostpath
```

### Download Clients