---
title: Rook Ceph on Baremetal
permalink: /docs/talos/rook-ceph/
---

## Preface

This is getting hairy to set up, few issues so far...

### Talosm01 Problem:

nvme0n1 changed from S436NA0N665541 to CT1000T500SSD8! Now talos is again installed on the wrong disk. However, this device has two disks and nvm2n1 was sipped because it was xfs -> `2024-10-13 14:56:59.743775 I | cephosd: skipping device "nvme2n1" because it contains a filesystem "xfs"`. 

```bash
⬢ [Docker] ❯ talosctl -n 192.168.40.93 disks
NODE            DEV            MODEL                        SERIAL           TYPE      UUID   WWID                                   MODALIAS      NAME   SIZE     BUS_PATH                                                                                SUBSYSTEM          READ_ONLY   SYSTEM_DISK
192.168.40.93   /dev/loop0     -                            -                UNKNOWN   -      -                                      -             -      139 kB   /virtual                                                                                /sys/class/block   *           
192.168.40.93   /dev/loop1     -                            -                UNKNOWN   -      -                                      -             -      4.1 kB   /virtual                                                                                /sys/class/block   *           
192.168.40.93   /dev/loop2     -                            -                UNKNOWN   -      -                                      -             -      295 kB   /virtual                                                                                /sys/class/block   *           
192.168.40.93   /dev/loop3     -                            -                UNKNOWN   -      -                                      -             -      6.1 MB   /virtual                                                                                /sys/class/block   *           
192.168.40.93   /dev/loop4     -                            -                UNKNOWN   -      -                                      -             -      179 MB   /virtual                                                                                /sys/class/block   *           
192.168.40.93   /dev/loop5     -                            -                UNKNOWN   -      -                                      -             -      123 kB   /virtual                                                                                /sys/class/block   *           
192.168.40.93   /dev/loop6     -                            -                UNKNOWN   -      -                                      -             -      4.1 kB   /virtual                                                                                /sys/class/block   *           
192.168.40.93   /dev/loop7     -                            -                UNKNOWN   -      -                                      -             -      77 MB    /virtual                                                                                /sys/class/block   *           
192.168.40.93   /dev/nvme0n1   CT1000T500SSD8               2409476716D3     NVME      -      eui.000000000000000100a07524476716d3   -             -      1.0 TB   /pci0000:00/0000:00:06.0/0000:02:00.0/nvme                                              /sys/class/block               *
192.168.40.93   /dev/nvme1n1   SAMSUNG MZ1LB1T9HALS-00007   S436NA0N639078   NVME      -      eui.343336304e6390780025384100000001   -             -      1.9 TB   /pci0000:00/0000:00:1d.0/0000:5a:00.0/nvme                                              /sys/class/block               
192.168.40.93   /dev/nvme2n1   SAMSUNG MZ1LB1T9HALS-00007   S436NA0N665541   NVME      -      eui.343336304e6655410025384100000001   -             -      1.9 TB   /pci0000:00/0000:00:1c.4/0000:59:00.0/nvme                                              /sys/class/block               
192.168.40.93   /dev/sda       SanDisk 3.2Gen1              -                HDD       -      -                                      scsi:t-0x00   -      31 GB    /pci0000:00/0000:00:14.0/usb4/4-3/4-3:1.0/host0/target0:0:0/0:0:0:0                     /sys/class/block               
192.168.40.93   /dev/sr0       CD-ROM Drive                 -                CD        -      -                                      scsi:t-0x05   -      0 B      /pci0000:00/0000:00:14.0/usb3/3-2/3-2.1/3-2.1.1/3-2.1.1:1.3/host1/target1:0:0/1:0:0:0   /sys/class/block               
```

```bash
⬢ [Docker] ❯ talosctl -n 192.168.40.93 get blockdevices
NODE            NAMESPACE   TYPE          ID          VERSION   TYPE        PARTITIONNAME   GENERATION
192.168.40.93   runtime     BlockDevice   loop0       2         disk                        2
192.168.40.93   runtime     BlockDevice   loop1       2         disk                        2
192.168.40.93   runtime     BlockDevice   loop2       2         disk                        2
192.168.40.93   runtime     BlockDevice   loop3       2         disk                        2
192.168.40.93   runtime     BlockDevice   loop4       2         disk                        2
192.168.40.93   runtime     BlockDevice   loop5       2         disk                        2
192.168.40.93   runtime     BlockDevice   loop6       2         disk                        2
192.168.40.93   runtime     BlockDevice   loop7       2         disk                        2
192.168.40.93   runtime     BlockDevice   nvme0n1     13        disk                        13
192.168.40.93   runtime     BlockDevice   nvme0n1p1   5         partition   EFI             5
192.168.40.93   runtime     BlockDevice   nvme0n1p2   5         partition   BIOS            5
192.168.40.93   runtime     BlockDevice   nvme0n1p3   5         partition   BOOT            5
192.168.40.93   runtime     BlockDevice   nvme0n1p4   7         partition   META            7
192.168.40.93   runtime     BlockDevice   nvme0n1p5   5         partition   STATE           5
192.168.40.93   runtime     BlockDevice   nvme0n1p6   5         partition   EPHEMERAL       5
192.168.40.93   runtime     BlockDevice   nvme1n1     5         disk                        5
192.168.40.93   runtime     BlockDevice   nvme2n1     5         disk                        5
192.168.40.93   runtime     BlockDevice   sda         6         disk                        6
192.168.40.93   runtime     BlockDevice   sda1        1         partition                   1
192.168.40.93   runtime     BlockDevice   sr0         2         disk                        2
```

### Talosm02 Problem: OS Installed on Samsung NVMe

```bash
⬢ [Docker] ❯ talosctl -n 192.168.40.59 disks
NODE            DEV            MODEL                SERIAL           TYPE      UUID   WWID                                   MODALIAS   NAME   SIZE     BUS_PATH                                     SUBSYSTEM          READ_ONLY   SYSTEM_DISK
192.168.40.59   /dev/loop0     -                    -                UNKNOWN   -      -                                      -          -      135 kB   /virtual                                     /sys/class/block   *           
192.168.40.59   /dev/loop1     -                    -                UNKNOWN   -      -                                      -          -      4.1 kB   /virtual                                     /sys/class/block   *           
192.168.40.59   /dev/loop2     -                    -                UNKNOWN   -      -                                      -          -      295 kB   /virtual                                     /sys/class/block   *           
192.168.40.59   /dev/loop3     -                    -                UNKNOWN   -      -                                      -          -      123 kB   /virtual                                     /sys/class/block   *           
192.168.40.59   /dev/loop4     -                    -                UNKNOWN   -      -                                      -          -      4.1 kB   /virtual                                     /sys/class/block   *           
192.168.40.59   /dev/loop5     -                    -                UNKNOWN   -      -                                      -          -      77 MB    /virtual                                     /sys/class/block   *           
192.168.40.59   /dev/nvme0n1   MZ1LB1T9HBLS-000FB   S5XANE0NC09500   NVME      -      eui.355841304ec095000025384500000001   -          -      1.9 TB   /pci0000:00/0000:00:1c.4/0000:58:00.0/nvme   /sys/class/block               *
192.168.40.59   /dev/nvme1n1   CT1000T500SSD8       2409474952B4     NVME      -      eui.000000000000000100a07524474952b4   -          -      1.0 TB   /pci0000:00/0000:00:06.0/0000:01:00.0/nvme   /sys/class/block               
```

```bash
⬢ [Docker] ❯ talosctl -n 192.168.40.59 get blockdevices
NODE            NAMESPACE   TYPE          ID          VERSION   TYPE        PARTITIONNAME   GENERATION
192.168.40.59   runtime     BlockDevice   loop0       2         disk                        2
192.168.40.59   runtime     BlockDevice   loop1       2         disk                        2
192.168.40.59   runtime     BlockDevice   loop2       2         disk                        2
192.168.40.59   runtime     BlockDevice   loop3       2         disk                        2
192.168.40.59   runtime     BlockDevice   loop4       2         disk                        2
192.168.40.59   runtime     BlockDevice   loop5       2         disk                        2
192.168.40.59   runtime     BlockDevice   loop6       2         disk                        2
192.168.40.59   runtime     BlockDevice   loop7       2         disk                        2
192.168.40.59   runtime     BlockDevice   nvme0n1     13        disk                        13
192.168.40.59   runtime     BlockDevice   nvme0n1p1   5         partition   EFI             5
192.168.40.59   runtime     BlockDevice   nvme0n1p2   5         partition   BIOS            5
192.168.40.59   runtime     BlockDevice   nvme0n1p3   5         partition   BOOT            5
192.168.40.59   runtime     BlockDevice   nvme0n1p4   7         partition   META            7
192.168.40.59   runtime     BlockDevice   nvme0n1p5   5         partition   STATE           5
192.168.40.59   runtime     BlockDevice   nvme0n1p6   5         partition   EPHEMERAL       5
192.168.40.59   runtime     BlockDevice   nvme1n1     5         disk                        5
```

### Talosm03 Problem:

Configured for nvme1n1 which is also CT1000T500SSD8! 

```bash
⬢ [Docker] ❯ talosctl -n 192.168.40.10 disks
NODE            DEV            MODEL                SERIAL           TYPE      UUID   WWID                                   MODALIAS   NAME   SIZE     BUS_PATH                                     SUBSYSTEM          READ_ONLY   SYSTEM_DISK
192.168.40.10   /dev/loop0     -                    -                UNKNOWN   -      -                                      -          -      135 kB   /virtual                                     /sys/class/block   *           
192.168.40.10   /dev/loop1     -                    -                UNKNOWN   -      -                                      -          -      4.1 kB   /virtual                                     /sys/class/block   *           
192.168.40.10   /dev/loop2     -                    -                UNKNOWN   -      -                                      -          -      295 kB   /virtual                                     /sys/class/block   *           
192.168.40.10   /dev/loop3     -                    -                UNKNOWN   -      -                                      -          -      123 kB   /virtual                                     /sys/class/block   *           
192.168.40.10   /dev/loop4     -                    -                UNKNOWN   -      -                                      -          -      4.1 kB   /virtual                                     /sys/class/block   *           
192.168.40.10   /dev/loop5     -                    -                UNKNOWN   -      -                                      -          -      77 MB    /virtual                                     /sys/class/block   *           
192.168.40.10   /dev/nvme0n1   MZ1LB1T9HBLS-000FB   S5XANA0T113526   NVME      -      eui.35584130541135260025384100000001   -          -      1.9 TB   /pci0000:00/0000:00:1c.4/0000:58:00.0/nvme   /sys/class/block               
192.168.40.10   /dev/nvme1n1   CT1000T500SSD8       240646E5C076     NVME      -      eui.000000000000000100a0752446e5c076   -          -      1.0 TB   /pci0000:00/0000:00:06.0/0000:01:00.0/nvme   /sys/class/block               *
```

```bash
⬢ [Docker] ❯ talosctl -n 192.168.40.10 get blockdevices
NODE            NAMESPACE   TYPE          ID          VERSION   TYPE        PARTITIONNAME   GENERATION
192.168.40.10   runtime     BlockDevice   loop0       2         disk                        2
192.168.40.10   runtime     BlockDevice   loop1       2         disk                        2
192.168.40.10   runtime     BlockDevice   loop2       2         disk                        2
192.168.40.10   runtime     BlockDevice   loop3       2         disk                        2
192.168.40.10   runtime     BlockDevice   loop4       2         disk                        2
192.168.40.10   runtime     BlockDevice   loop5       2         disk                        2
192.168.40.10   runtime     BlockDevice   loop6       2         disk                        2
192.168.40.10   runtime     BlockDevice   loop7       2         disk                        2
192.168.40.10   runtime     BlockDevice   nvme0n1     5         disk                        5
192.168.40.10   runtime     BlockDevice   nvme1n1     13        disk                        13
192.168.40.10   runtime     BlockDevice   nvme1n1p1   5         partition   EFI             5
192.168.40.10   runtime     BlockDevice   nvme1n1p2   5         partition   BIOS            5
192.168.40.10   runtime     BlockDevice   nvme1n1p3   5         partition   BOOT            5
192.168.40.10   runtime     BlockDevice   nvme1n1p4   7         partition   META            7
192.168.40.10   runtime     BlockDevice   nvme1n1p5   5         partition   STATE           5
192.168.40.10   runtime     BlockDevice   nvme1n1p6   5         partition   EPHEMERAL       5
```

## Path Forward

Blow it up, re-install on the right drive, somehow format the other shit.

[GParted](https://gparted.org/) was a lifesaver here. Somehow I got rid of the partition tables on every disk, they were fucked. After starting fresh on all the disks I booted each node to Talos via the USB but made sure to remove the USB. Then I ran sync against my template to bootstrap the cluster and hope the install hits the right disks this time!

The paths for each disk changed yet again but the partitions for talos are in the right spot!

### talosm01:

```bash
talosctl -n 192.168.40.93 disks
192.168.40.93   /dev/nvme0n1   SAMSUNG MZ1LB1T9HALS-00007   S436NA0N665541   NVME      -      eui.343336304e6655410025384100000001   -             -      1.9 TB   /pci0000:00/0000:00:1c.4/0000:59:00.0/nvme                                              /sys/class/block               
192.168.40.93   /dev/nvme1n1   SAMSUNG MZ1LB1T9HALS-00007   S436NA0N639078   NVME      -      eui.343336304e6390780025384100000001   -             -      1.9 TB   /pci0000:00/0000:00:1d.0/0000:5a:00.0/nvme                                              /sys/class/block               
192.168.40.93   /dev/nvme2n1   CT1000T500SSD8               2409476716D3     NVME      -      eui.000000000000000100a07524476716d3   -             -      1.0 TB   /pci0000:00/0000:00:06.0/0000:02:00.0/nvme                                              /sys/class/block               *
```

```bash
192.168.40.93   runtime     BlockDevice   nvme0n1     1         disk                        1
192.168.40.93   runtime     BlockDevice   nvme1n1     1         disk                        1
192.168.40.93   runtime     BlockDevice   nvme2n1     9         disk                        9
192.168.40.93   runtime     BlockDevice   nvme2n1p1   3         partition   EFI             3
192.168.40.93   runtime     BlockDevice   nvme2n1p2   3         partition   BIOS            3
192.168.40.93   runtime     BlockDevice   nvme2n1p3   3         partition   BOOT            3
192.168.40.93   runtime     BlockDevice   nvme2n1p4   3         partition   META            3
192.168.40.93   runtime     BlockDevice   nvme2n1p5   3         partition   STATE           3
192.168.40.93   runtime     BlockDevice   nvme2n1p6   3         partition   EPHEMERAL       3
```

### talosm02:

```bash
talosctl -n 192.168.40.59 disks
192.168.40.59   /dev/nvme0n1   CT1000T500SSD8       2409474952B4     NVME      -      eui.000000000000000100a07524474952b4   -          -      1.0 TB   /pci0000:00/0000:00:06.0/0000:01:00.0/nvme   /sys/class/block               *
192.168.40.59   /dev/nvme1n1   MZ1LB1T9HBLS-000FB   S5XANE0NC09500   NVME      -      eui.355841304ec095000025384500000001   -          -      1.9 TB   /pci0000:00/0000:00:1c.4/0000:58:00.0/nvme   /sys/class/block               
```

```bash
talosctl -n 192.168.40.59 get blockdevices
192.168.40.59   runtime     BlockDevice   nvme0n1     9         disk                        9
192.168.40.59   runtime     BlockDevice   nvme0n1p1   3         partition   EFI             3
192.168.40.59   runtime     BlockDevice   nvme0n1p2   3         partition   BIOS            3
192.168.40.59   runtime     BlockDevice   nvme0n1p3   3         partition   BOOT            3
192.168.40.59   runtime     BlockDevice   nvme0n1p4   3         partition   META            3
192.168.40.59   runtime     BlockDevice   nvme0n1p5   3         partition   STATE           3
192.168.40.59   runtime     BlockDevice   nvme0n1p6   3         partition   EPHEMERAL       3
192.168.40.59   runtime     BlockDevice   nvme1n1     1         disk                        1
```

### talosm03:

```bash
talosctl -n 192.168.40.10 disks
192.168.40.10   /dev/nvme0n1   CT1000T500SSD8       240646E5C076     NVME      -      eui.000000000000000100a0752446e5c076   -          -      1.0 TB   /pci0000:00/0000:00:06.0/0000:01:00.0/nvme   /sys/class/block               *
192.168.40.10   /dev/nvme1n1   MZ1LB1T9HBLS-000FB   S5XANA0T113526   NVME      -      eui.35584130541135260025384100000001   -          -      1.9 TB   /pci0000:00/0000:00:1c.4/0000:58:00.0/nvme   /sys/class/block               
```

```bash
talosctl -n 192.168.40.10 get blockdevices
192.168.40.10   runtime     BlockDevice   nvme0n1     9         disk                        9
192.168.40.10   runtime     BlockDevice   nvme0n1p1   3         partition   EFI             3
192.168.40.10   runtime     BlockDevice   nvme0n1p2   3         partition   BIOS            3
192.168.40.10   runtime     BlockDevice   nvme0n1p3   3         partition   BOOT            3
192.168.40.10   runtime     BlockDevice   nvme0n1p4   3         partition   META            3
192.168.40.10   runtime     BlockDevice   nvme0n1p5   3         partition   STATE           3
192.168.40.10   runtime     BlockDevice   nvme0n1p6   3         partition   EPHEMERAL       3
192.168.40.10   runtime     BlockDevice   nvme1n1     1         disk                        1
```

## Networking

At first I thought host networking like this would cut it:

```yaml
      network:
        provider: host
        addressRanges:
          public:
            - "192.168.40.0/24"
          cluster:
            - "192.168.60.0/24"
```

But the problem is only one adapter is attached to the pod, `.40` is the host network, and `.40` cannot hit `.60`!

https://rook.io/docs/rook/latest/CRDs/Cluster/network-providers/#multus looks to be the solution and will also be handy for IoT network isolation but there's much to consume. 

```bash
NODE            NAMESPACE   TYPE         ID             VERSION   TYPE       KIND        HW ADDR                                           OPER STATE   LINK STATE
192.168.40.93   network     LinkStatus   bond0          1         ether      bond        96:14:8c:c7:5f:5c                                 down         false
192.168.40.93   network     LinkStatus   cni0           4         ether      bridge      36:55:7c:ff:2d:32                                 up           true
192.168.40.93   network     LinkStatus   dummy0         1         ether      dummy       62:89:59:f1:40:00                                 down         false
192.168.40.93   network     LinkStatus   enp3s0f0       4         ether                  58:47:ca:78:bf:f6                                 up           true
192.168.40.93   network     LinkStatus   enp3s0f1       4         ether                  58:47:ca:78:bf:f7                                 up           true
192.168.40.93   network     LinkStatus   enp88s0        2         ether                  58:47:ca:78:bf:f8                                 down         false
192.168.40.93   network     LinkStatus   enp91s0        2         ether                  58:47:ca:78:bf:f9                                 down         false
192.168.40.93   network     LinkStatus   flannel.1      2         ether      vxlan       5a:e7:c0:38:ac:a0                                 unknown      true
192.168.40.93   network     LinkStatus   ip6tnl0        1         tunnel6    ip6tnl      00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00   down         false
192.168.40.93   network     LinkStatus   kube-ipvs0     1         ether      dummy       a2:c2:86:f1:c8:9c                                 down         false
192.168.40.93   network     LinkStatus   lo             2         loopback               00:00:00:00:00:00                                 unknown      true
192.168.40.93   network     LinkStatus   siderolink     1         nohdr      wireguard                                                     unknown      false
192.168.40.93   network     LinkStatus   sit0           1         sit        sit         00:00:00:00                                       down         false
192.168.40.93   network     LinkStatus   teql0          1         void                                                                     down         false
192.168.40.93   network     LinkStatus   tunl0          1         ipip       ipip        00:00:00:00                                       down         false
192.168.40.93   network     LinkStatus   veth000392cc   2         ether      veth        ca:b3:71:bd:61:42                                 up           true
192.168.40.93   network     LinkStatus   veth063af229   2         ether      veth        22:45:3f:99:92:78                                 up           true
192.168.40.93   network     LinkStatus   veth08f4fe8d   1         ether      veth        ca:4f:cc:3c:08:8d                                 up           true
192.168.40.93   network     LinkStatus   veth0b0a7c03   1         ether      veth        96:42:94:01:af:90                                 up           true
192.168.40.93   network     LinkStatus   veth134c03ae   2         ether      veth        1a:6b:b0:08:e8:5e                                 up           true
192.168.40.93   network     LinkStatus   veth2915ea78   1         ether      veth        aa:ed:e5:d6:d4:07                                 up           true
192.168.40.93   network     LinkStatus   veth2f803cd4   2         ether      veth        b6:b3:e6:28:bc:b0                                 up           true
192.168.40.93   network     LinkStatus   veth2f86f829   2         ether      veth        fe:9d:f1:38:c5:a5                                 up           true
192.168.40.93   network     LinkStatus   veth4464e3b6   2         ether      veth        2a:53:24:fb:5f:56                                 up           true
192.168.40.93   network     LinkStatus   veth55c3d11b   2         ether      veth        76:d6:b6:2a:f5:53                                 up           true
192.168.40.93   network     LinkStatus   veth5abe09d5   2         ether      veth        9e:fa:5f:79:1b:71                                 up           true
192.168.40.93   network     LinkStatus   veth5c9f79a9   1         ether      veth        de:f0:b5:b9:1d:12                                 up           true
192.168.40.93   network     LinkStatus   veth5fde9d80   2         ether      veth        b6:24:2f:3e:3f:ac                                 up           true
192.168.40.93   network     LinkStatus   veth65df51e8   2         ether      veth        8e:84:f9:db:da:df                                 up           true
192.168.40.93   network     LinkStatus   veth6692df38   1         ether      veth        56:ac:4e:f1:c8:2e                                 up           true
192.168.40.93   network     LinkStatus   veth67a84b5c   1         ether      veth        d2:89:91:1c:87:7d                                 up           true
192.168.40.93   network     LinkStatus   veth69e64371   2         ether      veth        fe:60:f6:a7:05:3d                                 up           true
192.168.40.93   network     LinkStatus   veth6a992a42   2         ether      veth        ca:12:be:2a:56:b9                                 up           true
192.168.40.93   network     LinkStatus   veth6cc6c55f   1         ether      veth        6a:0c:6b:65:5f:f4                                 up           true
192.168.40.93   network     LinkStatus   veth6eb87163   1         ether      veth        9a:8f:fb:e0:fa:7b                                 up           true
192.168.40.93   network     LinkStatus   veth6f48a91a   2         ether      veth        9a:d4:47:0c:13:5b                                 up           true
192.168.40.93   network     LinkStatus   veth723d6a2d   2         ether      veth        22:cc:38:1c:c7:b5                                 up           true
192.168.40.93   network     LinkStatus   veth775a2c58   2         ether      veth        46:0c:18:20:42:cc                                 up           true
192.168.40.93   network     LinkStatus   veth7c12785d   1         ether      veth        fa:b7:b3:de:8a:c5                                 up           true
192.168.40.93   network     LinkStatus   veth7f2d0ef7   2         ether      veth        aa:26:d1:2b:03:d7                                 up           true
192.168.40.93   network     LinkStatus   veth8392248e   1         ether      veth        16:af:98:c2:6f:61                                 up           true
192.168.40.93   network     LinkStatus   veth848ae99d   2         ether      veth        c6:59:30:82:95:4c                                 up           true
192.168.40.93   network     LinkStatus   veth853d8c78   1         ether      veth        6a:ed:10:2f:96:ad                                 up           true
192.168.40.93   network     LinkStatus   veth88952ede   2         ether      veth        c2:8b:f0:21:97:57                                 up           true
192.168.40.93   network     LinkStatus   veth8f122064   2         ether      veth        ea:c0:5e:fa:f0:17                                 up           true
192.168.40.93   network     LinkStatus   veth8fef19fe   2         ether      veth        06:b3:ab:1b:11:5a                                 up           true
192.168.40.93   network     LinkStatus   veth97fb9ccf   2         ether      veth        86:db:0f:95:f9:b6                                 up           true
192.168.40.93   network     LinkStatus   veth9950b22a   2         ether      veth        16:ff:89:a4:0f:f5                                 up           true
192.168.40.93   network     LinkStatus   veth99de7b85   2         ether      veth        4e:9c:35:26:fd:b6                                 up           true
192.168.40.93   network     LinkStatus   vetha6f8f70e   2         ether      veth        8a:2e:3b:5a:f1:51                                 up           true
192.168.40.93   network     LinkStatus   vethab83cb79   2         ether      veth        e2:39:60:47:b7:56                                 up           true
192.168.40.93   network     LinkStatus   vethadc74af8   2         ether      veth        46:02:a3:87:21:2e                                 up           true
192.168.40.93   network     LinkStatus   vethaee89621   2         ether      veth        b2:ed:ec:db:da:cb                                 up           true
192.168.40.93   network     LinkStatus   vethb2b919d4   1         ether      veth        a6:a3:87:ac:e8:fe                                 up           true
192.168.40.93   network     LinkStatus   vethb3bfe7b4   1         ether      veth        9a:09:d0:11:96:b8                                 up           true
192.168.40.93   network     LinkStatus   vethb58e69fe   2         ether      veth        fe:90:76:6a:cc:3e                                 up           true
192.168.40.93   network     LinkStatus   vethb706c6c8   2         ether      veth        92:c6:4e:32:b1:e9                                 up           true
192.168.40.93   network     LinkStatus   vethb71616f2   2         ether      veth        1e:47:61:1b:91:6d                                 up           true
192.168.40.93   network     LinkStatus   vethbd589115   2         ether      veth        26:4c:fd:d6:a3:98                                 up           true
192.168.40.93   network     LinkStatus   vethbe5629bd   2         ether      veth        76:d7:52:c9:25:df                                 up           true
192.168.40.93   network     LinkStatus   vethbf4be7bf   2         ether      veth        a6:59:a6:57:9d:73                                 up           true
192.168.40.93   network     LinkStatus   vethc044d996   2         ether      veth        aa:c2:ba:12:42:d4                                 up           true
192.168.40.93   network     LinkStatus   vethc66bf9e6   2         ether      veth        6a:4f:80:ec:6b:ea                                 up           true
192.168.40.93   network     LinkStatus   vethc8103177   2         ether      veth        fa:8e:7d:87:a2:d0                                 up           true
192.168.40.93   network     LinkStatus   vethcc660d9d   2         ether      veth        fe:2e:22:54:67:be                                 up           true
192.168.40.93   network     LinkStatus   vethd33f2689   2         ether      veth        2e:46:02:eb:de:06                                 up           true
192.168.40.93   network     LinkStatus   vethd5f06352   1         ether      veth        02:7a:f8:aa:85:43                                 up           true
192.168.40.93   network     LinkStatus   vethd8f5f456   2         ether      veth        ba:d7:9d:d1:dd:09                                 up           true
192.168.40.93   network     LinkStatus   vethe0ac43c8   2         ether      veth        32:3a:00:ff:77:79                                 up           true
192.168.40.93   network     LinkStatus   vethea406bdf   1         ether      veth        ca:73:30:13:69:9d                                 up           true
192.168.40.93   network     LinkStatus   vethec7a8cbf   1         ether      veth        e6:4a:5a:92:f0:70                                 up           true
192.168.40.93   network     LinkStatus   vethec971ae3   1         ether      veth        22:d6:af:d0:30:4d                                 up           true
192.168.40.93   network     LinkStatus   vethee08a680   2         ether      veth        d2:e9:e0:6b:3a:5f                                 up           true
192.168.40.93   network     LinkStatus   vethf449ab9e   1         ether      veth        6a:94:36:8c:b5:b2                                 up           true
192.168.40.93   network     LinkStatus   vethf7346d67   1         ether      veth        f6:cc:5e:26:28:00                                 up           true
192.168.40.93   network     LinkStatus   vethfb7c86de   2         ether      veth        56:a7:d0:cd:ab:f7                                 up           true
192.168.40.93   network     LinkStatus   vethfc437cfc   2         ether      veth        6a:73:26:f4:e8:9c                                 up           true
192.168.40.93   network     LinkStatus   vethfe25e63d   1         ether      veth        46:ed:18:20:ff:2a                                 up           true
```

```bash
NODE            NAMESPACE   TYPE         ID             VERSION   TYPE       KIND        HW ADDR                                           OPER STATE   LINK STATE
192.168.40.59   network     LinkStatus   bond0          1         ether      bond        32:96:2a:4b:57:4e                                 down         false
192.168.40.59   network     LinkStatus   cni0           2         ether      bridge      62:ec:6f:f6:9e:eb                                 up           true
192.168.40.59   network     LinkStatus   dummy0         1         ether      dummy       aa:58:b6:38:e5:fb                                 down         false
192.168.40.59   network     LinkStatus   enp2s0f0       4         ether                  58:47:ca:78:bc:4a                                 up           true
192.168.40.59   network     LinkStatus   enp2s0f1       4         ether                  58:47:ca:78:bc:4b                                 up           true
192.168.40.59   network     LinkStatus   enp87s0        2         ether                  58:47:ca:78:bc:4c                                 down         false
192.168.40.59   network     LinkStatus   enp90s0        2         ether                  58:47:ca:78:bc:4d                                 down         false
192.168.40.59   network     LinkStatus   flannel.1      2         ether      vxlan       d6:ad:4e:4d:f8:db                                 unknown      true
192.168.40.59   network     LinkStatus   ip6tnl0        1         tunnel6    ip6tnl      00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00   down         false
192.168.40.59   network     LinkStatus   kube-ipvs0     1         ether      dummy       f6:11:4d:b2:c7:40                                 down         false
192.168.40.59   network     LinkStatus   lo             2         loopback               00:00:00:00:00:00                                 unknown      true
192.168.40.59   network     LinkStatus   siderolink     1         nohdr      wireguard                                                     unknown      false
192.168.40.59   network     LinkStatus   sit0           1         sit        sit         00:00:00:00                                       down         false
192.168.40.59   network     LinkStatus   teql0          1         void                                                                     down         false
192.168.40.59   network     LinkStatus   tunl0          1         ipip       ipip        00:00:00:00                                       down         false
192.168.40.59   network     LinkStatus   veth1e15f174   1         ether      veth        12:43:ef:f7:87:07                                 up           true
192.168.40.59   network     LinkStatus   veth23c899ff   1         ether      veth        f6:b4:9c:12:c5:ea                                 up           true
192.168.40.59   network     LinkStatus   veth5387a8a3   2         ether      veth        a6:05:54:28:9e:01                                 up           true
192.168.40.59   network     LinkStatus   veth57f5a1a5   2         ether      veth        1a:3e:e0:af:a7:7e                                 up           true
192.168.40.59   network     LinkStatus   veth77276e63   2         ether      veth        86:08:55:d1:69:0f                                 up           true
192.168.40.59   network     LinkStatus   vethf2295162   1         ether      veth        b2:d5:a1:5f:e1:d5                                 up           true
```

```bash
NODE            NAMESPACE   TYPE         ID             VERSION   TYPE       KIND        HW ADDR                                           OPER STATE   LINK STATE
192.168.40.10   network     LinkStatus   bond0          1         ether      bond        c6:81:9d:5c:a7:d7                                 down         false
192.168.40.10   network     LinkStatus   cni0           2         ether      bridge      6a:33:d2:c5:03:d8                                 up           true
192.168.40.10   network     LinkStatus   dummy0         1         ether      dummy       e2:94:a8:1f:1c:29                                 down         false
192.168.40.10   network     LinkStatus   enp2s0f0       4         ether                  58:47:ca:77:c5:ae                                 up           true
192.168.40.10   network     LinkStatus   enp2s0f1       4         ether                  58:47:ca:77:c5:af                                 up           true
192.168.40.10   network     LinkStatus   enp87s0        2         ether                  58:47:ca:77:c5:b0                                 down         false
192.168.40.10   network     LinkStatus   enp90s0        2         ether                  58:47:ca:77:c5:b1                                 down         false
192.168.40.10   network     LinkStatus   flannel.1      2         ether      vxlan       6e:9a:98:33:8e:a7                                 unknown      true
192.168.40.10   network     LinkStatus   ip6tnl0        1         tunnel6    ip6tnl      00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00   down         false
192.168.40.10   network     LinkStatus   kube-ipvs0     1         ether      dummy       2e:34:64:66:c5:2f                                 down         false
192.168.40.10   network     LinkStatus   lo             2         loopback               00:00:00:00:00:00                                 unknown      true
192.168.40.10   network     LinkStatus   siderolink     1         nohdr      wireguard                                                     unknown      false
192.168.40.10   network     LinkStatus   sit0           1         sit        sit         00:00:00:00                                       down         false
192.168.40.10   network     LinkStatus   teql0          1         void                                                                     down         false
192.168.40.10   network     LinkStatus   tunl0          1         ipip       ipip        00:00:00:00                                       down         false
192.168.40.10   network     LinkStatus   veth076515d9   2         ether      veth        5a:ce:ae:c5:57:91                                 up           true
192.168.40.10   network     LinkStatus   veth0be6e1e6   2         ether      veth        fa:31:0a:70:ef:9d                                 up           true
192.168.40.10   network     LinkStatus   veth21a13b50   2         ether      veth        f6:2b:2d:fc:a9:c2                                 up           true
192.168.40.10   network     LinkStatus   veth2329fe16   2         ether      veth        1a:7c:f5:d3:ed:cc                                 up           true
192.168.40.10   network     LinkStatus   veth2ee441d3   2         ether      veth        c2:c1:b5:f7:c2:bf                                 up           true
192.168.40.10   network     LinkStatus   veth4eeff87b   2         ether      veth        0a:9a:a7:57:72:17                                 up           true
192.168.40.10   network     LinkStatus   veth50cc865a   2         ether      veth        fe:c2:f6:f7:8d:d6                                 up           true
192.168.40.10   network     LinkStatus   veth7b736aa9   2         ether      veth        d2:10:bc:f0:95:05                                 up           true
192.168.40.10   network     LinkStatus   veth7dd69bf1   2         ether      veth        6e:4e:15:c1:50:37                                 up           true
192.168.40.10   network     LinkStatus   veth80e87825   2         ether      veth        36:2b:b8:9b:96:98                                 up           true
192.168.40.10   network     LinkStatus   veth9bb6850c   1         ether      veth        0e:96:9f:e3:44:7d                                 up           true
192.168.40.10   network     LinkStatus   vethc31b35cf   2         ether      veth        3a:f7:a0:56:55:d5                                 up           true
192.168.40.10   network     LinkStatus   vethe102aff9   2         ether      veth        ea:5d:13:a0:b3:51                                 up           true
192.168.40.10   network     LinkStatus   vethe3805a5d   2         ether      veth        c6:97:b2:46:cb:4d                                 up           true
192.168.40.10   network     LinkStatus   vethe58e3fb3   2         ether      veth        16:d4:c0:f4:d2:02                                 up           true
192.168.40.10   network     LinkStatus   vethe7502b04   2         ether      veth        2a:c2:56:8b:3d:f3                                 up           true
192.168.40.10   network     LinkStatus   vethee60ac5c   2         ether      veth        c2:9e:b9:f4:95:d6                                 up           true
```

### Multus

  Warning  FailedCreatePodSandBox  82s  kubelet  Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "b6e4bdb5f7e1f46bbb4e6b06b694588171a277d0a71ff1588bcafb595a09ebc3": plugin type="multus-shim" name="multus-cni-network" failed (add): CmdAdd (shim): CNI request failed with status 400: 'ContainerID:"b6e4bdb5f7e1f46bbb4e6b06b694588171a277d0a71ff1588bcafb595a09ebc3" Netns:"/var/run/netns/cni-bb91d408-e9c3-3280-8453-07515339cdb6" IfName:"eth0" Args:"IgnoreUnknown=1;K8S_POD_NAMESPACE=rook-ceph;K8S_POD_NAME=multus-validation-test-web-server;K8S_POD_INFRA_CONTAINER_ID=b6e4bdb5f7e1f46bbb4e6b06b694588171a277d0a71ff1588bcafb595a09ebc3;K8S_POD_UID=8f00eb5d-fa65-4215-9389-478c34aa11d6" Path:"" ERRORED: error configuring pod [rook-ceph/multus-validation-test-web-server] networking: Multus: [rook-ceph/multus-validation-test-web-server/8f00eb5d-fa65-4215-9389-478c34aa11d6]: error loading k8s delegates k8s args: TryLoadPodDelegates: error in getting k8s network for pod: GetNetworkDelegates: failed getting the delegate: getKubernetesDelegate: cannot find a network-attachment-definition (multus-public) in namespace (network): networkattachmentdefinition.k8s.cni.cncf.io "multus-public" not found



Then

  Warning  FailedCreatePodSandBox  8s  kubelet  Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "31fb37656aebbe3090817e293fc96e9a979c3393d756403b080ee187250a61ad": plugin type="multus-shim" name="multus-cni-network" failed (add): CmdAdd (shim): CNI request failed with status 400: 'ContainerID:"31fb37656aebbe3090817e293fc96e9a979c3393d756403b080ee187250a61ad" Netns:"/var/run/netns/cni-b472cb35-4ee6-18a8-a574-72420e24e5c0" IfName:"eth0" Args:"K8S_POD_NAMESPACE=rook-ceph;K8S_POD_NAME=multus-validation-test-web-server;K8S_POD_INFRA_CONTAINER_ID=31fb37656aebbe3090817e293fc96e9a979c3393d756403b080ee187250a61ad;K8S_POD_UID=a307c0a2-67e3-4e6e-a6ea-f5e5bef9c304;IgnoreUnknown=1" Path:"" ERRORED: error configuring pod [rook-ceph/multus-validation-test-web-server] networking: Multus: [rook-ceph/multus-validation-test-web-server/a307c0a2-67e3-4e6e-a6ea-f5e5bef9c304]: error loading k8s delegates k8s args: TryLoadPodDelegates: error in getting k8s network for pod: GetNetworkDelegates: failed getting the delegate: GetCNIConfig: err in getCNIConfigFromSpec: failed to unmarshal Spec.Config: invalid character '}' looking for beginning of object key string
': StdinData: {"binDir":"/opt/cni/bin","capabilities":{"portMappings":true},"clusterNetwork":"/host/etc/cni/net.d/10-flannel.conflist","cniVersion":"0.3.1","logLevel":"error","logToStderr":true,"name":"multus-cni-network","type":"multus-shim"}


  Warning  FailedCreatePodSandBox  5s  kubelet  Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "8d659eca1238208ba3099da9c49a0e3da20dfbc7579594d0b46bfd90ad05667f": plugin type="multus-shim" name="multus-cni-network" failed (add): CmdAdd (shim): CNI request failed with status 400: 'ContainerID:"8d659eca1238208ba3099da9c49a0e3da20dfbc7579594d0b46bfd90ad05667f" Netns:"/var/run/netns/cni-c0d05c33-0e72-6935-888a-1ee85109796a" IfName:"eth0" Args:"K8S_POD_NAME=multus-validation-test-web-server;K8S_POD_INFRA_CONTAINER_ID=8d659eca1238208ba3099da9c49a0e3da20dfbc7579594d0b46bfd90ad05667f;K8S_POD_UID=d68ec0f8-1ab3-4f7a-a4f5-71c2f98135fc;IgnoreUnknown=1;K8S_POD_NAMESPACE=rook-ceph" Path:"" ERRORED: error configuring pod [rook-ceph/multus-validation-test-web-server] networking: Multus: [rook-ceph/multus-validation-test-web-server/d68ec0f8-1ab3-4f7a-a4f5-71c2f98135fc]: error loading k8s delegates k8s args: TryLoadPodDelegates: error in getting k8s network for pod: GetNetworkDelegates: failed getting the delegate: GetCNIConfig: err in getCNIConfigFromSpec: failed to unmarshal Spec.Config: invalid character '}' looking for beginning of object key string
': StdinData: {"binDir":"/opt/cni/bin","capabilities":{"portMappings":true},"clusterNetwork":"/host/etc/cni/net.d/10-flannel.conflist","cniVersion":"0.3.1","logLevel":"error","logToStderr":true,"name":"multus-cni-network","type":"multus-shim"}


server/4a20508c-1d01-42e4-b7e2-d8be8657046b:multus-public]: error adding container to network "multus-public": plugin type="macvlan" failed (add): error dialing DHCP daemon: dial unix /run/cni/dhcp.sock: connect: no such file or directory
': StdinData: {"binDir":"/opt/cni/bin","capabilities":{"portMappings":true},"clusterNetwork":"/host/etc/cni/net.d/10-flannel.conflist","cniVersion":"0.3.1","logLevel":"error","logToStderr":true,"name":"multus-cni-network","type":"multus-shim"}