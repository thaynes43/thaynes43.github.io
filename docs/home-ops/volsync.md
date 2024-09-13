---
title: VolSync
permalink: /docs/home-ops/volsync/
---

## Snapshot Dependencies

This one is quite confusing. Starting with the [official docs](https://volsync.readthedocs.io/en/stable/installation/index.html) I immediatly can't get past step two. 

```
thaynes@kubem01:~/workspace/volume-setup$ kubectl get volumesnapshotclasses
No resources found
```

Seemed as easy as enabling in the helm values for `rook-ceph-cluster`:

```yaml
    cephFileSystemVolumeSnapshotClass:
      enabled: true
      name: cephfs

    cephBlockPoolsVolumeSnapshotClass:
      enabled: true
      name: ceph-rbd
```

This got them going!

```
thaynes@kubem01:~/workspace/volume-setup$ kubectl get volumesnapshotclasses
NAME       DRIVER                          DELETIONPOLICY   AGE
ceph-rbd   rook-ceph.rbd.csi.ceph.com      Delete           12s
cephfs     rook-ceph.cephfs.csi.ceph.com   Delete           12s
```

Later we will need more details fo these classes:

### ceph-rbd VolumeSnapshotClass

```
thaynes@kubem01:~/workspace/volume-setup$ k describe volumesnapshotclass ceph-rbd
Name:             ceph-rbd
Namespace:        
Labels:           app.kubernetes.io/managed-by=Helm
                  helm.toolkit.fluxcd.io/name=rook-ceph-cluster
                  helm.toolkit.fluxcd.io/namespace=rook-ceph
Annotations:      meta.helm.sh/release-name: rook-ceph-cluster
                  meta.helm.sh/release-namespace: rook-ceph
                  snapshot.storage.kubernetes.io/is-default-class: false
API Version:      snapshot.storage.k8s.io/v1
Deletion Policy:  Delete
Driver:           rook-ceph.rbd.csi.ceph.com
Kind:             VolumeSnapshotClass
Metadata:
  Creation Timestamp:  2024-09-08T15:38:11Z
  Generation:          1
  Resource Version:    14560973
  UID:                 109b2020-969f-4881-809d-769c6859cdff
Parameters:
  Cluster ID:                                       rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name:       rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace:  rook-ceph
Events:                                             <none>
```

### cephfs VolumeSnapshotClass

```
thaynes@kubem01:~/workspace/volume-setup$ k describe volumesnapshotclass cephfs
Name:             cephfs
Namespace:        
Labels:           app.kubernetes.io/managed-by=Helm
                  helm.toolkit.fluxcd.io/name=rook-ceph-cluster
                  helm.toolkit.fluxcd.io/namespace=rook-ceph
Annotations:      meta.helm.sh/release-name: rook-ceph-cluster
                  meta.helm.sh/release-namespace: rook-ceph
                  snapshot.storage.kubernetes.io/is-default-class: true
API Version:      snapshot.storage.k8s.io/v1
Deletion Policy:  Delete
Driver:           rook-ceph.cephfs.csi.ceph.com
Kind:             VolumeSnapshotClass
Metadata:
  Creation Timestamp:  2024-09-08T15:38:11Z
  Generation:          1
  Resource Version:    14560972
  UID:                 225405d0-e19e-4f4e-997b-ee47fc6c5258
Parameters:
  Cluster ID:                                       rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name:       rook-csi-cephfs-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace:  rook-ceph
Events:                                             <none>
```

### Snapshot Controller

We may or may not need this because we manually installed some CRDs but it will help long term if we need to bootstrap a new cluster.

#### Flus Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: snapshot-controller
  namespace: flux-system
spec:
  interval: 15m
  retryInterval: 1m
  timeout: 2m
  prune: true
  wait: true
  dependsOn:
    - name: rook-ceph
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./kubernetes/main/apps/system/storage-controllers/snapshot-controller/app
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: snapshot-controller
      namespace: snapshot-controller
```

#### HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: snapshot-controller
  namespace: backup-controllers
spec:
  interval: 15m
  chart:
    spec:
      # https://github.com/piraeusdatastore/helm-charts/tree/main/charts/snapshot-controller
      chart: snapshot-controller
      version: 3.0.6
      sourceRef:
        kind: HelmRepository
        name: piraeus
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
    controller:
      serviceMonitor:
        create: true
    webhook:
      enabled: false
```

### VolSync

This one requires some research but may not be too bad. [Manual Triggers](https://volsync.readthedocs.io/en/stable/usage/triggers.html) look interesting.

Getting up and running is nothing but it gets complicated for restoring things dynamically.

#### Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: volsync
  namespace: flux-system
spec:
  interval: 15m
  retryInterval: 1m
  timeout: 2m
  prune: true
  wait: true
  dependsOn:
    - name: rook-ceph
    - name: snapshot-controller
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./kubernetes/main/apps/system/backup-controllers/volsync/app
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: volsync
      namespace: backup-controllers
```

#### HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: volsync
  namespace: backup-controllers
spec:
  interval: 15m
  timeout: 5m
  chart:
    spec:
      chart: volsync
      version: 0.10.0
      sourceRef:
        kind: HelmRepository
        name: backube
        namespace: flux-system
  maxHistory: 3
  install:
    createNamespace: true
    remediation:
      retries: 3
  upgrade:
    cleanupOnFail: true
    remediation:
      retries: 3
  uninstall:
    keepHistory: false
  values:
    manageCRDs: true
    metrics:
      disableAuth: true
```

#### My First Volsync

Now we need a `Secret` and a `ReplicationSource` to get a volume backed up. For this test we are going to go with Sonarr.

VolSync docs state `restic init` will be ran to initialize the repo on the first backup so we shouldn't have to worry about that.

##### Restic Secret

> **TODO** This will get much easier if I move to `external-secrets` but for now we be sealing.

First create a secret to seal:

`volsync-sonarr-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sonarr-restic-secret
  namespace: media-management
type: Opaque
stringData:
  RESTIC_REPOSITORY: s3:s3-us-east-2.amazonaws.com/s3-volsync/sonarr
  RESTIC_PASSWORD: SAMEASVELERO
  AWS_ACCESS_KEY_ID: REDACTED
  AWS_SECRET_ACCESS_KEY: REDACTED
```

Then seal it from the file:

```bash
cat volsync-sonarr-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-volsync-sonarr-secret.yaml
```

And now we have the sealed version:

`sealedsecret-volsync-sonarr-secret.yaml`
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: sonarr-restic-secret
  namespace: media-management
spec:
  encryptedData:
    AWS_ACCESS_KEY_ID: REDACTED
    AWS_SECRET_ACCESS_KEY: REDACTED
    RESTIC_PASSWORD: REDACTED
    RESTIC_REPOSITORY: REDACTED
  template:
    metadata:
      creationTimestamp: null
      name: sonarr-restic-secret
      namespace: media-management
    type: Opaque
```

##### ReplicationSource

Now the magic that creates the backup. To start I am going to make a manual trigger and just pop one off. [This doc](https://volsync.readthedocs.io/en/stable/usage/restic/index.html) explains everything very well.

Retain is a bit complicated but [this doc](https://restic.readthedocs.io/en/stable/060_forget.html#removing-snapshots-according-to-a-policy) explains how you can keep a variety which seems to let you spread out the purge. The value for each of these is the number of time units (?) to keep one of each type. So for `retain.monthly=12` it will keep the most recent snapshot for the last 12 months. This way you can go futher back in time with a sparse set of backups. 

For the cron [this tool](https://crontab.guru/) is helpful but llama3.1 nailed it before I could.

`replicationsource.yaml`
```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: sonarr-config-replica
  namespace: media-management
spec:
  sourcePVC: sonarr-config
  trigger:
    manual: first-snapshot
    schedule: "0 */6 * * *"
  restic:
    copyMethod: Snapshot
    pruneIntervalDays: 10
    repository: sonarr-restic-secret
    cacheCapacity: 10Gi
    storageClassName: ceph-rbd
    volumeSnapshotClassName: ceph-rbd
    retain:
      daily: 10  # Keep most recent from each day for 10 days
      within: 3d # And keep all that were created within 3 days
```

##### Let It Rip!

Well small snag, it doesn't like the secret name the snapshot class is configured for:

```
2024-09-08 21:23:52	E0908 21:23:52.472090       1 snapshot_controller.go:141] checkandUpdateContentStatus [snapcontent-0e54e8c5-fdd8-44fc-94a3-7e5b15b280dd]: error occurred failed to get input parameters to create snapshot for content snapcontent-0e54e8c5-fdd8-44fc-94a3-7e5b15b280dd: "cannot get credentials for snapshot content \"snapcontent-0e54e8c5-fdd8-44fc-94a3-7e5b15b280dd\""
2024-09-08 21:23:52	E0908 21:23:52.472096       1 snapshot_controller_base.go:359] could not sync content "snapcontent-0e54e8c5-fdd8-44fc-94a3-7e5b15b280dd": failed to get input parameters to create snapshot for content snapcontent-0e54e8c5-fdd8-44fc-94a3-7e5b15b280dd: "cannot get credentials for snapshot content \"snapcontent-0e54e8c5-fdd8-44fc-94a3-7e5b15b280dd\""
2024-09-08 21:24:56	I0908 21:24:56.472525       1 snapshot_controller.go:307] createSnapshotWrapper: Creating snapshot for content snapcontent-0e54e8c5-fdd8-44fc-94a3-7e5b15b280dd through the plugin ...
```

These are the real names:

```
thaynes@kubem01:~/workspace$ k -n rook-ceph get secrets
NAME                                             TYPE                 DATA   AGE
rook-ceph-config                                 kubernetes.io/rook   2      26d
rook-ceph-mon                                    kubernetes.io/rook   6      26d
rook-csi-cephfs-node-default-k8s-cephfs          kubernetes.io/rook   2      26d
rook-csi-cephfs-provisioner-default-k8s-cephfs   kubernetes.io/rook   2      26d
rook-csi-rbd-node-default-k8s-disks              kubernetes.io/rook   2      26d
rook-csi-rbd-provisioner-default-k8s-disks       kubernetes.io/rook   2      26d
```

And here's what is in the volumesnapshotclass:

```
thaynes@kubem01:~/workspace$ k describe volumesnapshotclass ceph-rbd
Name:             ceph-rbd
Namespace:        
Labels:           app.kubernetes.io/managed-by=Helm
                  helm.toolkit.fluxcd.io/name=rook-ceph-cluster
                  helm.toolkit.fluxcd.io/namespace=rook-ceph
Annotations:      meta.helm.sh/release-name: rook-ceph-cluster
                  meta.helm.sh/release-namespace: rook-ceph
                  snapshot.storage.kubernetes.io/is-default-class: false
API Version:      snapshot.storage.k8s.io/v1
Deletion Policy:  Delete
Driver:           rook-ceph.rbd.csi.ceph.com
Kind:             VolumeSnapshotClass
Metadata:
  Creation Timestamp:  2024-09-08T15:38:11Z
  Generation:          1
  Resource Version:    14560973
  UID:                 109b2020-969f-4881-809d-769c6859cdff
Parameters:
  Cluster ID:                                       rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name:       rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace:  rook-ceph
Events:                                             <none>
```

So, I just need to set the parameter! Fortunately this is a blank flow mapping in [the default values](https://github.com/rook/rook/blob/master/deploy/charts/rook-ceph-cluster/values.yaml).


```yaml
    cephFileSystemVolumeSnapshotClass:
      enabled: true
      name: cephfs
      parameters:
        csi.storage.k8s.io/snapshotter-secret-name: rook-csi-cephfs-provisioner-default-k8s-cephfs

    cephBlockPoolsVolumeSnapshotClass:
      enabled: true
      name: ceph-rbd
      parameters:
        csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner-default-k8s-disks
```

Maybe?

Nope:

```
2024-09-09T01:42:54.386110421Z: resetting values to the chart's original version
  Warning  UpgradeFailed  75s  helm-controller  Helm upgrade failed for release rook-ceph/rook-ceph-cluster with chart rook-ceph-cluster@v1.14.10: error while running post render on files: map[string]interface {}(nil): yaml: unmarshal errors:
  line 16: mapping key "csi.storage.k8s.io/snapshotter-secret-name" already defined at line 14
```

Gonna try and just define them on the side:

```yaml
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ceph-rbd-snapshotter
driver: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner-default-k8s-disks
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
deletionPolicy: Delete
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: cephfs-snapshotter
driver: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-cephfs-provisioner-default-k8s-cephfs
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
deletionPolicy: Delete
```

Still stuck

```
thaynes@kubem01:~/workspace/secret-sealing$ k get volumesnapshots -A
NAMESPACE          NAME                                READYTOUSE   SOURCEPVC       SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS   SNAPSHOTCONTENT                                    CREATIONTIME   AGE
media-management   volsync-sonarr-config-replica-src   false        sonarr-config                                         ceph-rbd        snapcontent-0e54e8c5-fdd8-44fc-94a3-7e5b15b280dd                  49m
```

And now the cluster is deleting!!!

```
thaynes@kubem01:~/workspace/secret-sealing$ kubectl -n rook-ceph get CephCluster
NAME        DATADIRHOSTPATH   MONCOUNT   AGE   PHASE      MESSAGE                    HEALTH      EXTERNAL   FSID
rook-ceph   /var/lib/rook     3          26d   Deleting   Deleting the CephCluster   HEALTH_OK   true       9b0628e1-1fe9-49d2-b65b-746d05215e3d
```

Zero idea why. `kubectl edit volumesnapshot -n media-management volsync-sonarr-config-replica-src` to fix the stuck snapshot first. 

```bash
kubectl delete namespace rook-ceph
velero create restore --from-backup velero-hayneslab-backups-20240907000022 --include-namespaces rook-ceph --wait
```

Something is stuck with the damn finalizers again!

```
kubectl get all,cm,secret,ing -n rook-ceph
for i in `kubectl api-resources | awk '{print $1}'`; do kubectl get $i; done
```

Then I found this beast:

```bash
function kubectlgetall {
  for i in $(kubectl api-resources --verbs=list --namespaced -o name | grep -v "events.events.k8s.io" | grep -v "events" | sort | uniq); do
    echo "Resource:" $i
    
    if [ -z "$1" ]
    then
        kubectl get --ignore-not-found ${i}
    else
        kubectl -n ${1} get --ignore-not-found ${i}
    fi
  done
}
```

Usage: `kubectlgetall rook-ceph`

And found I had to delete a ConfigMap and the CephCluster itself.

And that salvaged whatever the hell wen't wrong:

```
thaynes@kubem01:~$ kubectl -n rook-ceph get CephCluster
NAME        DATADIRHOSTPATH   MONCOUNT   AGE    PHASE       MESSAGE                          HEALTH      EXTERNAL   FSID
rook-ceph   /var/lib/rook     3          118s   Connected   Cluster connected successfully   HEALTH_OK   true       9b0628e1-1fe9-49d2-b65b-746d05215e3d
```

It's still trying to restore, I think this backup wants the snapshot dudes. 

That will work itself out. Now I want to very carefully add back in the snapshotter. Fortunately that went fine so it's time for a manual test before we get fancy.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: sonarr-config-snapshot-test
  namespace: media-management
spec:
  volumeSnapshotClassName: ceph-rbd-snapshotter
  source:
    persistentVolumeClaimName: sonarr-config
```

And we have a winner!

```
thaynes@kubem01:~/workspace/volume-setup$ k get volumesnapshot -n media-management 
NAME                          READYTOUSE   SOURCEPVC       SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS          SNAPSHOTCONTENT                                    CREATIONTIME   AGE
sonarr-config-snapshot-test   true         sonarr-config                           15Gi          ceph-rbd-snapshotter   snapcontent-ce550f3c-a623-45f1-b74d-8a9650224a69   8s             8s
```

An no problem adding Cephfs-snapshotter:

```
thaynes@kubem01:~/workspace/volume-setup$ kubectl get volumesnapshotclasses
NAME                   DRIVER                       DELETIONPOLICY   AGE
ceph-rbd-snapshotter   rook-ceph.rbd.csi.ceph.com   Delete           11m
cephfs-snapshotter     rook-ceph.rbd.csi.ceph.com   Delete           3m27s
```

For good measure:

```
thaynes@kubem01:~/workspace/volume-setup$ kubectl -n rook-ceph get CephCluster
NAME        DATADIRHOSTPATH   MONCOUNT   AGE   PHASE       MESSAGE                          HEALTH      EXTERNAL   FSID
rook-ceph   /var/lib/rook     3          31m   Connected   Cluster connected successfully   HEALTH_OK   true       9b0628e1-1fe9-49d2-b65b-746d05215e3d
```

Now to test that:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: open-webui-snapshot-test
  namespace: haynes-intelligence
spec:
  volumeSnapshotClassName: cephfs-snapshotter
  source:
    persistentVolumeClaimName: open-webui
```

And this one is stuck:

```
E0908 23:26:55.960159       1 snapshot_controller_base.go:359] could not sync content "snapcontent-6e14d4bd-f5e7-49a1-9494-9e936aedb184": failed to take snapshot of the volume 0001-0009-rook-ceph-0000000000000004-f0198da4-4204-4727-bfc9-11f836bb52da: "rpc error: code = Internal desc = missing ID field 'userID' in secrets"
```

Let's see:

```
thaynes@kubem01:~/workspace/volume-setup$ k describe volumesnapshotclasses.snapshot.storage.k8s.io cephfs-snapshotter 
Name:             cephfs-snapshotter
Namespace:        
Labels:           kustomize.toolkit.fluxcd.io/name=rook-ceph-cluster
                  kustomize.toolkit.fluxcd.io/namespace=flux-system
Annotations:      <none>
API Version:      snapshot.storage.k8s.io/v1
Deletion Policy:  Delete
Driver:           rook-ceph.rbd.csi.ceph.com
Kind:             VolumeSnapshotClass
Metadata:
  Creation Timestamp:  2024-09-09T03:19:45Z
  Generation:          1
  Resource Version:    14855824
  UID:                 1278842d-5883-4f3a-bec1-61fc1307eb99
Parameters:
  Cluster ID:                                       rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name:       rook-csi-cephfs-provisioner-default-k8s-cephfs
  csi.storage.k8s.io/snapshotter-secret-namespace:  rook-ceph
Events:                                             <none>
```

And yeah it has admin shit not user:

```
thaynes@kubem01:~/workspace/volume-setup$ k -n rook-ceph describe secret rook-csi-cephfs-provisioner-default-k8s-cephfs
Name:         rook-csi-cephfs-provisioner-default-k8s-cephfs
Namespace:    rook-ceph
Labels:       velero.io/backup-name=velero-hayneslab-backups-20240907000022
              velero.io/restore-name=velero-hayneslab-backups-20240907000022-20240908225225
Annotations:  <none>

Type:  kubernetes.io/rook

Data
====
adminID:   41 bytes
adminKey:  40 bytes
```

Maybe I just use the same one? 

```
thaynes@kubem01:~/workspace/volume-setup$ k -n rook-ceph describe secret rook-csi-rbd-provisioner-default-k8s-disks
Name:         rook-csi-rbd-provisioner-default-k8s-disks
Namespace:    rook-ceph
Labels:       velero.io/backup-name=velero-hayneslab-backups-20240907000022
              velero.io/restore-name=velero-hayneslab-backups-20240907000022-20240908225225
Annotations:  <none>

Type:  kubernetes.io/rook

Data
====
userID:   37 bytes
userKey:  40 bytes
```

Ah, I set up the CRD wrong!!! The driver is still using rbd.

Now for this:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: cephfs-snapshotter
driver: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-cephfs-provisioner-default-k8s-cephfs
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
deletionPolicy: Delete
```

And we are good!

```
thaynes@kubem01:~/workspace/volume-setup$ k -n haynes-intelligence get volumesnapshot
NAME                       READYTOUSE   SOURCEPVC    SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS        SNAPSHOTCONTENT                                    CREATIONTIME   AGE
open-webui-snapshot-test   true         open-webui                           12Gi          cephfs-snapshotter   snapcontent-38fc191b-05df-4e5c-b9af-c01582125e27   4h             5s
```

NOW to VolSync TAKE TWO!

Well, the cluster didn't explode but it just created the volumesnapshot and delete it. 

Not sure what this means:

```
2024-09-09T00:22:11.098-0400    ERROR   controllers.ReplicationSource   unable to delete volume snapshot(s)     {"replicationsource": {"name":"sonarr-config-replica","namespace":"media-management"}, "method": "R
estic", "owned-by": "44082153-1403-494f-a656-67a05bf843b4", "error": "Operation cannot be fulfilled on VolumeSnapshot.snapshot.storage.k8s.io \"volsync-sonarr-config-replica-src\": the ResourceVersion in the pre
condition (14885511) does not match the ResourceVersion in record (14885516). The object might have been modified"}
github.com/backube/volsync/controllers/utils.CleanupObjects
        /workspace/controllers/utils/cleanup.go:65
github.com/backube/volsync/controllers/mover/restic.(*Mover).Cleanup
        /workspace/controllers/mover/restic/mover.go:169
github.com/backube/volsync/controllers.(*rsMachine).Cleanup
        /workspace/controllers/replicationsource_controller.go:362
github.com/backube/volsync/controllers/statemachine.doCleanupState
        /workspace/controllers/statemachine/machine.go:122
github.com/backube/volsync/controllers/statemachine.Run
        /workspace/controllers/statemachine/machine.go:72
github.com/backube/volsync/controllers.(*ReplicationSourceReconciler).Reconcile
        /workspace/controllers/replicationsource_controller.go:149
sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).Reconcile
        /go/pkg/mod/sigs.k8s.io/controller-runtime@v0.17.3/pkg/internal/controller/controller.go:119
sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).reconcileHandler
        /go/pkg/mod/sigs.k8s.io/controller-runtime@v0.17.3/pkg/internal/controller/controller.go:316
sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).processNextWorkItem
        /go/pkg/mod/sigs.k8s.io/controller-runtime@v0.17.3/pkg/internal/controller/controller.go:266
sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).Start.func2.2
        /go/pkg/mod/sigs.k8s.io/controller-runtime@v0.17.3/pkg/internal/controller/controller.go:227
2024-09-09T00:22:11.102-0400    ERROR   Reconciler error        {"controller": "replicationsource", "controllerGroup": "volsync.backube", "controllerKind": "ReplicationSource", "ReplicationSource": {"name":"sona
rr-config-replica","namespace":"media-management"}, "namespace": "media-management", "name": "sonarr-config-replica", "reconcileID": "4c437ed9-658d-4c55-b1e8-695fd19da80d", "error": "Operation cannot be fulfille
d on VolumeSnapshot.snapshot.storage.k8s.io \"volsync-sonarr-config-replica-src\": the ResourceVersion in the precondition (14885511) does not match the ResourceVersion in record (14885516). The object might hav
e been modified"}
```

But it did delete it? Next try I did not get that error but still nothing. 

Maybe this will help:

```yaml
  annotations:
    volsync.backube/privileged-movers: "true"
```

But more this:

```
E0909 00:51:08.319992       1 snapshot_controller_base.go:470] could not sync snapshot "media-management/volsync-sonarr-config-replica-src": snapshot controller failed to update volsync-sonarr-config-replica-src on API server: Operation cannot be fulfilled on volumesnapshots.snapshot.storage.k8s.io "volsync-sonarr-config-replica-src": the object has been modified; please apply your changes to the latest version and try again
```

Found some context on the professionals repo [here](https://github.com/backube/volsync/issues/627#issuecomment-1647985942).


Maybe the path is wrong? 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sonarr-restic-secret
  namespace: media-management
type: Opaque
stringData:
  RESTIC_REPOSITORY: s3:s3-us-east-2.amazonaws.com/s3-volsync/sonarr
  RESTIC_PASSWORD: REDACTED
  AWS_ACCESS_KEY_ID: REDACTED
  AWS_SECRET_ACCESS_KEY: REDACTED
```

Maybe this? `s3:s3.amazonaws.com/s3-volsync/sonarr` - I'll try for Open WebUI:

`open-webui-restic-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: open-webui-restic-secret
  namespace: media-management
type: Opaque
stringData:
  RESTIC_REPOSITORY: s3:s3.amazonaws.com/s3-volsync/s3-volsync/open-webui
  RESTIC_PASSWORD: REDACTED
  AWS_ACCESS_KEY_ID: REDACTED
  AWS_SECRET_ACCESS_KEY: REDACTED
```

```bash
cat open-webui-restic-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-open-webui-restic-secret.yaml
```

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: open-webui-restic-secret
  namespace: haynes-intelligence
spec:
  encryptedData:
    AWS_ACCESS_KEY_ID: REDACTED
    AWS_SECRET_ACCESS_KEY: REDACTED
    RESTIC_PASSWORD: REDACTED
    RESTIC_REPOSITORY: REDACTED
  template:
    metadata:
      creationTimestamp: null
      name: open-webui-restic-secret
      namespace: haynes-intelligence
    type: Opaque
```

`replicationsource.yaml`
```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: open-webui-replica
  namespace: haynes-intelligence
spec:
  sourcePVC: open-webui
  trigger:
    manual: initial-snapshot-09102024
    #schedule: "0 */6 * * *"
  restic:
    copyMethod: Snapshot
    pruneIntervalDays: 10
    repository: open-webui-restic-secret
    cacheCapacity: 15Gi
    storageClassName: cephfs
    volumeSnapshotClassName: cephfs-snapshotter
    retain:
      daily: 10  # Keep most recent from each day for 10 days
      within: 3d # And keep all that were created within 3 days
```

`kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: haynes-intelligence
resources:
  - ./sealedsecret-open-webui-restic-secret.yaml
  - ./sealedsecret-open-webui-provider-credentials.yaml
  - ./sealedsecret-openai-open-webui-api-key.yaml
  - ./helmrelease-open-webui.yaml
  - ./replicationsource.yaml
```

It worked! I think it was the path!

```bash
cat rdt-client-restic-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-rdt-client-restic-secret.yaml
```

```bash
cat radarr-restic-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-radarr-restic-secret.yaml
```

```bash
cat sonarr-restic-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-sonarr-restic-secret.yaml
```