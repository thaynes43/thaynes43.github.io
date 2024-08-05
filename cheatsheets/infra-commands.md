---
title: Infrastructure Cheat Sheet
permalink: /cheatsheets/infra-commands/
---

## Proxmox

### Update

```
pveam update
```

### Remove Node

```
pvecm delnode <node>
rm -r /etc/pve/nodes/<node>
```

Not sure why the folder needs to be deleted but you can refresh the web browser after and it'll be gone! 

### Logs

#### Journal

```
journalctl
```

### Reboot

#### Trigger Reboot

```
systemctl reboot
```

#### See Last Reboot

See [comment](https://unix.stackexchange.com/questions/9819/how-to-find-out-from-the-logs-what-caused-system-shutdown)

```
last -x | head | tac
```

### Reload Network Stuff

```
ifreload -a
```

### After Editing Grub File

After you edit `/etc/default/grub` you run the following and then reboot.

#### For Regular Installs

```
update-grub
```

#### For ZFS Installs

```
pve-efiboot-tool refresh
```

## USB

### See Drives

```
lsblk

or

fdisk -l
```

### Map one to a VM

Note sata1 here is the second sata drive to this VM. Can also use `virtio1` or something.

```
sata1: /dev/sda2,size=953852M
```

## ZFS

### See Issues

```
zpool status -v
```

### Clear Warnings

```
zpool clear <poolname> <devicename>

zpool clear tank ata-ST20000NM004E-3HR103_ZX2119KV
```

## Ceph

### Archive crash warnings

These happen when I reboot.

```
ceph crash archive-all
```

### Cluster info

#### Status

Overall status of the cluster.

```
ceph status || ceph -w
```

#### Monitors

Get details about the monitors.

```
ceph mon dump
```

