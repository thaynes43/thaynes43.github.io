---
title: Bare Metal K8S
permalink: /docs/moving-day/bare-metal/
---

Goal for the new stack is [Omni](https://haynes.omni.siderolabs.io/omni) controlled bare metal Talos stack.

https://www.talos.dev/v1.7/talos-guides/install/omni/

## Reclaim Nodes from PVE

Follow [these steps](https://forum.proxmox.com/threads/pve-remove-one-node-from-ceph-cluster.122456/) to remove nodes from proxmox. 

Then clean it out here:

```bash
root@pve01:/etc/pve/nodes# cd /etc/pve/nodes
root@pve01:/etc/pve/nodes# rm -R pve05/
```

You can also clean the node itself out by deleting:

```bash
systemctl stop pve-cluster corosync
pmxcfs -l
rm /etc/corosync/*
rm /etc/pve/corosync.conf
killall pmxcfs
systemctl start pve-cluster
```

And maybe even `rm -R /etc/pve/nodes` but I didn't get that far.

## Update MS-01 BIOS

First upgrade BIOS for MS-01.
- [tutorial](https://www.virtualizationhowto.com/2024/09/how-to-upgrade-the-minisforum-ms-01-bios/) 
- [download](https://www.minisforum.com/support/?lang=en#/support/page/download/108)

## Prep Installation Media

Using Omni we can download an ISO and then use rufus to image the nodes.

### Cluster Config

Since this is a single node cluster to start we need:

```yaml
cluster:
    allowSchedulingOnControlPlanes: true
```

### Talos Extensions

Then figure out [extensions](https://github.com/siderolabs/extensions) so we can make installation media.

- intel-ucode
- nut-client
- nvidia-container-toolkit-production
- nvidia-open-gpu-kernel-modules-production
- thunderbolt

Note that some extensions need configuration. Must follow instructions for found [here](https://github.com/siderolabs/extensions/tree/main).

#### Configure NVIDIA

Instructions for ext-nvidia-persistenced are [here](https://github.com/siderolabs/extensions/tree/main/nvidia-gpu/nvidia-container-toolkit).


```yaml
machine:
  sysctls:
    net.core.bpf_jit_harden: 1
```

Found some more NVIDIA stuff [here](https://www.talos.dev/v1.7/talos-guides/configuration/nvidia-gpu-proprietary/)! 

This worked:

```yaml
machine:
  kernel:
    modules:
      - name: nvidia
        parameters:
            - "NVreg_OpenRmEnableUnsupportedGpus=1"
      - name: nvidia_uvm
      - name: nvidia_drm
      - name: nvidia_modeset
```

https://pxe.factory.talos.dev/pxe/2de5e70592507dc569fcedbafa8b6dae666aada2945206aaf565fe8975773229/1.7.4/metal-amd64

- ext-nut-client "Waiting for extension service config" I think I didn't do this: https://github.com/siderolabs/extensions/tree/main/power/nut-client

- ext-qemu-guest-agent "Waiting for file "/dev/virtio-ports/org.qemu.guest_agent.0" to exist" (since this isn't a VM I don't need this!!!!!)

Nut needs a config patch:

```yaml
configPatches:
  - path: /system/files
    op: add
    value:
      - path: "/var/etc/nut/upsmon.conf"
        contents: |
          MONITOR myups@localhost 1 upsmon secret master
          MINSUPPLIES 1
          SHUTDOWNCMD "/sbin/shutdown -h +0"
          NOTIFYCMD /sbin/upssched
          POLLFREQ 
```



Complaining that I need to set kernel module parameter `modprobe nvidia NVreg_OpenRmEnableUnsupportedGpus=1` as described [here](https://download.nvidia.com/XFree86/Linux-x86_64/515.43.04/README/kernel_open.html)

### Configure Nut Client

`MONITOR APC-900W-01@nut01.haynesnetwork 1 admin REDACTED slave`
```yaml
---
apiVersion: v1alpha1
kind: ExtensionServiceConfig
name: nut-client
configFiles:
  - content: |-
        MONITOR HaynesTowerUPS@haynestower.haynesnetwork:3493 1 thaynes REDACTED secondary
        SHUTDOWNCMD "/sbin/poweroff"
    mountPath: /usr/local/etc/nut/upsmon.conf
```