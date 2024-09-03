---
title: CSI Driver SMB
permalink: /docs/csi-driver-smb/
---

In order to get to my media for __media management__ I will need to mount the share. To do this I should be able to use [csi-driver-smb](https://github.com/kubernetes-csi/csi-driver-smb). I found a tutorial [here](https://rguske.github.io/post/using-windows-smb-shares-in-kubernetes/) which can provide some reference material but gtp4o seems pretty confident...

## GTP4o Instructions

Will use official docs to install `csi-driver-smb` but after this is the gist of how to mount the volume:

> **WARNING** we do NOT want velero moving this to S3! 

### PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: smb-pv
spec:
  capacity:
    storage: 10Gi  # Informational; does not create storage
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: smb.csi.k8s.io
    readOnly: false
    volumeHandle: smb-pv
    volumeAttributes:
      source: "//smb-server/share"  # Your existing SMB share
    nodeStageSecretRef:
      name: smb-secret
      namespace: default
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: smb-secret
  namespace: default
stringData:
  username: smb-username
  password: smb-password
```

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: smb-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi  # Informational; the actual capacity is governed by the SMB server
  storageClassName: ""
  volumeName: smb-pv
```

### Usage

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: smb-test-pod
spec:
  containers:
    - name: smb-container
      image: nginx
      volumeMounts:
        - name: smb-volume
          mountPath: /mnt/smb
  volumes:
    - name: smb-volume
      persistentVolumeClaim:
        claimName: smb-pvc
```

## First, a Test Share

I will need to re-learn the steps I followed [here]({{ site.url }}/hardware/haynesintelligence#smb-shares-with-cockpit) and create a test share for k8s to bang around on.

### Adding to Tank

> **Note** I have already set up the nas_shares account and permissions from [this-guide](https://blog.kye.dev/proxmox-zfs-mounts)

On `HaynesIntelligence`:

```
zfs create tank/k8s_root
chown -R nas:nas_shares /tank/k8s_root/
pct set 121 -mp1 /tank/k8s_root,mp=/mnt/k8s_root
```

I thought I would have to reboot nas01 (LXC 121) to see the share but it's just there, like magic!

### Exposing via SMB

First I am going to create a new user for this share so I don't need to use my personal account in secrets:

* User: k8suser
* Password: REDACTED

It said the password was weak because it couldn't check it, so whatever, it's not weak. Now add to nas_shares group. Then I created the share by clicking the + near my first share and using the same settings. 

Finally last thing was to get permissions right this time: 

![cockpit-share-permissions]({{ site.url }}/images/web/cockpit-share-permissions.png)

And boom, I can read/write from it in Windows. 

## Mounting The New Share in Kubernetes

Now to see if we can use it without destroying it. 

### Flux Files

We need to deploy [this chart](https://github.com/kubernetes-csi/csi-driver-smb/tree/master/charts) now via flux.

#### Namespace

The namespace for this driver will be `kube-system`

#### HelmRepository

`helmrepository-csi-driver-smb.yaml`
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: csi-driver-smb
  namespace: flux-system
spec:
  interval: 15m
  url: https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts
```

#### Kustomization

I am going to create one file for both the volume CRDs and the `cis-driver`.

`kustomization-smb.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: csi-driver-smb
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
  path: ./smb/driver
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: csi-driver-smb
      namespace: kube-system
```

#### HelmRelease

I am going with the defaults. The example is very specific about a `v` in the version name so I'll try that first and leave out wildcards.

`helmrelease-csi-driver-smb.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: csi-driver-smb
  namespace: kube-system
spec:
  chart:
    spec:
      chart: csi-driver-smb
      version: "v1.15.0" # https://github.com/kubernetes-csi/csi-driver-smb/tree/master/charts
      sourceRef:
        kind: HelmRepository
        name: csi-driver-smb
        namespace: flux-system
  interval: 15m
  timeout: 5m
  releaseName: csi-driver-smb
  values: values.yaml
```

## Test It!

This [usage](https://github.com/kubernetes-csi/csi-driver-smb/blob/master/deploy/example/e2e_usage.md) guide seems pretty straight forward. After that I just need to use the existing claim in my services accessing the data.

### Secret

Temporarily store it in a file and verify it's what we want:

```bash
kubectl create secret generic smbcreds-tank-k8s \
--namespace default \
--dry-run=client \
--from-literal username="USER" \
--from-literal password="PASSWORD" \
-o yaml > smbcreds-tank-k8s.yaml
```

Seal it up:

```bash
cat smbcreds-tank-k8s.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem --format yaml > sealedsecret-smbcreds-tank-k8.yaml
```

### PersistentVolume

I'm not really sure what the 100Gi does here since the share could have more or less storage.

`volume-tank-k8s.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: smb.csi.k8s.io
  name: pv-smb-tank-k8s
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: smb
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - uid=1000
    - gid=1000
  csi:
    driver: smb.csi.k8s.io
    # volumeHandle format: {smb-server-address}#{sub-dir-name}#{share-name}
    # make sure this value is unique for every share in the cluster
    volumeHandle: nas01#k8s
    volumeAttributes:
      source: //nas01/k8s
    nodeStageSecretRef:
      name: smbcreds-tank-k8s
      namespace: default
```

### PersistentVolumeClaim

Supposedly the 10Gi doesn't matter here.

`claim-media-management-tank-k8s.yaml`
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-smb-tank-k8s
  namespace: media-management
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: smb
```

## Test It!

> **WARING** A bunch of iterating was done on the CRDs above. They landed fairly close and those should work but I threw the final product in down below.

Everything bound successfully:

```bash
thaynes@kubem01:~/workspace/secret-sealing$ k get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                         STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-smb-tank-k8s                            100Gi      RWX            Retain           Bound    media-management/pvc-smb-tank-k8s                             smb            <unset>                          62s

thaynes@kubem01:~/workspace/secret-sealing$ k get pvc -n media-management 
NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
pvc-smb-tank-k8s   Bound    pv-smb-tank-k8s                            100Gi      RWX            smb            <unset>                 100s
```

Well, it got weird with permissions so I ended up going with the original account that I had been using. The new account couldn't write in folders or files the new account created and vice versa. Was a nightmare. 

After a long round of testing I landed with two pv's and two pvc's, one per namespace where I'd need the data. Here is what one looked like, the other was identical except for the name:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: smb-tank-k8s
provisioner: smb.csi.k8s.io
parameters:
  source: //nas01/k8s
  # if csi.storage.k8s.io/provisioner-secret is provided, will create a sub directory
  # with PV name under source
  csi.storage.k8s.io/provisioner-secret-name: smbcreds-tank-k8s
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: smbcreds-tank-k8s
  csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Retain  # available values: Delete, Retain
volumeBindingMode: Immediate
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000
---
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: smb.csi.k8s.io
  name: pv-smb-tank-k8s-download-clients
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: smb-tank-k8s
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
  csi:
    driver: smb.csi.k8s.io
    # volumeHandle format: {smb-server-address}#{sub-dir-name}#{share-name}
    # make sure this value is unique for every share in the cluster
    volumeHandle: nas01#k8s
    volumeAttributes:
      source: //nas01/k8s
    nodeStageSecretRef:
      name: smbcreds-tank-k8s
      namespace: default
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-smb-tank-k8s-download-clients
  namespace: download-clients
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeName: pv-smb-tank-k8s-download-clients
  storageClassName: smb-tank-k8s
```

Then anywhere I needed the data already on the share I'd just use this as the `existingClaim` and it would mount at the path I provided:

```yaml
      tank:
        type: persistentVolumeClaim
        existingClaim: pvc-smb-tank-k8s-download-clients
        globalMounts:
          - path: /tank
            readOnly: false
```

> **NOTE** I don't know if the StorageClass comes into play with the existing volumes but it can be used to create new volumes in the future. These would be easier to edit since windows can map directly to them for big configs like Kometa.

Though annoying I had to copy paste the same pv and pvc twice to get it across the namespace it does seem like this doesn't have to be the case if I use [alpha features](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#cross-namespace-data-sources) but for now I'll duplicate things.

## HaynesTower (unRAID) Test

### SealedSecrets w/ SMB Credentials

Once again we need the credentials

```bash
kubectl create secret generic smbcreds-haynestower-k8s \
--namespace default \
--dry-run=client \
--from-literal username="USERNAME" \
--from-literal password="PASSWORD" \
-o yaml > smbcreds-haynestower-k8s.yaml
```

Seal it up:

```bash
cat smbcreds-haynestower-k8s.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem --format yaml > sealedsecret-smbcreds-haynestower-k8.yaml
```

Copy paste in:

`sealedsecret-smbcreds-haynestower-k8.yaml`
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: smbcreds-haynestower-k8s
  namespace: default
spec:
  encryptedData:
    password: SEALEDREDACTED
    username: SEALEDREDACTED
  template:
    metadata:
      creationTimestamp: null
      name: smbcreds-haynestower-k8s
      namespace: default
```

### StorageClass

Next I'll create a storage class if we want to provision volumes on the fly. Not sure we need this for the static volume we'll be testing though.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: smb-haynestower-k8s
provisioner: smb.csi.k8s.io
parameters:
  source: //HAYNESTOWER/k8s
  # if csi.storage.k8s.io/provisioner-secret is provided, will create a sub directory
  # with PV name under source
  csi.storage.k8s.io/provisioner-secret-name: smbcreds-haynestower-k8s
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: smbcreds-haynestower-k8s
  csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Retain  # available values: Delete, Retain
volumeBindingMode: Immediate
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000
```

### Static Volume Mounting to More Test Media

Before we really "go live" I'm setting up some test media on HaynesTower to make sure everything plays nicely and we don't throw it all up to S3 again.