---
title: Cluster2
permalink: /docs/talos/cluster2/
---

I have decided to do a bake off and move to two 3 node clusters, each using a [Ring Network](https://gist.github.com/gavinmcfall/ea6cb1233d3a300e9f44caf65a32d519) for Ceph's private network. Oen will use proxmox while the other hardcore bare metal k8s.

To do this I will need to [remove OSDs](https://docs.ceph.com/en/latest/rados/operations/add-or-rm-osds/#:~:text=It%20is%20possible%20to%20remove%20an%20OSD%20manually) from two of my nodes in the other cluster.