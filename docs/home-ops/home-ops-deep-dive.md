---
title: Home Ops Deep Dive
permalink: /docs/home-ops/deep-dive/
---

Finding other people's flux repos has shown me the way more than any tutorials could. Here's what I've got so far

- onedr0p https://github.com/onedr0p/home-ops
- kashalls https://github.com/kashalls/home-cluster
- bjw-s https://github.com/bjw-s/home-ops 

Or some templates:

- https://github.com/techno-tim/k3s-ansible
- https://github.com/danmanners/aws-argo-cluster-template
- https://github.com/khuedoan/homelab
- https://github.com/ricsanfre/pi-cluster

More to be found from contributors of [this archived goldmine](https://github.com/k8s-at-home).

- angelnu https://github.com/angelnu/k8s-gitops calls out tearing it down with no data loss but seems to use NFS for everything.
- billimek https://github.com/billimek/k8s-gitops has [good readmes](https://github.com/billimek/k8s-gitops/blob/master/kube-system/README.md) every step of the way

## Reloader

Sometimes I update stuff and the pods do not take it. [Reloader](https://github.com/stakater/Reloader) makes em take it. I noticed it when reviewing [these charts](https://github.com/onedr0p/home-ops/blob/main/kubernetes/main/apps/default/sonarr/app/helmrelease.yaml) and figured it was worth a look since I've already been stuck waiting for updates that never come.

### The plan

Maybe the bot we need? I see the bot all over [home-ops](https://github.com/onedr0p/home-ops/commits?author=bot-ross%5Bbot%5D) and I also see [dependabot](https://github.com/external-secrets/external-secrets/commits?author=dependabot%5Bbot%5D) referenced in a lot of repositories. 

The approach seems simple. 

1. We have a bot update our flux repo to keep things up to date. Maybe that is [Renovate](https://github.com/renovatebot/renovate)
1. We use flux or something to roll back after `HelmRelease` update failures so if the update didn't succeed we can come back to it later.
1. We use [reloader](https://github.com/stakater/Reloader) to make sure things like config maps being updated are also consumed, this may not be as important for automatic upgrades but in general will save me some "why isn't this updated" triage time
1. Check out [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller) for upgrading kernel and stuff
1. I have some dashboard, maybe [portainer](https://github.com/portainer/portainer) or [weave](https://github.com/weaveworks/weave-gitops) or even something in Grafana to monitor what isn't being updated and other shit I need to know

## Core Concepts

### Not Afraid of Hardware

Abstracting the hardware to the point where you can just "drop" your cluster repo into any cluster and be up and running is the dream here. This was my main motive for migrating from unRAID to a HA cluster but even this takes it to the next level because, if executed correctly, I won't even be tethered to the HA cluster I have now.

#### Effortless Bootstrapping

I don't think I'm quite where I need to be for effortlessly being able to deploy on any cluster. 

1. [volsync](https://github.com/backube/volsync) to automicatlly restore volumes while velero is more for complete backups.
1. Ansible? for creating VMs and installing CLI stuff I use
1. Talos instead of Debian for easier cluster creation? 

### Observability

There is a lot of work to do in this area, examples [here](https://github.com/bjw-s/home-ops/tree/main/kubernetes/main/apps/monitoring) but probably others in other cluster repos. Need to get a solid foundation I can keep adding to as I host something new.

#### Alert Manager

[Check this out](https://status.devbu.io/)! This is part of `kube-prometheus-stack` under kashalls but everyone is rocking it. 

### Seamless DNS Entries for Internal and External Services

This was a **Just do it** as I urgently wanted UniFi external-dns to kick in for internal ingress classes. The journey was documented [here]({{ site.url }}/docs/home-ops/internal-vs-external-ingress/)

### Repo Organization

> **TODO** 

### Automatic Upgrades

I started to work out bots that support updating the repo [here]({{ site.url }}/docs/home-ops/version-management/).

#### HelmRelease upgrade / rollback

It looks like we can setup HelmRelease with Flux to roll back if something breaks them. It may be as easy as adding this above `values` but I need to dig in more to what exactly this means.

```yaml
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
```

## Project to Consider

### k8tz

[k8tz](https://github.com/k8tz/k8tz) syncs up all the pod time zones!!!!! This sounds FANTASTIC. 

### Secrets 

#### External Secrets

[External Secrets](https://github.com/external-secrets/external-secrets) is being used here over sealed secrets and looks like a lot of stuff, even the API keys for arrs, are in there. Instead of sealing them in-repo this looks to pull them (wait for it) externally. `1Password` seems populate to use in conjunction with this.

#### 1Password

Seems like 1Password is a popular source of `external-secrets`. There is a [connect api](https://github.com/1Password/connect) that is used to link em up, not sure if that is the same as what [this](https://github.com/bjw-s/home-ops/blob/main/kubernetes/main/apps/security/onepassword-connect/app/helmrelease.yaml) is pulling but I don't see a chart. 

### Descheduler

When I kill a node pods get stuck on the dead node. Not sure if this helps but [descheduler](https://github.com/kubernetes-sigs/descheduler) is aimed to get things off nodes so the main scheduler can re-schedule them. Worth a look. 

### VolSync

[VolSync](https://github.com/backube/volsync) seems a bit lighter weight than velero as it only does volumes. Seems to fit in better with flux since you __should__ get everything else when you bootstrap against the cluster repo. I believe VolSync can even pull the volumes at the time of bootstrapping so it'll be needed for that "run on any hardware" level achievement. 

### Spegel 

[Spegel](https://github.com/spegel-org/spegel) is really a nice to have. It's to prevent every node from remotely pulling images and having a centralized repo in-cluster than the nodes can use to speed up the process. At least that is what I gathered form the docs.

### Snapshot Controller

[snapshot-controller](https://artifacthub.io/packages/helm/piraeus-charts/snapshot-controller/1.4.0) sounds like a simple chart that installs some CRDs for volume snapshots that are always missing. I forget what I did this time but I remember one time around I followed instructions from [external-snapshotter](https://github.com/kubernetes-csi/external-snapshotter) to install them.

### System Upgrade Controller

[System Update Controller](https://github.com/rancher/system-upgrade-controller) handles upgrading the system vs. the images and charts running on it. I already found this one so it's likely mentioned elsewhere...

### Hardware Variations Between Nodes

#### Node Feature Discovery

"NFD" automatically labels nodes for what hardware is available on them. This will be great when I start adding GPUs. 

- [Node Feature Discovery GitHub](https://github.com/kubernetes-sigs/node-feature-discovery)
- [Node Feature Discovery Dos](https://kubernetes-sigs.github.io/node-feature-discovery/v0.16/get-started/introduction.html)

#### iGPU Support

The [intel-device-plugins-for-kubernetes](https://github.com/intel/intel-device-plugins-for-kubernetes) must be needed if you are going to use an iGPU in the cluster. Not sure on that yet but worth keeping in mind. 

### Smart Home

### Music Assistant

[Music Assistant](https://github.com/music-assistant/hass-music-assistant) compliances Home Assistant (which we will run as a VM for the full OS).

### Esphome

[Esphome](https://github.com/esphome/esphome) is currently embedded in my HAOS instance and controls a few devices but I can run it standalone like Z2M instead. 

### Databases

I'm not managing database right. Just using what comes in the chart is not going to cut it. 

#### Cloudnative-PG

TODO

#### Postgres Backup

Need a way to do DB backups independently of volume backups

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: database
spec:
  schedule: "0 4 * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          automountServiceAccountToken: false
          enableServiceLinks: false
          securityContext:
            runAsUser: 1031
            runAsGroup: 1031
            fsGroup: 1031
            supplementalGroups:
              - 65541

          containers:
            - name: zalando-postgres-backup
              image: docker.io/prodrigestivill/postgres-backup-local:15@sha256:7f12039af361b71c987ad06b5c0a9dca67bad92f10bffa5012a614311363eebb
              imagePullPolicy: IfNotPresent
              command:
                - "/backup.sh"
              env:
                - name: POSTGRES_EXTRA_OPTS
                  value: "-n public -Z6"
                - name: POSTGRES_HOST
                  value: postgres.database.svc.cluster.local
                - name: POSTGRES_USER
                  valueFrom:
                    secretKeyRef:
                      name: postgres.postgres.credentials.postgresql.acid.zalan.do
                      key: username
                - name: POSTGRES_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres.postgres.credentials.postgresql.acid.zalan.do
                      key: password
                - name: POSTGRES_DB
                  value: "dsmr-reader,home-assistant,immich,miniflux"
              volumeMounts:
                - name: nas-backups
                  mountPath: /backups

          restartPolicy: OnFailure

          volumes:
            - name: nas-backups
              nfs:
                server: "nas.bjw-s.casa"
                path: /volume1/Backup/Databases/postgresql
```

#### Redis / Dragonfly

[Dragonfly](https://www.dragonflydb.io/) makes some bold claims about beating out redis. Seems like people host one instance and use it for things, may be worth a look. 

## Hardware Dependency 

Check out what this guy says [here](https://github.com/billimek/k8s-gitops/blob/master/kube-system/README.md#descheduler) - seems like we may be able do deal with usb devices for stuff like the NUT servers if we use this feature discovery thing! 

## Namespace Alerts

See this in every namespace (in onedr0p)!

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: default
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
    volsync.backube/privileged-movers: "true"
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/notification.toolkit.fluxcd.io/provider_v1beta3.json
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: alert-manager
  namespace: default
spec:
  type: alertmanager
  address: http://alertmanager-operated.observability.svc.cluster.local:9093/api/v2/alerts/
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/notification.toolkit.fluxcd.io/alert_v1beta3.json
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: alert-manager
  namespace: default
spec:
  providerRef:
    name: alert-manager
  eventSeverity: error
  eventSources:
    - kind: HelmRelease
      name: "*"
  exclusionList:
    - "error.*lookup github\\.com"
    - "error.*lookup raw\\.githubusercontent\\.com"
    - "dial.*tcp.*timeout"
    - "waiting.*socket"
  suspend: false
```

## WTF Is SOPS

[Sops](https://github.com/getsops/sops) is about secrets. That is all I know. Why does someone need sops? 

OK it let's you push shit like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: github-deploy-key
    namespace: flux-system
stringData:
    identity: HUGE
    known_hosts: HUGE
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age:
        - recipient: age15uzrw396e67z9wdzsxzdk7ka0g2gr3l460e0slaea563zll3hdfqwqxdta
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            HUGE
            -----END AGE ENCRYPTED FILE-----
    lastmodified: "2024-06-28T22:37:16Z"
    mac: HUGE
    pgp: []
    encrypted_regex: ^(data|stringData)$
    mac_only_encrypted: true
    version: 3.9.0
```

And then you can bootstrap when you don't have an `external-secrets` pod running.

## Docs

https://onedr0p.github.io/home-ops/

These are the docs in the home-ops repo. I guess you can use a folder as a symbol too, that is fancy: 📁

Cool up time via https://uptimerobot.com/

Email via https://migadu.com/

These are SOLID manuals -> https://www.sphinx-doc.org/en/master/ themed with https://github.com/readthedocs/sphinx_rtd_theme from https://about.readthedocs.com/?ref=readthedocs.org

## Gatus

[Gatus](https://github.com/TwiN/gatus) is a very nice health dashboard that is tied into everything - setting this up looks like it'll teach me a lot too as there are templates, postgres, fancy YAML all involved. 

## EMQX

[EMQX](https://www.emqx.com/en) seems to replace mosquitto and has a dashboard! May be worthy. It [works with HomeAssistant](https://docs.emqx.com/en/cloud/latest/connect_to_deployments/home_assistant.html) too.

## R2

I bought some [R2 Storage](https://dash.cloudflare.com/1adbb78981186f1bd409cc11913b459a/r2/overview) for free though. Seems better than what I am using over at AWS.

## Local Storage

* [OpenEBS](https://github.com/openebs/openebs)
* Minio 

`cacheStorageClassName: "${VOLSYNC_CACHE_SNAPSHOTCLASS:-openebs-hostpath}"` puts the VolSync cache in openebs instead of ceph.

It is using OpenEBS to create a single node volume that holds the cache.

## Keycloak GitOps

Someone said to use [Flux with Terraform](https://fluxcd.io/blog/2022/09/how-to-gitops-your-terraform/) to configure KeyCloak so it was GitOps - "I deploy it with the helm chart and configure it via Terraform with the tf-controller for Flux". See [his repo](https://github.com/mirceanton/home-ops/)

