---
title: Infrastructure Cheat Sheet
permalink: /cheatsheets/infra-commands/
---

## Proxmox

### Reboot

```
systemctl reboot
```

### Reload Network Stuff

```
ifreload -a
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

