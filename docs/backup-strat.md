---
title: Backup Strategy for Everything (so far) 
permalink: /docs/backup-strat/
---

## unRAID Dockers

unRAID is currently backing up on it's own array using the [Appdata.Backup](https://forums.unraid.net/topic/137710-plugin-appdatabackup/) plugin but once the containers are migrated we need a better strategy for at least backing up the Plex db. 

## Proxmox Backup Server

For this we've already followed [these steps](https://forum.proxmox.com/threads/how-to-setup-pbs-as-a-vm-in-unraid-and-uses-virtiofs-to-passthrough-shares.120271/) to run a [Proxmox Backup Server](https://www.proxmox.com/en/proxmox-backup-server/overview) VM on unRAID. The only pitfall here was selecting the wrong network adapter and not knowing how to fix it so I just re-did the thing and we're good.

The cluster is configured for various backups which we should review and update later.

## Kubernetes

Kubernetes was set up to create volumesnapshots using [rook ceph documentation](https://rook.io/docs/rook/latest/Storage-Configuration/Ceph-CSI/ceph-csi-snapshot/). At first [Gemini](https://github.com/FairwindsOps/gemini) looked like a solid option but I don't see a way to migrate the backups off the ceph cluster with it.

May be able to do something if we look like S3 with S3 [seaweedfs](https://github.com/seaweedfs/seaweedfs) or [seafile](https://github.com/haiwen/seafile) though this might be a bad approach. 

Next we need to see if we want [Velero](https://velero.io/docs/v1.14/) or [CloudCasa](https://docs.cloudcasa.io/help/index.html) but if we go Velero [this guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/backup/velero/) looks pretty good.

It may also be worth exploring [cephgeorep](https://github.com/45Drives/cephgeorep) or [backy2](https://backy2.com/docs/index.html) for generic ceph backups.

### Velero 

Before we can follow this guide we need to setup something called flux which will benefit everything we do going further. W

#### Flux

##### Install Flux

```
curl -s https://fluxcd.io/install.sh | sudo bash
```

Then make a token (REDACTED).

##### Bootstrap & Run Flux

GITHUB_TOKEN=REDACTED \
flux bootstrap github \
  --owner=thaynes43 \
  --repository=flux-repo \
  --personal \
  --path bootstrap

A new namespace was added with the following:

```
flux-system            helm-controller-76dff45854-ghhzd                        1/1     Running   0               5m31s
flux-system            kustomize-controller-6bc5d5b96-wzjxv                    1/1     Running   0               5m31s
flux-system            notification-controller-7f5cd7fdb8-z49tr                1/1     Running   0               5m31s
flux-system            source-controller-54c89dcbf6-q4wbf                      1/1     Running   0               5m31s
```

The example also added `podinfo` which tests stuff or something, we'll find out later.

Now for [learning](https://geek-cookbook.funkypenguin.co.nz/kubernetes/deployment/flux/design/) and [operating](https://geek-cookbook.funkypenguin.co.nz/kubernetes/deployment/flux/operate/)...

To see flux update:

```
watch -n1 flux get kustomizations
```

To force an update:

```
flux reconcile source git flux-system
flux reconcile helmrelease -n <namespace> <name of helmrelease>
```

See it:

```
flux get kustomizations
kubectl get helmreleases -n podinfo
```

### Now for Velero

Now that the flux stuff is out of the way we can follow [this guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/backup/velero/) in getting Velero up and running. We are also going to need off-cluster support for PVC snapshots which is listed as optional here.

#### S3 Secret

Right away I got hit with creating some S3 stuff so I figured I'd just go for it. Not really sure what I'm doing but I made a bucket and a key and will see what it does...

##### Roadblock #1

First I need to learn how to setup [sealed secrets](https://geek-cookbook.funkypenguin.co.nz/kubernetes/sealed-secrets)

The instructions for installing kubeseal were wrong but this worked:

```
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/kubeseal-0.27.1-linux-amd64.tar.gz
tar -xvf kubeseal-0.27.1-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

Rest of the instructions worked but I didn't get into `Using our own keypair` for when the pod with the secrets go down, will need to look that over more when it's less late.

##### Back to S3

I can now create and store the sealed secret in a file though I can't kill the pod or I won't know how to get it back.

```
kubectl create secret generic -n velero velero-credentials \
  --from-file=cloud=velero/awssecret \
  -o yaml --dry-run=client \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets > velero/sealedsecret-velero-credentials.yaml
```

That worked, but now the helm chart values pattern has shifted from the earlier examples...

![dude changed his mind]({{ site.url }}/images/web/dude-changed-his-mind.png)

Adding the values to the helm file worked fine. I only configured S3 and the logs are promising:

```
thaynes@kubevip:~/workspace/rook-setup/snapshots$ kubectl logs -n velero -l app.kubernetes.io/name=velero
Defaulted container "velero" out of: velero, velero-plugin-for-aws (init)
time="2024-08-04T02:52:22Z" level=info msg="BackupStorageLocations is valid, marking as available" backup-storage-location=velero/default controller=backup-storage-location logSource="pkg/controller/backup_storage_location_controller.go:126"
time="2024-08-04T02:52:22Z" level=info msg="plugin process exited" backup-storage-location=velero/default cmd=/plugins/velero-plugin-for-aws controller=backup-storage-location id=802 logSource="pkg/plugin/clientmgmt/process/logrus_adapter.go:80" plugin=/plugins/velero-plugin-for-aws
time="2024-08-04T02:53:22Z" level=info msg="plugin process exited" backupLocation=velero/default cmd=/plugins/velero-plugin-for-aws controller=backup-sync id=815 logSource="pkg/plugin/clientmgmt/process/logrus_adapter.go:80" plugin=/plugins/velero-plugin-for-aws
time="2024-08-04T02:53:22Z" level=info msg="Validating BackupStorageLocation" backup-storage-location=velero/default controller=backup-storage-location logSource="pkg/controller/backup_storage_location_controller.go:141"
time="2024-08-04T02:53:22Z" level=info msg="BackupStorageLocations is valid, marking as available" backup-storage-location=velero/default controller=backup-storage-location logSource="pkg/controller/backup_storage_location_controller.go:126"
```

Next I need [velero CSI](https://velero.io/docs/v1.12/basic-install/#install-the-cli).

```
wget https://github.com/vmware-tanzu/velero/releases/download/v1.14.0/velero-v1.14.0-linux-amd64.tar.gz
tar -xvf velero-v1.14.0-linux-amd64.tar.gz
sudo install -m 755 velero /usr/local/bin/velero
```

Now for the backup:

```
velero backup create hellothere --include-namespaces=default --wait
```

CLI looks good!

```
Backup request "hellothere" submitted successfully.
Waiting for backup to complete. You may safely press ctrl-c to stop waiting - your backup will continue in the background.
.
Backup completed with status: Completed. You may check for more information using the commands `velero backup describe hellothere` and `velero backup logs hellothere`.
```

![s3 first backup]({{ site.url }}/images/web/s3-first-backup.png)

##### Debug Automated Backup

Few issues checking if the automated backup worked this AM.

1. The backup did not happen
1. The logs are in the wrong timezone and gone for when the backup should have happen
1. The pod restarted right when the backup would have been triggered given the timezone the pod is logging



##### Restore from S3