---
title: Backup Strategy for Everything (so far) 
permalink: /docs/backup-strat/
---

unRAID is currently backing up on it's own array using the [Appdata.Backup](https://forums.unraid.net/topic/137710-plugin-appdatabackup/) plugin but once the containers are migrated we need a better strategy for at least backing up the Plex db. 

For this we've already followed [these steps](https://forum.proxmox.com/threads/how-to-setup-pbs-as-a-vm-in-unraid-and-uses-virtiofs-to-passthrough-shares.120271/) to run a [Proxmox Backup Server](https://www.proxmox.com/en/proxmox-backup-server/overview) VM on unRAID. The only pitfall here was selecting the wrong network adapter and not knowing how to fix it so I just re-did the thing and we're good.

The cluster is configured for various backups which we should review and update later.

Kubernetes was set up to create volumesnapshots using [rook ceph documentation](https://rook.io/docs/rook/latest/Storage-Configuration/Ceph-CSI/ceph-csi-snapshot/). At first [Gemini](https://github.com/FairwindsOps/gemini) looked like a solid option but I don't see a way to migrate the backups off the ceph cluster with it.

May be able to do something if we look like S3 with S3 [seaweedfs](https://github.com/seaweedfs/seaweedfs) or [seafile](https://github.com/haiwen/seafile) though this might be a bad approach. 

Next we need to see if we want [Velero](https://velero.io/docs/v1.14/) or [CloudCasa](https://docs.cloudcasa.io/help/index.html) but if we go Velero [this guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/backup/velero/) looks pretty good.

It may also be worth exploring [cephgeorep](https://github.com/45Drives/cephgeorep) or [backy2](https://backy2.com/docs/index.html) for generic ceph backups.

