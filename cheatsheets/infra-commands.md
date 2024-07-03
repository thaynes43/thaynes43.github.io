---
title: Infrastructure Cheat Sheet
permalink: /cheatsheets/infra-commands/
---

## Proxmox

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

## ZFS

### Clear Warnings

```
zpool clear <poolname>
```

## ceph

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

