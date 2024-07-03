---
title: Disk Speed Benchmarks
permalink: /benchmarks/disk-speed/
---

I realized I installed a gaming VM, `WinGaming03`, on top of Ceph so I decided to see how much I'd be paying for that mistake. Fortunately, I had some old baselines to compare against.

## Existing Bare Metal Benchmarks

[CrystalDiskMark](https://crystalmark.info/en/software/crystaldiskmark/) is a simple tool that lets you benchmark whatever drives you have on windows. I had previously used this for a whole bunch of drives from older builds.

### Old PC

Before Hyperion I must have had a bunch of disks flopping around because I ended up with a bunch of benchmarks:

![many disks speed]({{ site.url }}/images/windows/many-disks-speed.png)

### Hyperion

I cleaned it up a bit to benchmark the drives in Hyperion:

![hyperion disk speed]({{ site.url }}/images/windows/hyperion-disk-speed.png)

## New VM Benchmarks

Now to see how the VM compared.

### Ceph RBD

This mistake proved to be costly:

![ceph disk speed]({{ site.url }}/images/windows/ceph-disk-speed.png)

It beat out the HDD and old 2.5" SSD I had benchmarks for but wasn't even close to a gen 4 M.2. This made me curious how the gaming VMs on [HaynesIntelligence]({{ site.url }}/hardware/haynesintelligence/) would fair. 

### 990 Pro Raid 0 Array

[HaynesIntelligence]({{ site.url }}/hardware/haynesintelligence/) has two 2TB Samsung 990 Pros in a Raid 0 array which is risky business but theoretically blazing fast. 

However, `WinGaming01` was not fast at all:

![wingaming01 disk speed]({{ site.url }}/images/windows/wingaming01-disk-speed.png)

Oddly, `WinGaming02` was faster but this was after a reboot and it was the only VM running other than the Samba stuff:

![wingaming02 disk speed]({{ site.url }}/images/windows/wingaming02-disk-speed.png)

Since it didn't make much sense I wanted to give `WinGaming01` another shot. Of course I stayed up to late and now `PBS` was backing it up... 

![wingaming01 backup]({{ site.url }}/images/windows/wingaming01-backup.png)

Maybe another time.

### Back to the Costly Mistake

Fortunately it was quite easy to move disks from my ceph rbd to the local SSD that Proxmox is installed on. Since `WinGaming03` needed the GPU on the node anyway for it to be any use I wasn't going to be migrating it anywhere so this was fine with me. After this it prove to be the fastest VM which wasn't bad though defiantly a lot of speed lost compared to what I get with my similar Gen 4 Crucial M.2 on my desktop.

![wingaming03 disk speed]({{ site.url }}/images/windows/wingaming03-disk-speed.png)



