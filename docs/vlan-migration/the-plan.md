---
title: Ip Migration Plan
permalink: /docs/vlan-migration/the-plan/
---

I got pretty far into my cluster without flux so there was a mismatch between paradigms. I also want to move the homelab to it's own VLAN and changing the IP of the cluster hosts seems to be very difficult. Since I had some housekeeping to do I think pulling the band-aid all at once will be easiest.

## The Plan

1. Create `Hayneslab VLAN` (192.168.40.x) that allows traffic to and from my default subnet
1. Move the MS-01's primary 10Gb NIC to this VLAN and reconfigure the bridges
1. Fix Ceph to use this as the public network
1. Create and configure Debian VMs for Kubernetes worker and control planes (HA) on the VLAN that is now vmbr0
1. Boostrap - Init the server, join the workers, and install flux (using [k3s](https://docs.k3s.io/datastore/ha-embedded) ths time)
1. GitOps rook-ceph, ceph-csi, and snapshotters which I don't have in my repo yet
1. Fix sealed secrets since I didn't do it right the first ime

## The Execution

Step #1 was suck a disaster it deserved it's own fucking page. I'll summarize this here 

1. ![Hayneslab VLAN Fiasco]({{ site.url }}/docs/vlan-migration/ip-disaster/) 
1. ![New k8s Cluster]({{ site.url }}/docs/vlan-migration/build-k3s-cluster/) 
1. ![Re-Flux]({{ site.url }}/docs/vlan-migration/flux-migration/)

### Step2 1 - 3 ... Disaster

Migrating to a VLAN was a disaster. You can read about it ![here]({{ site.url }}/docs/vlan-migration/ip-disaster/) but long story short it's really hard to change the IPs of your nodes in the cluster. This plus network loops plus lost connection to the NUT Server shutting everything down WAAYY faster than 300 seconds lead to a long day.

### Steps 4 - ??

Part of the motivation behind this was that I only had an [old gist](https://gist.github.com/thaynes43/6135cdde0b228900d70ab49dfe386f91) and a test file with notes documenting how I set things up initially. I was going to use this to document what I missed from the early days but here we are with a mini series.