
---
title: VLAN Migration - Migrate Flux
permalink: /docs/vlan-migration/flux-migration/
---

## Migrating What I have

Since I'm one step away from a `pv` with flux I can move on that before circling back to `rook-ceph` external.

At this point I have a cluster and a VM to access it with what I believe to be all the dependencies needed to run what is in flux EXCEPT velero since I can't take snapshots of anything.

### Changes to Values Before We Bootstrap

I can anticipate a few things that are wrong with my current files but we will find way more once we bootstrap the cluster with the repo.

- MetalLB needs to now use the range `192.168.40.100-192.168.40.254` since I've got hosts and VMs below 100
- Traefik can now use `192.168.40.100` as it's endpoint
- DNS entries for reverse proxy local endpoints need to hit this now
- Sealed secrets should break, I was told, but we can't re-seal anything yet

### Bootstrapping Time! 

I am going to move out a bunch of `Kustomization` files so I can do a slow roll out and get decencies working first. I didn't know how to disable them so I just copy pasted the files. I will start with:

1. MetalLB
2. Sealed-Secrets
3. PodInfo

```
GITHUB_TOKEN=REDACTED \
flux bootstrap github \
  --owner=REDACTED \
  --repository=flux-repo \
  --personal \
  --path bootstrap
```

### Changes After Bootstrapping

Everything came up fine so it's time to get it all back in action!

#### Fixing Sealed Secrets

I don't expect any Sealed Secrets to work because I did not use my own keypair as it was at the bottom of [this guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/sealed-secrets/) and they were set up when I got there. Now I can [generate one](https://github.com/bitnami-labs/sealed-secrets/blob/main/docs/bring-your-own-certificates.md) and use it. 

Will need this for the following commands:

```bash
kubeseal --fetch-cert \
--controller-name=sealed-secrets \
--controller-namespace=sealed-secrets \
> pub-cert.pem
```

> **TODO** Need to include the part about using the custom keypair in all the commands below. These were the ones I ran without it.

#### Fixing Traefik's Secret

First `nano traefik-dash-secret.yaml` and add:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: traefik-dashboard-auth
  namespace: traefik
type: Opaque
data:
  users: REDACTED
```

Then seal and copy paste to git repo:

```bash
cat traefik-dash-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-traefik-dashboard-auth.yaml
```

> **WARNING** I didn't end up using this but keeping here for reference as CF secrets are useful

First `nano traefik-cf-secret.yaml` and add:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-credentials
  namespace: traefik

type: Opaque
stringData:
  email: test@gmail.com
  apiKey: REDACTED
```

Then seal and copy paste to git repo:

```bash
cat traefik-cf-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-cloudflare-api-credentials.yaml
```

#### Velero Sealed Secret

First `nano awssecret` and add:

```
[default]
aws_access_key_id = REDACTED
aws_secret_access_key = REDACTED
```

Then seal and copy paste to git repo:

```bash
kubectl create secret generic -n velero velero-credentials \
  --from-file=cloud=awssecret \
  -o yaml --dry-run=client \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --format yaml > sealedsecret-velero-credentials.yaml
```

#### External-DNS Secret

This one has no file to pass as it's one key:

```bash
  kubectl create secret generic cloudflare-api-token \
  --namespace external-dns \
  --dry-run=client \
  --from-literal=cloudflare_api_token=REDACTED -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem  \
  --format yaml \
  > sealedsecret-cloudflare-api-token.yaml
```

#### Cert Manager Secret

Basically the same as external-dns except for the name:

```bash
  kubectl create secret generic cloudflare-api-token-secret \
  --namespace cert-manager \
  --dry-run=client \
  --from-literal=api-token=REDACTED -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml \  
  > sealedsecret-cloudflare-api-token-secret.yaml
```

### Few Tweaks

CRDs were brutal to get installed because I had already defined many. Helm charts automatically install these but the charts failed pre-flight checks and were not ran until I moved the things using the new CRDs out, ran just the services, then moved them back in. Everything was healthy after that.

I also managed to get `external-dns` working for a new entry by deleting everything in cloudflare from before and creating one `DNSEndpoint` CRD for tautulli. I tweaked the values too so I could use resources across namespaces:

```yaml
    providers:
      kubernetesCRD:
        enabled: true
        ingressClass: traefik-external
        allowExternalNameServices: true
        allowCrossNamespace: true
```

Now I can clean things up a bit. `HaynesTower` service is in namespace `arr-stack` but it should be centralized. So can `Middleware` like default-headers since that is used a lot. I think I justs need to reflect secrets and the rest can be used across namespaces. So, some tech debt:

1. Move the haynestower Service to a common place
1. Move default-headers middleware to a common place
1. Centralize IngressRoutes that just proxy haynestower port/IPs
1. Decide if DNSEndpoints should live with the services or in a central spot
1. Clean up dependencies. Right now `monitoring` (for tautulli IngresssRoute) should really depend on `arr-stack` since that is where haynestower comes out of but that is silly
1. Add IngressRoute for and DNS Entry plex backdoor to test this all out

I am writing that down to solidify it's place as tech debt as I do not feel like paying it right now.

### GitOps Ceph Stuff

On to the hard part.

#### Rook Ceph

[Rook External](https://rook.io/docs/rook/v1.12/CRDs/Cluster/external-cluster/) was not easy to set up the first go at it. Funky man also has a [guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/persistence/rook-ceph/) for GitOps'in the full blown deployment so between those and my notes, plus the files on the previous cluster, I might have a chance.

I think I can forgo the funkyman guide, the external guide can be funkified with my level of funk.

First we go to PVE, no need to install ceph on the kubernetes node.

```bash
python3 create-external-cluster-resources.py --cephfs-filesystem-name k8s-cephfs --rbd-data-pool-name k8s-disks --restricted-auth-permission false --format bash --namespace rook-ceph --ceph-conf /etc/ceph/ceph.conf --keyring /etc/pve/priv/ceph.client.admin.keyring --k8s-cluster-name default
```

We get some good stuff:

```
export NAMESPACE=rook-ceph
export ROOK_EXTERNAL_FSID=9b0628e1-1fe9-49d2-b65b-746d05215e3d
export ROOK_EXTERNAL_USERNAME=client.healthchecker
export ROOK_EXTERNAL_CEPH_MON_DATA=pve01=192.168.40.6:6789
export ROOK_EXTERNAL_USER_SECRET=REDACTED
export CSI_RBD_NODE_SECRET=REDACTED
export CSI_RBD_NODE_SECRET_NAME=csi-rbd-node-default-k8s-disks
export CSI_RBD_PROVISIONER_SECRET=REDACTED
export CSI_RBD_PROVISIONER_SECRET_NAME=csi-rbd-provisioner-default-k8s-disks
export CEPHFS_POOL_NAME=k8s-cephfs_data
export CEPHFS_METADATA_POOL_NAME=k8s-cephfs_metadata
export CEPHFS_FS_NAME=k8s-cephfs
export RESTRICTED_AUTH_PERMISSION=false
export CSI_CEPHFS_NODE_SECRET=REDACTED
export CSI_CEPHFS_PROVISIONER_SECRET=REDACTED
export CSI_CEPHFS_NODE_SECRET_NAME=csi-cephfs-node-default-k8s-cephfs
export CSI_CEPHFS_PROVISIONER_SECRET_NAME=csi-cephfs-provisioner-default-k8s-cephfs
export MONITORING_ENDPOINT=192.168.40.6
export MONITORING_ENDPOINT_PORT=9283
export RBD_POOL_NAME=k8s-disks
export RGW_POOL_PREFIX=default
```

Now I just set all that in the shell and run some other script.

```
thaynes@kubem01:~/workspace$ ./import-external-cluster.sh 
namespace/rook-ceph created
secret/rook-ceph-mon created
configmap/rook-ceph-mon-endpoints created
secret/rook-csi-rbd-node-default-k8s-disks created
secret/rook-csi-rbd-provisioner-default-k8s-disks created
secret/rook-csi-cephfs-node-default-k8s-cephfs created
secret/rook-csi-cephfs-provisioner-default-k8s-cephfs created
storageclass.storage.k8s.io/ceph-rbd created
storageclass.storage.k8s.io/cephfs created
```

Now we need to take the helm instructions to funky town:

```yaml
    clusterNamespace=rook-ceph
    operatorNamespace=rook-ceph
    cd deploy/examples/charts/rook-ceph-cluster
    helm repo add rook-release https://charts.rook.io/release
    helm install --create-namespace --namespace $clusterNamespace rook-ceph rook-release/rook-ceph -f values.yaml
    helm install --create-namespace --namespace $clusterNamespace rook-ceph-cluster \
    --set operatorNamespace=$operatorNamespace rook-release/rook-ceph-cluster -f values-external.yaml
```

After making some of the funkiest files I landed myself a first try gold rush:

```
thaynes@kubem01:~/workspace$ kubectl -n rook-ceph get CephCluster
NAME        DATADIRHOSTPATH   MONCOUNT   AGE     PHASE       MESSAGE                          HEALTH      EXTERNAL   FSID
rook-ceph   /var/lib/rook     3          2m36s   Connected   Cluster connected successfully   HEALTH_OK   true       9b0628e1-1fe9-49d2-b65b-746d05215e3d
```

And for both `ceph-rbd` and `cephfs`! Hopefully it works, I have not added back and stacks that have persistance yet.

```
thaynes@kubem01:~/workspace$ kubectl -n rook-ceph get sc
NAME                   PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ceph-rbd               rook-ceph.rbd.csi.ceph.com      Delete          Immediate              true                   37m
cephfs                 rook-ceph.cephfs.csi.ceph.com   Delete          Immediate              true                   37m
local-path (default)   rancher.io/local-path           Delete          WaitForFirstConsumer   false                  8h
```

#### Snapshots

Before we can put velero back we need to get this resource going again:

```
kubectl api-resources | grep volumesnapshots
```

Nothing returned right now! I followed [rook documentation](https://rook.io/docs/rook/latest/Storage-Configuration/Ceph-CSI/ceph-csi-snapshot/) before and will do so again.

I think I just need to run:

```
kubectl kustomize client/config/crd | kubectl create -f -
```

Which will point here: https://github.com/kubernetes-csi/external-snapshotter/tree/master/client/config/crd

```
thaynes@kubem01:~/workspace/volume-setup$ cd external-snapshotter/
thaynes@kubem01:~/workspace/volume-setup/external-snapshotter$ kubectl kustomize client/config/crd | kubectl create -f -
customresourcedefinition.apiextensions.k8s.io/volumesnapshotclasses.snapshot.storage.k8s.io created
customresourcedefinition.apiextensions.k8s.io/volumesnapshotcontents.snapshot.storage.k8s.io created
customresourcedefinition.apiextensions.k8s.io/volumesnapshots.snapshot.storage.k8s.io created
```

And for the grand finale:

```
thaynes@kubem01:~/workspace/volume-setup/external-snapshotter$ kubectl api-resources | grep volumesnapshots
volumesnapshots                     vs                  snapshot.storage.k8s.io/v1               true         VolumeSnapshot
```

And then it just says to rock:

```
kubectl -n kube-system kustomize deploy/kubernetes/snapshot-controller | kubectl create -f -
```

Which looks good:

```
thaynes@kubem01:~/workspace/volume-setup/external-snapshotter$ kubectl -n kube-system kustomize deploy/kubernetes/snapshot-controller | kubectl create -f -
serviceaccount/snapshot-controller created
role.rbac.authorization.k8s.io/snapshot-controller-leaderelection created
clusterrole.rbac.authorization.k8s.io/snapshot-controller-runner created
rolebinding.rbac.authorization.k8s.io/snapshot-controller-leaderelection created
clusterrolebinding.rbac.authorization.k8s.io/snapshot-controller-role created
deployment.apps/snapshot-controller created
```

We won't mention that we didn't GitOps this, this should really be part of "building" the cluster that we deploy to since it's a one and done (hopefully).

### Velero

Hopefully we finish the day on a strong foot by adding `velero` in and having it just work for the default namespace. Nothing there has a volume now so we won't be testing the snapshotter yet.

I am going to add a dependency to it for ceph just because that can't be faulty for it to really work:

``` yaml
  dependsOn:
    - name: sealed-secrets
    - name: rook-ceph  
```

Everything else went smooth. Testing to come later since I have no volumes yet.
