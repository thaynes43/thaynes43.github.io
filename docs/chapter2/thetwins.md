---
title: The Twins
permalink: /docs/chapter2/thetwins/
---

# The Last Mile

Time to reboot some documentation as I bring the final hardware online. The cluster is running well and Ceph is now configured on the Twins + HI for a three node 600TB HDD array. We are going to use this CephFS small file storage while using unRAID for big media files. 

## iGPU Passthrough

This got crazy last time by splitting the iGPU in a way that VMs could share it. Since only the Talos VM needs it things can be much easier. From there Talos can share it with the pods on the host.

### Prepping Proxmox

#### Enable VFIO Kernel modules:

```sh
nano /etc/modules

// add these
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

#### Configure GRUB

```sh
nano /etc/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

update-grub
reboot
```

#### Blacklist iGPU Driver

```sh
echo "blacklist i915" > /etc/modprobe.d/blacklist-i915.conf
update-initramfs -u
reboot
```

#### Bind iGPU to vfio-pci:

```sh
lspci -nn | grep -i vga
00:02.0 VGA compatible controller [0300]: Intel Corporation Arrow Lake-S [Intel Graphics] [8086:7d67] (rev 06)
00:02.0 VGA compatible controller [0300]: Intel Corporation Arrow Lake-S [Intel Graphics] [8086:7d67] (rev 06)

PCI ID: 00:02.0
Vendor:Device: 8086:7d67

nano /etc/modprobe.d/vfio.conf
options vfio-pci ids=8086:7d67
update-initramfs -u
reboot

// confirm

lspci -nnk | grep -A3 -i 'VGA'
Kernel driver in use: vfio-pci
```

#### pciutils for debugging

(optional)

```sh
apt install pciutils
find /sys/kernel/iommu_groups/ -type l
```

### Creating the VM

Straightforward so far:

- q35 machine
- host CPU
- 512 HDD write-through cache
- 163840 Max, 40960 Min memory

Once it starts mash escape to disable secure boot or it won't work.

Add the GPU identified via `lspci -nn | grep -i vga`

- Pick the right bus
- Select pci express but not main GPU
- Display using VirtIO instead of default

### node-feature-discovery

TODO find the right values with `lspci -nn | grep -i vga` and apply to cluster.

```
00:02.0 VGA compatible controller [0300]: Intel Corporation Arrow Lake-S [Intel Graphics] [8086:7d67] (rev 06)
```

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/nfd.k8s-sigs.io/nodefeaturerule_v1alpha1.json
apiVersion: nfd.k8s-sigs.io/v1alpha1
kind: NodeFeatureRule
metadata:
  name: intel-arrow-lake-gpu
spec:
  rules:
    - name: intel-arrow-lake.gpu
      labels:
        feature.node.kubernetes.io/intel-arrow-lake-gpu: "true"
      matchFeatures:
        - feature: pci.device
          matchExpressions:
            class:  { op: In, value: ["0300"] }
            vendor: { op: In, value: ["8086"] }
            device: { op: In, value: ["7d67"] }
```

## Joining the family

Now that the machine is up with the iGPU maybe passed through we can update Omni's template.

twin-top Machine ID: `2fe46add-9e72-401c-8ec1-b5fb6837ffa0`
twin-bottom Machine ID: `960513a6-7a1d-4ece-949d-54a022fe85e5`

```yaml
---
kind: Machine
systemExtensions:
  - siderolabs/qemu-guest-agent
name: 960513a6-7a1d-4ece-949d-54a022fe85e5
patches:
  - idOverride: 400-cm-960513a6-7a1d-4ece-949d-54a022fe85e5
    inline:
      machine:
        install:
          extraKernelArgs:
            - net.ifnames=0
          wipe: true
        network:
          hostname: talosw03
          interfaces:
            - deviceSelector:
                hardwareAddr: BC:24:11:C5:5B:92
              mtu: 1500
              dhcp: true
          disableSearchDomain: true
        sysctls:
          net.core.bpf_jit_harden: 1
          fs.inotify.max_queued_events: "65536"
          fs.inotify.max_user_instances: "8192"
          fs.inotify.max_user_watches: "524288"
          net.core.rmem_max: "7500000"
          net.core.wmem_max: "7500000"
        kubelet:
          extraArgs:
            rotate-server-certificates: "true"
          defaultRuntimeSeccompProfileEnabled: true
          nodeIP:
            validSubnets:
              - 192.168.40.0/24
          disableManifestsDirectory: true
          extraMounts:
            - destination: /var/openebs/local
              type: bind
              source: /var/openebs/local
              options:
                - bind
                - rshared
                - rw
        files:
            - content: |
                [plugins."io.containerd.grpc.v1.cri"]
                  enable_unprivileged_ports = true
                  enable_unprivileged_icmp = true
                [plugins."io.containerd.grpc.v1.cri".containerd]
                  discard_unpacked_layers = false
                [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
                  discard_unpacked_layers = false
              permissions: 0
              path: /etc/cri/conf.d/20-customization.part
              op: create
            - content: |
                [ NFSMount_Global_Options ]
                nfsvers=4.2
                hard=True
                noatime=True
                nodiratime=True
                rsize=131072
                wsize=131072
                nconnect=8
              permissions: 420
              path: /etc/nfsmount.conf
              op: overwrite
        features:
            rbac: true
            stableHostname: true
            apidCheckExtKeyUsage: true
            diskQuotaSupport: true
            kubePrism:
                enabled: true
                port: 7445
            hostDNS:
                enabled: true
                resolveMemberNames: true
                forwardKubeDNSToHost: false
        nodeLabels:
            topology.kubernetes.io/region: main
            topology.kubernetes.io/zone: w
```

## TODOs

Quick break from our saga to denote some things we must do soooon.

- Move [NFS Ganesha](https://github.com/nfs-ganesha/nfs-ganesha) server to k8s or at least add monitoring cause it gets weird sometimes
- Setup NUT again, across the board
- Remove NFS dependency from unRAID Plex
- Auto updates instead of manually merging
- Fix alarms and pushover notifications (monitoring & observability)

Later on:

- After adding music check out https://www.meilisearch.com/docs/learn/getting_started/what_is_meilisearch for an API people can search and request things. It's used [for kyoo](https://github.com/zoriya/kyoo) which is a immature but neat project

## CephFS Plex

The iGPU in these was all for running a secondary instance of Plex that could roll over to either twin on failure. For now this will just host small file content in CephFS.

Digging out some examples - the golden standards are using custom images (likely rootless):

- https://github.com/onedr0p/home-ops/blob/main/kubernetes/apps/default/plex/app/helmrelease.yaml
- https://github.com/bjw-s-labs/home-ops/blob/main/kubernetes/apps/media/plex/app/helmrelease.yaml

There are some on kubesearch using the official image with the app template

- https://github.com/lucidph3nx/home-ops/blob/main/kubernetes/cluster0/apps/media/plex/app/helm-release.yaml

There is even an official Plex helm chart people are using:

- https://github.com/denniseffing/home-infrastructure/blob/main/kubernetes/main/apps/default/plex-media-server/app/helmrelease.yaml

### But first, limits

I want to be able to configure something like:

```yaml
            resources:
              requests:
                cpu: 100m
              limits:
                gpu.intel.com/i915: 1
                memory: 9248M
```

For my iGPU. So I need the intel device plugin. 