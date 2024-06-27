---
title: Linux Cheat Sheet
permalink: /cheatsheets/linux-commands/
---

## Server Management

### Disk Space

```
df -h
```

### Reboot

```
sudo shutdown -r now
```

### Halt  

```
sudo shutdown -h now
```

## Package Management

## Updating packages from sources

```
sudo apt update
```

### Upgrading what is installed

```
sudo apt upgrade
```

### Packages

#### Direct from deb file

```
sudo dpkg install -i <file.deb>
```

If you get complaints about packages missing just do:

```
sudo apt --fix-broken install
```

Or you can remove it


#### Install helpers from package repos 

These are both very similar. `apt` seems more feature rich and can do stuff like `apt search`, `apt show`, `apt list option`, `apt edit-sources` to give more control over the packages.

`apt-get` is basically a simplified interface to `dkpg` for packages from your sources.

```
apt install <package>
```

```
apt-get intall <package>
```

#### Uninstall

##### dpkg

```
dpkg -r <package>
```

Or to delete config too:

```
dpkg -P <package>
```

##### apt and apt-get

Same `remove` command for both here:

```
sudo apt remove <what you installed>
sudo apt-get remove <what you installed>
```

## SCP

### Copy from Local to Remote

```
scp file.txt remote_username@192.168.0.100:/remote/directory
```

### Copy From Remote to Local

```
scp remote_username@10.10.0.2:/remote/file.txt /local/directory
```