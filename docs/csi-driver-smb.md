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