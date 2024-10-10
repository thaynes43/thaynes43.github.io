---
title: VolSync With Templates
permalink: /docs/home-ops/volsync-with-templates/
---

After the last train wreck VolSync adventure in [Volsync]({{ site.url }}/docs/home-ops/volsync/)

TODO List:

1. External Secrets
1. Create template
1. Follow the rules:

```yaml
# This taskfile is used to manage certain VolSync tasks for a given application, limitations are described below.
#   1. Fluxtomization, HelmRelease, PVC, ReplicationSource all have the same name (e.g. plex)
#   2. ReplicationSource and ReplicationDestination are a Restic repository
#   3. Applications are deployed as either a Kubernetes Deployment or StatefulSet
#   4. Each application only has one PVC that is being replicated
```

1. Make sure we set `targetNamespace: setme` in `ks.yaml` so template stuff has the right namespace
1. Set template values in the `Kustomization` like this:

```yaml
  postBuild:
    substitute:
      APP: *app
      VOLSYNC_CAPACITY: 5Gi
```

> **NOTE** Will have to get rid of the stuff I am switching to the templates and re-create the volumes

## Back from Vacation

While on vacation I set up external secrets using 1Password. I also started to use 1Password as my password manager for my phone and surface tablet but kinda hate it (for now, growing pains I guess).

### Backup From Template

First I need to translate the properties in my working replication source and volsync secret to the template:

```yaml
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: rdt-client-restic-secret
  namespace: download-clients
spec:
  encryptedData:
    AWS_ACCESS_KEY_ID: REDACTED
    AWS_SECRET_ACCESS_KEY: REDACTED
    RESTIC_PASSWORD: REDACTED
    RESTIC_REPOSITORY: REDACTED
  template:
    metadata:
      creationTimestamp: null
      name: rdt-client-restic-secret
      namespace: download-clients
    type: Opaque
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: rdt-client-replica
  namespace: download-clients
spec:
  sourcePVC: rdt-client-config
  trigger:
    #manual: initial-snapshot-09102024
    schedule: "0 */6 * * *"
  restic:
    copyMethod: Snapshot
    pruneIntervalDays: 10
    repository: rdt-client-restic-secret
    cacheCapacity: 4Gi
    storageClassName: ceph-rbd
    volumeSnapshotClassName: ceph-rbd-snapshotter
    retain:
      daily: 10  # Keep most recent from each day for 10 days
      within: 3d # And keep all that were created within 3 days
```

Templated as:

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/external-secrets.io/externalsecret_v1beta1.json
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: "${APP}-volsync-aws"
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: onepassword-connect
  target:
    name: "${APP}-volsync-aws-secret"
    template:
      engineVersion: v2
      data:
        RESTIC_REPOSITORY: "{{ .REPOSITORY_TEMPLATE }}/${APP}"
        RESTIC_PASSWORD: "{{ .RESTIC_PASSWORD }}"
        AWS_ACCESS_KEY_ID: "{{ .AWS_ACCESS_KEY_ID }}"
        AWS_SECRET_ACCESS_KEY: "{{ .AWS_SECRET_ACCESS_KEY }}"
  dataFrom:
    - extract:
        key: aws
    - extract:
        key: volsync-restic
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/volsync.backube/replicationsource_v1alpha1.json
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: "${APP}-aws"
spec:
  sourcePVC: "${APP}"
  trigger:
    schedule: "0 0 * * *"
  restic:
    copyMethod: "${VOLSYNC_COPYMETHOD:-Snapshot}"
    pruneIntervalDays: 7
    repository: "${APP}-volsync-aws-secret"
    volumeSnapshotClassName: "${VOLSYNC_SNAPSHOTCLASS:-ceph-rbd-snapshotter}}"
    cacheCapacity: "${VOLSYNC_CACHE_CAPACITY:-4Gi}"
    storageClassName: "${VOLSYNC_STORAGECLASS:-ceph-rbd}"
    accessModes: ["${VOLSYNC_ACCESSMODES:-ReadWriteOnce}"]
    moverSecurityContext:
      runAsUser: 568
      runAsGroup: 568
      fsGroup: 568
    retain:
      daily: 10  # Keep most recent from each day for 10 days
      within: 3d # And keep all that were created within 3 days
```

And this all ties into the claim's template which must now be used:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "${APP}"
spec:
  accessModes: ["${VOLSYNC_ACCESSMODES:-ReadWriteOnce}"]
  dataSourceRef:
    kind: ReplicationDestination
    apiGroup: volsync.backube
    name: "${APP}-dst"
  resources:
    requests:
      storage: "${VOLSYNC_CAPACITY}"
  storageClassName: "${VOLSYNC_STORAGECLASS:-ceph-rbd}"
```

Which means I'll be blowing over whatever config I setup before and using this claim which should be like this:

```yaml
    persistence:
      config:
        existingClaim: sabnzb-default
        globalMounts:
          - path: /config
            readOnly: false
```

Hmm, but I don't want to blow over Authentik, so either this will require a lot of reconfiguring things or we work out a way to restore into a new claim. For now I'll let the template rip! 

### Restore From Template

I think I will need to add this but IDK yet! The claim defines one too so I'm gonna wait.

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/volsync.backube/replicationdestination_v1alpha1.json
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: "${APP}-dst"
spec:
  trigger:
    manual: restore-once
  restic:
    repository: "${APP}-volsync-aws-secret"
    copyMethod: Snapshot # must be Snapshot
    volumeSnapshotClassName: "${VOLSYNC_SNAPSHOTCLASS:-ceph-rbd-snapshotter}}"
    cacheCapacity: "${VOLSYNC_CACHE_CAPACITY:-8Gi}"
    storageClassName: "${VOLSYNC_STORAGECLASS:-ceph-rbd}}"
    accessModes: ["${VOLSYNC_ACCESSMODES:-ReadWriteOnce}"]
    capacity: "${VOLSYNC_CAPACITY}"
    # moverSecurityContext:
    #   runAsUser: 568
    #   runAsGroup: 568
    #   fsGroup: 568
```

Running into something related to [this](https://volsync.readthedocs.io/en/latest/usage/volume-populator/index.html).

```
  Warning  VolSyncPopulatorReplicationDestinationMissing  3m49s (x2 over 3m49s)  volsync-controller
                        Unable to populate volume: ReplicationDestination.volsync.backube "sabnzb-default-dst" not found
  Normal   Provisioning                                   3m49s                  rook-ceph.rbd.csi.ceph.com_csi-rbdplugin-provisioner-668cdb6cb5-kgz6t_8d2663e0-1a1a-4277-a3de-8d72396f7d78  External provisioner is provisioning volume for claim "download-clients/sabnzb-default"
  Normal   Provisioning                                   3m49s                  external-provisioner
                        Assuming an external populator will provision the volume
  Normal   ExternalProvisioning                           12s (x17 over 3m49s)   persistentvolume-controller
                        Waiting for a volume to be created either by the external provisioner 'rook-ceph.rbd.csi.ceph.com' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and
```

My PVC is waiting to be populated from VolSync but nothing ever created the initial `ReplicationSource`. I may be able to use this later to manually create a replica before moving things and letting it auto-populate. However, right now, IDK. I am going to slap in the `ReplicationDestination` above and see what happens.

A bit more troubleshooting and I found a few extra brackets in the template - thanks to this error:

```
thaynes@HaynesHyperion:~$ k describe replicationdestination -n download-clients sabnzb-default-dst
Name:         sabnzb-default-dst
Namespace:    download-clients
Labels:       app.kubernetes.io/name=sabnzb-default
              kustomize.toolkit.fluxcd.io/name=sabnzb-default
              kustomize.toolkit.fluxcd.io/namespace=flux-system
Annotations:  <none>
API Version:  volsync.backube/v1alpha1
Kind:         ReplicationDestination
Metadata:
  Creation Timestamp:  2024-09-24T02:26:28Z
  Generation:          1
  Resource Version:    23749280
  UID:                 af011757-eb51-4566-9b28-059e224804a4
Spec:
  Restic:
    Access Modes:
      ReadWriteOnce
    Cache Capacity:              8Gi
    Capacity:                    5Gi
    Copy Method:                 Snapshot
    Repository:                  sabnzb-default-volsync-aws-secret
    Storage Class Name:          ceph-rbd}
    Volume Snapshot Class Name:  ceph-rbd-snapshotter}
  Trigger:
    Manual:  restore-once
Status:
  Conditions:
    Last Transition Time:  2024-09-24T02:26:28Z
    Message:               PersistentVolumeClaim "volsync-sabnzb-default-dst-dest" is invalid: spec.storageClassName: Invalid value: "ceph-rbd}": a lowercase RFC 1123 subdomain must consist of lower case alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character (e.g. 'example.com', regex used for validation is '[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*')
    Reason:                Error
    Status:                False
    Type:                  Synchronizing
  Last Sync Start Time:    2024-09-24T02:26:28Z
  Latest Mover Status:
Events:  <none>
```

Once I fixed up the ReplicationDestination and others that had it the pod came up great!

```
thaynes@HaynesHyperion:~$ k -n download-clients get pvc
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
pvc-smb-tank-k8s-download-clients   Bound    pv-smb-tank-k8s-download-clients           100Ti      RWX            smb-tank-k8s   <unset>                 18d
rdt-client-config                   Bound    pvc-96356461-1327-4b7f-ae42-fcf49d18c9c4   4Gi        RWO            ceph-rbd       <unset>                 58m
sabnzb-default                      Bound    pvc-787f1c4f-617f-4469-ac98-8c52bdab493f   5Gi        RWO            ceph-rbd       <unset>                 54s
sabnzb-mediarequests-config         Bound    pvc-471e69d8-e5e9-47f3-b436-8961665a0d37   4Gi        RWO            ceph-rbd       <unset>                 58m
volsync-rdt-client-replica-cache    Bound    pvc-3a9f8121-dc34-4472-b25b-aa27e543ea9e   4Gi        RWO            ceph-rbd       <unset>                 58m
volsync-sabnzb-default-aws-cache    Bound    pvc-39eeeff9-09c4-4959-b40d-cffd013315aa   4Gi        RWO            ceph-rbd       <unset>                 27s
volsync-sabnzb-default-dst-cache    Bound    pvc-b616cb0e-4733-43ae-bf64-433bdfb93ba0   8Gi        RWO            ceph-rbd       <unset>                 54s
```

I couldn't be happier since this created an initial snapshot, moved it to S3, and then restored it as the pvc for my pod. I should now be able to manually create replicas of things I care about, with the same name as the app, before migrating and have everything restore clean.

### minio & openebs

The repo I'm looking at takes frequent snapshots to minio S3 and caches everything on a local disk w/ openebs. I assume openebs for a cache can be used regardless and save precious Ceph storage. Minio must be storing restic to a NAS instead of R2 (in this case, while I'm on AWS) to save money.

Though enticing, I don't think this is critical path, but how hard would it be to implement later is the real question...

### Mass Roll Out to Previously Migrated 'Apps'

#### Rename PVC By Backup & Restore

Gonna see if I can migrate existing data to this template. `rdt-client` has a backup using:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: rdt-client-restic-secret
  namespace: download-clients
spec:
  encryptedData:
    AWS_ACCESS_KEY_ID: SECRET
    AWS_SECRET_ACCESS_KEY: SECRET
    RESTIC_PASSWORD: SECRET
    RESTIC_REPOSITORY: SECRET
  template:
    metadata:
      creationTimestamp: null
      name: rdt-client-restic-secret
      namespace: download-clients
    type: Opaque
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: rdt-client-replica
  namespace: download-clients
spec:
  sourcePVC: rdt-client-config
  trigger:
    #manual: initial-snapshot-09102024
    schedule: "0 */6 * * *"
  restic:
    copyMethod: Snapshot
    pruneIntervalDays: 10
    repository: rdt-client-restic-secret
    cacheCapacity: 4Gi
    storageClassName: ceph-rbd
    volumeSnapshotClassName: ceph-rbd-snapshotter
    retain:
      daily: 10  # Keep most recent from each day for 10 days
      within: 3d # And keep all that were created within 3 days
```

Plan is to nuke this Kustomization and verify the pvc is gone. Then add it back using the template and see if what's in S3 from this config gets pulled down.

Well, I forgot to back it up, so I can only verify with events on the pvc that it pulled back down.

```
  Type     Reason                                               Age                    From
                                  Message
  ----     ------                                               ----                   ----
                                  -------
  Warning  VolSyncPopulatorReplicationDestinationMissing        2m14s (x2 over 2m15s)  volsync-controller
                                  Unable to populate volume: ReplicationDestination.volsync.backube "rdt-client-dst" not found
  Normal   Provisioning                                         2m14s (x2 over 2m14s)  rook-ceph.rbd.csi.ceph.com_csi-rbdplugin-provisioner-668cdb6cb5-kgz6t_8d2663e0-1a1a-4277-a3de-8d72396f7d78  External provisioner is provisioning volume for claim "download-clients/rdt-client"
  Normal   Provisioning                                         2m14s (x2 over 2m14s)  external-provisioner
                                  Assuming an external populator will provision the volume
  Warning  VolSyncPopulatorReplicationDestinationNoLatestImage  114s (x4 over 2m14s)   volsync-controller
                                  Unable to populate volume, waiting for replicationdestination to have latestImage
  Normal   VolSyncPopulatorPVCCreated                           113s                   volsync-controller
                                  Populator pvc created from snapshot volsync-rdt-client-dst-dest-20240925222939
  Normal   ExternalProvisioning                                 112s (x5 over 2m15s)   persistentvolume-controller
                                  Waiting for a volume to be created either by the external provisioner 'rook-ceph.rbd.csi.ceph.com' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.
  Normal   VolSyncPopulatorFinished                             83s (x3 over 97s)      volsync-controller
                                  Populator finished
```    

Really hard to tell... But we can re-try with the last sab.

##### Official Migration Process for 1 PVC

> **TODO** Authentik, the big one, has two!

Take 2!

The process, if successful, could be templated so that you just point to the folder for the migration backups and then the new ones to pull them back down.

Here's what the migration CRDs would be (not templated) for sabnzb-mediarequests:

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/external-secrets.io/externalsecret_v1beta1.json
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: sabnzb-mediarequests-volsync-aws
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: onepassword-connect
  target:
    name: sabnzb-mediarequests-volsync-aws-secret
    template:
      engineVersion: v2
      data:
        RESTIC_REPOSITORY: "{{ .REPOSITORY_TEMPLATE }}/sabnzb-mediarequests"
        RESTIC_PASSWORD: "{{ .RESTIC_PASSWORD }}"
        AWS_ACCESS_KEY_ID: "{{ .AWS_ACCESS_KEY_ID }}"
        AWS_SECRET_ACCESS_KEY: "{{ .AWS_SECRET_ACCESS_KEY }}"
  dataFrom:
    - extract:
        key: aws
    - extract:
        key: volsync-restic
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/volsync.backube/replicationsource_v1alpha1.json
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: sabnzb-mediarequests-aws
spec:
  sourcePVC: sabnzb-mediarequests-config
  trigger:
    manual: 20240925-01
  restic:
    copyMethod: Snapshot
    pruneIntervalDays: 10
    repository: sabnzb-mediarequests-volsync-aws-secret
    cacheCapacity: 4Gi
    storageClassName: ceph-rbd
    volumeSnapshotClassName: ceph-rbd-snapshotter
    moverSecurityContext:
      runAsUser: 568
      runAsGroup: 568
      fsGroup: 568
    retain:
      daily: 10  # Keep most recent from each day for 10 days
      within: 3d # And keep all that were created within 3 days
```

Now we nuke sabnzb-mediarequests:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Pre Flux-Kustomizations
  - ./namespace.yaml
  # Flux-Kustomizations
  - ./rdt-client/ks.yaml
  - ./sabnzb-default/ks.yaml
  #- ./sabnzb-mediarequests/ks.yaml
```

And switch to the template by:

- Deleting the ExternalSecret & ReplicationSource CRDs
- Updating sab's kustomization to use the template instead of those CRDs

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: download-clients
resources:
  - ./helmrelease.yaml
  - ../../../../templates/volsync
```

- Adding vars for the template we just referenced

```yaml
  postBuild:
    substitute:
      APP: *app
      VOLSYNC_CAPACITY: 4Gi
```

- Use an existing claim in the HelmRelease which will be the claim the template creates with the data we pushed to S3

```
    persistence:
      config:
        existingClaim: sabnzb-mediarequests
        globalMounts:
          - path: /config
            readOnly: false
```

And finally make sure to uncomment `- ./sabnzb-mediarequests/ks.yaml` in the namespaces kustomization.

And BOOM without even refreshing the page sab is back to it's previously deleted self.

> **TODO** See if these matter

```
WARNING a minute ago /tank/data/usenet/complete is not writable with special character filenames. This can cause problems.
To prevent all helpful warnings, disable Special setting 'helpful_warnings'.
WARNING a minute ago /tank/data/usenet/incomplete is not writable with special character filenames. This can cause problems.
To prevent all helpful warnings, disable Special setting 'helpful_warnings'.
```

#### Steps for the stuff we don't care about losing data for

Now that we used `download-clients` as test subjects we can just breeze through `media-management` as we already nuked all the volumes in our last failed attempt. 

##### HelmRelease Updates

Make sure reloader is configured:

```yaml
    controllers:
      controller-name:
        annotations:
          reloader.stakater.com/auto: "true"
```

Switch to existing claim:

```yaml
    persistence:
      config:
        existingClaim: <app-name>
        globalMounts:
          - path: /config
            readOnly: false
```

##### Kustomization Updates

Delete pvc and the file if it's there and add:

```yaml
  - ../../../../templates/volsync/appvol
```

##### Flux KS Updates

Fancy YAML:

```yaml
metadata:
  name: &app appName
```

Use fancy here:

```yaml
spec:
  targetNamespace: media-management
  commonMetadata:
    labels:
      app.kubernetes.io/name: *app
```

Add dependency for secrets:

```yaml
  dependsOn:
    - name: external-secrets-stores
```

Fill in template taking the capacity from what we set in the HelmRelease before.

```yaml
  postBuild:
    substitute:
      APP: *app
      VOLSYNC_CAPACITY: 4Gi
```

For CephFS you will need some more stuff:

```yaml
  postBuild:
    substitute:
      APP: *app
      VOLSYNC_CAPACITY: 4Gi
      VOLSYNC_STORAGECLASS: cephfs
      VOLSYNC_SNAPSHOTCLASS: cephfs-snapshotter
      VOLSYNC_ACCESSMODES: ReadWriteMany
```

##### Test The No Backup Required Migration Process

Go up a dir and uncomment the nuked app to re-deploy to the cluster with the new Volsync capable volume.

#### Don't Break Authentik 

Before trying to make Authentik work with VolSync I need to test how I'll backup and restore what I have now. If it goes south I'm banking on Velero to be the hero.

##### Keeping Access To The Cluster

Right now I go through Authentik to auth into the cluster so we don't want to do that anymore!

The ODIC authentication is easily circumvented by editing `~\.kube\config` to change the user to default (see below)

From:

```yaml
contexts:
- context:
    cluster: default
    user: odic
  name: default
```

To:

```yaml
contexts:
- context:
    cluster: default
    user: default
  name: default
```

##### Volsync for Two PVCs

Because we are using lidar-on-steroids we have two PVCs. This template only supports one per flux kustomization so we either need to make another template or manually do it. Or we can use the template for one volume and then CRDs for others? IDK.

For the extra volume template I basically renamed `$APP` and `$VOLSYNC_` to `$EXTRAVOL` and `$EXTRAVOL_`.

Before we can try that we need to recreate the original PVCs and do the manual backup. Since Authentik's are already created we will do that in two steps.

Step 1 - just re-add to the kustomization. I also added the non-template specific changes we did for the other services in.

Step 2 - configure some stuff that needs to persist in Lidar and Deemix

Step 3 - Manually create snapshots

`externalsecret.yaml`
```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/external-secrets.io/externalsecret_v1beta1.json
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: lidarr-on-steroids-volsync-aws
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: onepassword-connect
  target:
    name: lidarr-on-steroids-volsync-aws-secret
    template:
      engineVersion: v2
      data:
        RESTIC_REPOSITORY: "{{ .REPOSITORY_TEMPLATE }}/lidarr-on-steroids"
        RESTIC_PASSWORD: "{{ .RESTIC_PASSWORD }}"
        AWS_ACCESS_KEY_ID: "{{ .AWS_ACCESS_KEY_ID }}"
        AWS_SECRET_ACCESS_KEY: "{{ .AWS_SECRET_ACCESS_KEY }}"
  dataFrom:
    - extract:
        key: aws
    - extract:
        key: volsync-restic
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/external-secrets.io/externalsecret_v1beta1.json
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: deemix-volsync-aws
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: onepassword-connect
  target:
    name: deemix-volsync-aws-secret
    template:
      engineVersion: v2
      data:
        RESTIC_REPOSITORY: "{{ .REPOSITORY_TEMPLATE }}/deemix"
        RESTIC_PASSWORD: "{{ .RESTIC_PASSWORD }}"
        AWS_ACCESS_KEY_ID: "{{ .AWS_ACCESS_KEY_ID }}"
        AWS_SECRET_ACCESS_KEY: "{{ .AWS_SECRET_ACCESS_KEY }}"
  dataFrom:
    - extract:
        key: aws
    - extract:
        key: volsync-restic
```

`replicationsource.yaml`
```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/volsync.backube/replicationsource_v1alpha1.json
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: lidarr-on-steroids-aws
spec:
  sourcePVC: lidarr-config
  trigger:
    manual: 20240927-01
  restic:
    copyMethod: Snapshot
    pruneIntervalDays: 10
    repository: lidarr-on-steroids-volsync-aws-secret
    cacheCapacity: 40Gi
    storageClassName: ceph-rbd
    volumeSnapshotClassName: ceph-rbd-snapshotter
    moverSecurityContext:
      runAsUser: 568
      runAsGroup: 568
      fsGroup: 568
    retain:
      daily: 10  # Keep most recent from each day for 10 days
      within: 3d # And keep all that were created within 3 days
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/volsync.backube/replicationsource_v1alpha1.json
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: deemix-aws
spec:
  sourcePVC: deemix-config
  trigger:
    manual: 20240927-01
  restic:
    copyMethod: Snapshot
    pruneIntervalDays: 10
    repository: deemix-volsync-aws-secret
    cacheCapacity: 2Gi
    storageClassName: ceph-rbd
    volumeSnapshotClassName: ceph-rbd-snapshotter
    moverSecurityContext:
      runAsUser: 568
      runAsGroup: 568
      fsGroup: 568
    retain:
      daily: 10  # Keep most recent from each day for 10 days
      within: 3d # And keep all that were created within 3 days
```

Make sure to add the files to the kustomization.yaml:

```yaml
  - ./externalsecret.yaml
  - ./replicationsource.yaml
```

Now to see if we get backups! And we did!

Time to use those new templates to nuke these claims and pull what I just show to aws.

`ks.yaml` updates to set vars for the templates:

```yaml
  postBuild:
    substitute:
      APP: *app
      VOLSYNC_CAPACITY: 40Gi
      VOLSYNC_CACHE_CAPACITY: 40Gi
      EXTRAVOL: deemix
      EXTRAVOL_CAPACITY: 2Gi
```

`helmrelease.yaml` updates to match volumes to variables:

```yaml
    persistence:
      config:
        existingClaim: lidarr-on-steroids
        globalMounts:
          - path: /config
            readOnly: false
      config-deemix:
        existingClaim: deemix
        globalMounts:
          - path: /config_deemix
            readOnly: false
```

`kustomization.yaml` updates to use the templates:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: media-management
resources:
  - ./helmrelease.yaml
  - ../../../../templates/volsync/appvol
  - ../../../../templates/volsync/extravol
  #- ./ingressroute.yaml
commonLabels:
  app.kubernetes.io/name: lidarr-on-steroids
  app.kubernetes.io/instance: lidarr-on-steroids
```

And now we wait and see.

Woo the data came back down into the new PVC and nothing was lost for either volume!

##### Backup / Restore from Postgres (cloudnative-pg)

I think I can bench this for now. I see people using a single database for many services which has nightly backups set up. I may want this, especially for HAOS and Authentik, but right now I'm not running and databases outside of what comes with the charts.

