---
title: Home Ops Deep Dive
permalink: /docs/home-ops/deep-dive/
---

Finding other people's flux repos has shown me the way more than any tutorials could. Here's what I've got so far

- onedr0p https://github.com/onedr0p/home-ops
- kashalls https://github.com/kashalls/home-cluster
- bjw-s https://github.com/bjw-s/home-ops 

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