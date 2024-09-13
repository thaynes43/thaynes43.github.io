---
title: Going Public
permalink: /docs/home-ops/going-public/
---

This is currently to keep track of what I need to do to make the repo public or make a public mirror.

* Vikunja has an exposed secret
* Needs better organization
* Needs to be boostrapable, hardware agnostic, volsync for deploying w/ state on new clusters
* Need to backup databases independently of volumes with something like `postgres-backup`
* Needs instructions on how to use the cluster and bootstrap repos

May want to look into NFS vs. SMB as these repos use NFS. AI says:

```
Summary
NFS is generally preferred in Unix/Linux environments due to its seamless integration with Unix permissions and straightforward setup process. It’s often chosen for high-performance needs and simplicity.
SMB is typically favored in Windows environments because of its compatibility with Windows security models, ACLs, and additional features like Active Directory integration.
Ultimately, the choice between NFS and SMB depends on the specific requirements of your environment and the operating systems of the clients and servers. In mixed environments, both protocols might be used simultaneously to leverage their respective advantages.
```

After this the volume was stuck in a state where it could not be bound to:

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                                STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-smb-tank-k8s-download-clients           100Ti      RWX            Retain           Bound      download-clients/pvc-smb-tank-k8s-download-clients   smb-tank-k8s   <unset>                          41h
pv-smb-tank-k8s-media-management           100Ti      RWX            Retain           Released   media-management/pvc-smb-tank-k8s-media-management   smb-tank-k8s   <unset>                          41h
```

I found this was because a `claimRef` from before was still there. Easy fix:

```bash
kubectl patch pv pv-smb-tank-k8s-media-management -p '{"spec":{"claimRef": null}}'
```