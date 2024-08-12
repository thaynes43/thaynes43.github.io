---
title: VLAN Migration - IP Disaster
permalink: /docs/vlan-migration/ip-disaster/
---

## Learning by Fire

### Create the VLAN

This was done in my failed attempt to just change the clusters IPs:

![k8s vlan]({{ site.url }}/images/unifi/k8s-vlan.png)

#### Changing IP in Proxmox

1. Make sure VPNLan is up to date and we can access the node from there
1. Update vmbr0 for the new IP and gateway 
1. Edit `/etc/hosts/` to fix the hostname
1. Edit `/etc/pve/corosync.conf/` to fix the cluster 
1. Edit `/etc/pve/priv/known_hosts`
1. Few more from grepping /etc/pve the old IP (192.168.0.34)
1. Edit `/etc/resolv.conf`
1. Reboot

I also did this on the node:

```
Update Network Settings:
Log in to the node you want to modify.
Edit the following files to reflect the new IP address:
/etc/network/interfaces: Update the IP address and gateway.
/etc/hosts: Replace any host entries with the new IP addresses.
/etc/resolv.conf: Adjust the DNS server if necessary.
Cluster Configuration:
Stop the cluster services:
systemctl stop pve-cluster
systemctl stop corosync

Force local mode to update the cluster configuration:
pmxcfs -l

Edit /etc/pve/corosync.conf:
Replace the old IPs with the new IPs for all hosts.
Increment the config_version by one to prevent overwriting.
If using Ceph, also update /etc/ceph/ceph.conf.
Known Hosts File:
Edit /etc/pve/priv/known_hosts to have the correct IPs.
Verify and Restart:
Check for any remaining instances of the old IP:
cd /etc && grep -R '192.168.1.'
cd /var && grep -R '192.168.1.'

Restart the cluster:
killall pmxcfs
systemctl start pve-cluster
```

And I tried some stuff [here](https://forum.proxmox.com/threads/change-ip-of-cluster-node.106676/post-459672) so I'm not 100% what worked.

Ceph is stull a mess too.

[This guide](https://www.reddit.com/r/Proxmox/comments/ttsstt/the_wifes_out_how_do_i_change_the_subnet_of_my/) looks good but I'd rather do them one at a time.

##### PVE02 - Cleaner?

BEFORE CHANGING THE IP FOR REAL

1. Delete Ceph Mons and Mgrs
1. Fix VPNLan (I changed that subnet)
1. Roll out `/etc/pve/corosync.conf` changes with the anticipated IP, `192.168.40.7`, and restart it `systemctl restart corosync`. You will now be alone as a node.

CHANGING THE IP

1. Edit `/etc/hosts` and `/etc/resolv.conf`, change DNS and interfaces to `192.168.40.7` and THEN add VLAN for port of switch
1. `ifdown vmbr0; ifup vmbr0` for good measure
1. On node run `systemctl restart corosync` and `systemctl restart pve-cluster`
1. On every other node that are in the cluster run `systemctl restart corosync`
1. Reboot node

That was cleaner!

##### PVE03 - WOW

It went to shit when I followed the steps. I think somehow the NUT server was involved...

```bash
service corosync restart
service pve-cluster restart
service pveproxy restart
service pvedaemon restart
service pvestatd restart
```

``` bash
root@pve01:~# systemctl stop pve-cluster
root@pve01:~# systemctl stop corosync
root@pve01:~# pmxcfs -l
[main] notice: resolved node name 'pve01' to '192.168.40.6' for default node IP address
[main] notice: forcing local mode (although corosync.conf exists)
root@pve01:~# rm /etc/pve/corosync.conf
root@pve01:~# rm -r /etc/corosync/*
root@pve01:~# killall pmxcfs
root@pve01:~# systemctl start pve-cluster
root@pve01:~# systemctl start corosync
```

root@pve02:/etc/corosync# ls
authkey  corosync.conf  uidgid.d

And copy /etc/pve/corosync.conf

```
pmxcfs -l
root@HaynesIntelligence:/var/log/pveproxy# pmxcfs -l
[main] notice: resolved node name 'HaynesIntelligence' to '192.168.0.250' for default node IP address
[main] notice: unable to acquire pmxcfs lock - trying again
[main] crit: unable to acquire pmxcfs lock: Resource temporarily unavailable
[main] notice: exit proxmox configuration filesystem (-1)
```
13360
pve01:00003430:0005877D:66B94C77:srvreload:networking:root@pam

##### PVE05 - Cause I Can't Hit 4

BEFORE CHANGING THE IP FOR REAL

1. Delete Ceph Mons and Mgrs
1. Fix VPNLan (I changed that subnet)
1. Roll out `/etc/pve/corosync.conf` changes with the anticipated IP, `192.168.40.7`, and restart it `systemctl restart corosync`. You will now be alone as a node.

CHANGING THE IP

1. Edit `/etc/hosts` and `/etc/resolv.conf`, change DNS and interfaces to `192.168.40.10` and THEN add VLAN for port of switch
1. `ifdown vmbr0; ifup vmbr0` for good measure
1. On node run `systemctl restart corosync` and `systemctl restart pve-cluster`
1. On every other node that are in the cluster run `systemctl restart corosync`
1. Reboot node

It got stuck in a weird microcluster with pve04 which fine before. Running this across nodes I found pve01 sill saw it in `cat /etc/pve/.members`:

```
cd /etc && grep -R '192.168.0.9'
cd /var && grep -R '192.168.0.9'
```

Running `service pve-cluster restart` on pve01 fixed everything.

##### PVE04 - Always Last

This guy had the wrong NIC configured for the VPN VLan which I am using to manage the node when it's knocked off the main network. Needed to fix that anyways....

1. Delete Ceph Mons and Mgrs
1. Roll out `/etc/pve/corosync.conf` changes with the anticipated IP, `192.168.40.7`, and restart it `systemctl restart corosync`. You will now be alone as a node.

CHANGING THE IP

1. Edit `/etc/hosts` and `/etc/resolv.conf`, change DNS and interfaces to `192.168.40.9` and THEN add VLAN for port of switch
1. `ifdown vmbr0; ifup vmbr0` for good measure
1. On node run `systemctl restart corosync` and `systemctl restart pve-cluster`
1. On every other node that are in the cluster run `systemctl restart corosync` and check with `cat /etc/pve/.members`
1. Reboot node

### Fix Ceph Public Network

[This post](https://forum.proxmox.com/threads/change-public-network-and-or-cluster-network-in-ceph.92581/) makes it sound pretty easy but [here](https://forum.proxmox.com/threads/not-able-to-use-pveceph-purge-to-completely-remove-ceph.59606/#post-274958) someone had to delete mons to get it to work which I think is the right way to do it.

