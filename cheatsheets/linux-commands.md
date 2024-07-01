---
title: Linux Cheat Sheet
permalink: /cheatsheets/linux-commands/
---

## Server Management

### CPU Cores

```
nproc --all
```

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

## SMB Share

[This](https://linuxconfig.org/how-to-mount-a-samba-shared-directory-at-boot) was a nice writeup of parts of what is below.

### Dependency 

Install `cifs` which will mount the share:

```
sudo apt-get install cifs-utils
```

> If you do not have `cifs` installed you will get a "No route to host" error.

Create the directory you want the share to be mounted to:

```
sudo mkdir /mnt/nas01-data
```

> If required make a `.credentials` file w/ `username` and `password` and `sudo chmod 600 .credentials`

### Mount

#### Single Instance

```
sudo mount -v -t cifs -o credentials=/home/thaynes/.credentials,uid=1000,gid=1000 //nas01/Data /mnt/nas01-data
```

Sucess will show this because we included `-v`:

```
mount.cifs kernel mount options: ip=192.168.0.10,unc=\\nas01\Data,user=thaynes,pass=********
```

Then you can see what you mounted via `cd /mnt/nas01-data`.

#### On Startup

For mounting on startup we need to edit fstab via `sudo nano /etc/fstab/`. The example above may be added via:

```
//nas01/Data /mnt/nas01-data cifs credentials=/home/thaynes/.credentials,uid=1000,gid=1000 0 0
```

Reboot to verify the mount is recreated on startup or `umount` anything previously mounted and run `sudo mount -a`.

### Umount

Unmount but without the `n` because I guess that makes it too long to bare:

```
sudo umount /mnt/nas01-data
```

There is a `-f` option but I have not needed to use it.