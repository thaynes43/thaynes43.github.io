---
title: GPU Passthrough w/ Radeon RX 6400
permalink: /docs/gpu-passthrough/
---

## Installing the GPU 

Ordered on [Amazon](https://www.amazon.com/dp/B09Y7358KJ?ref=ppx_yo2ov_dt_b_product_details&th=1) for my hackintosh since it was cheap and seemed like it wasn't going to be a risky fit in the MS-01.

### The Easy Part

I decided to install an Radeon RX 6400 in one of the nodes to get a feel for how it'd perform, fit, and how hot it'd run. TODO update the hardware doc with the installation and link from here.

### The Nightmare

Tne MAJOR thing I missed is the network adapters my proxmox node was relying on would have their names change after installing the GPU. This drove me crazy for a bit as I was looking for an MS-01 specific issue. Once I searched for "PROXMOX BROKE AFTER I DID A GPU THING" it was pretty clear a lot of folks didn't realize this either...

To fix fine the new adapters:

```
ip link show

ip a
```

Then update `/etc/network/interfaces` to match the new names. For me I had to change a `2` to a `5` for my two SPF+ adapters.

## Passing it Through

### Configuring Proxmox to use the GPU

Next I had to configure proxmox so I could get at the GPU:

```
nano /etc/default/grub
Add intel_iommu=on like this GRUB_CMDLINE_LINUX_DEFAULT=”quiet intel_iommu=on”
update-grub 
reboot
```

Still didn't show up so I turned to [official documentation](https://pve.proxmox.com/wiki/PCI_Passthrough) which lead me to this command:

```
pvesh get /nodes/{nodename}/hardware/pci --pci-class-blacklist ""
```

Which gave me some relief since it was right in the list:

```
│ class    │ device │ id           │ iommugroup │ vendor │ device_name                                          │ mdev │ subsystem_device │ subsystem_device_name       
│ 0x030000 │ 0x743f │ 0000:03:00.0 │         18 │ 0x1002 │ Navi 24 [Radeon RX 6400/6500 XT/6500M]               │      │ 0x6405           │                             
```

Switching gears to [this guide](https://www.reddit.com/r/homelab/comments/b5xpua/the_ultimate_beginners_guide_to_gpu_passthrough/) I added the modules:

#### Passthrough Modules

```
nano /etc/modules
```

Paste in:

```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

Then some wild west shit:

```
echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf
echo "options kvm ignore_msrs=1" > /etc/modprobe.d/kvm.conf
```

#### Driver Blacklist

Then blacklist drivers so proxmox doesn't mess with the GPU:

```
echo "blacklist amdgpu" >> /etc/modprobe.d/blacklist.conf
echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf
```

Others seem to be from nvidia

```
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nvidia*" >> /etc/modprobe.d/blacklist.conf
```

Or intel:

```
echo "blacklist i915" >> /etc/modprobe.d/blacklist.conf
```

#### Add to VFIO

Then we add it to VFIO:

```
root@pve05:~# lspci -v | grep AMD
```

Yields:

```
03:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Navi 24 [Radeon RX 6400/6500 XT/6500M] (rev c7) (prog-if 00 ;
...
03:00.1 Audio device: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21/23 HDMI/DP Audio Controller
```

Some more voodoo:

```
root@pve05:~# lspci -n -s 03:00
03:00.0 0300: 1002:743f (rev c7)
03:00.1 0403: 1002:ab28
```

GPU Vendor Code: 1002:743f
Audio Bus Code: 1002:ab28

Now add the vendor codes to VFIO:

```
echo "options vfio-pci ids=1002:743f,1002:ab28 disable_vga=1"> /etc/modprobe.d/vfio.conf
```

Then update some shit:

```
update-initramfs -u
```

I think it needs a reboot after. The guide says to run `reset` which just clears the cmd.

### Adding to MacOS VM

Seems too easy but this is what I came up with:

![amd gpu dry run]({{ site.url }}/images/proxmox/amd-gpu-dry-run.png)

The hackintosh lost it's shit at first but rebooted just fine after. You can tell adding the GPU has stressed it out. 

No dice, going to see how [the master](https://www.nicksherlock.com/2018/11/my-macos-vm-proxmox-setup/) did it.

`nano /etc/pve/qemu-server/112.conf`

```
hostpci0: 03:00,pcie=1,x-vga=on
```

Mine was missing `x-vga=on` and I needed `vga: none` instead of `vmware`.

Note I many need `Host configuration` later for the two GPU server...

Back to `/etc/default/grub` to add `rootdelay=10`

And some other stuff from the guide:

```
nano /etc/modprobe.d/kvm-intel.conf
# Nested VM support
options kvm-intel nested=1
```

```
nano /etc/modprobe.d/vfio-pci.conf

options vfio-pci ids=1002:743f,1002:ab28 disable_vga=1
# Note that adding disable_vga here will probably prevent guests from booting in SeaBIOS mode
```

Then:

```
update-grub
update-initramfs -k all -u
reboot
```

Still didn't work. 

### PIVOT to Windows

[The reddit guide](https://www.reddit.com/r/homelab/comments/b5xpua/the_ultimate_beginners_guide_to_gpu_passthrough/) walks through on windows and I can manage that a bit better to get my bearings.

For installing the VM I loosely followed [these steps](https://www.wundertech.net/how-to-install-windows-11-on-proxmox/) but the reddit guide (above) suggested I change a few things:

* Change `machine: pc-q35-8.1` to `machine: q35`
* Change `cpu: x86-64-v2-AES` to `cpu: host,hidden=1,flags=+pcid`
* Add `args: -cpu 'host,+kvm_pv_unhalt,+kvm_pv_eoi,hv_vendor_id=NV43FIX,kvm=off'`

After going through the motions of installing Windows offline I was in. I set up nomachine and RDP so I could disable the virtual display in proxmox via `Hardware` -> `Display` -> `None`. After doing so nomachine no longer worked so I assume this relies on a display. RDP, however, not only worked but AMD's driver software picked up the card no problem:

![amd detected]({{ site.url }}/images/windows/amd-detected.png)

Even better, after installing the drivers `nomachine` worked again! And unlike RDP I wasn't constrained to an upscaled resolution, I could crank this baby up to 4k!

![namachine 4k]({{ site.url }}/images/windows/namachine-4k.png)

However, this might be thanks to the [dummy plug](https://www.amazon.com/gp/product/B06XSZR7CG/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1) I stuck in when troubleshooting the macos VM.

Before going any further I activated windows using my a key from what use to be called MSDN. I then had a few years of updates to install.

#### Gaming

Since that was taking forever I decided to give a lightweight offline game a try. I use `nomachine` for everything and this is about the most responsive I've seen it. 

![hades vm]({{ site.url }}/images/windows/hades-vm.png)

Now that's just a 2D game so I wanted to really push it. For the final boss I chose...

![final boss]({{ site.url }}/images/windows/final-boss.png)

TODO do [this](https://pve.proxmox.com/wiki/Qemu-guest-agent) for windows 

Well the final boss was playable but even on high settings I seemed to be missing textures:

![final boss goopy]({{ site.url }}/images/windows/final-boss-goopy.png)

Enough distractions, time to go install some AMD drivers on a macos VM.

#### Benchmarks

After all this passing through I experimented a bit with a game. See the results over [here]({{ site.url }}/benchmarks/hades/).

### Back to Mac! 

It clicked when I went back. The mac VM isn't booting because of the damn screen to select a disk. It was time to go back to `config.plist`. 

Since the damn thing moves around and needs to be mounted every time I went ahead and followed these steps:

```
diskutil list
# Pick the EFI one
sudo mkdir /Volumes/EFI
sudo mount -t msdos /dev/disk1s1 /Volumes/EFI
```

A million XML elements away you'll come to `Misc/Boot/Timeout` and then set the row below Timeout to a non zero value (I did 3 seconds). Now reboot and count to 3 and BAM we are in.

Now we can go back and turn off the display again!

This time I was able to ssh so at least I got past that:

```
manof@HaynesHyperion MINGW64 ~
$ ssh thaynes@mac02.haynesnetwork
(thaynes@mac02.haynesnetwork) Password:
Last login: Tue Jun 11 00:46:25 2024
thaynes@mac02 ~ %
```

However, now nomachine can't find a desktop (what desktop!)

![no desktop]({{ site.url }}/images/mac/no-desktop.png)

Going back to my VM config I'll poke at a few more settings:

```
args: -device isa-applesmc,osk="ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc" -smbios type=2 -device usb-kbd,bus=ehci.0,port=2 -global nec-usb-xhci.msi=off -global ICH9-LPC.acpi-pci-hotplug-with-bridge-support=off -cpu host,vendor=GenuineIntel,+invtsc,+hypervisor,kvm=on,vmware-cpuid-freq=on
balloon: 0
bios: ovmf
boot: order=virtio0;net0
cores: 8
cpu: x86-64-v2-AES
efidisk0: vm-disks:vm-112-disk-0,efitype=4m,pre-enrolled-keys=1,size=1M
hostpci0: 0000:03:00,pcie=1,x-vga=1
ide0: ISO-Templates:iso/macOS-Sonoma-14.1.1.iso,cache=unsafe,size=16000M
machine: q35
memory: 20480
meta: creation-qemu=8.1.5,ctime=1717904063
name: mac02
net0: vmxnet3=BC:24:11:A0:69:A4,bridge=vmbr0,firewall=1
numa: 0
ostype: other
scsihw: virtio-scsi-pci
smbios1: uuid=931fc13d-1caa-45e3-b042-472a907eb4fc
sockets: 1
vga: std,memory=512
virtio0: vm-disks:vm-112-disk-1,cache=unsafe,discard=on,iothread=1,size=256G
vmgenid: 89085a0f-8b1a-4de2-976e-edd86c9b0d09
```

I changed `cpu` to `cpu: host,hidden=1,flags=+pcid`
Add `args: -cpu '...+kvm_pv_unhalt,+kvm_pv_eoi,kvm=off'` (`kvm=on` was set before)

Here's what Nick man had:

```
args: -device isa-applesmc,osk="..." -smbios type=2 cpu host,kvm=on,vendor=GenuineIntel,+kvm_pv_unhalt,+kvm_pv_eoi,+hypervisor,+invtsc,+pdpe1gb,check -smp 32,sockets=2,cores=8,threads=2 -device 'pcie-root-port,id=ich9-pcie-port-6,addr=10.1,x-speed=16,x-width=32,multifunction=on,bus=pcie.0,port=6,chassis=6' -device 'vfio-pci,host=0000:0a:00.0,id=hostpci5,bus=ich9-pcie-port-6,addr=0x0,x-pci-device-id=0x10f6,x-pci-sub-vendor-id=0x0000,x-pci-sub-device-id=0x0000' -global ICH9-LPC.acpi-pci-hotplug-with-bridge-support=off
```

But still no dice. My main suspicion was that I've been following guides for people who plug their peripherals right into the devices passed the the VM. But then I found [this](https://dortania.github.io/GPU-Buyers-Guide/modern-gpus/amd-gpu.html) and I learned that the RX 6400 didn't cut the mustard. I did what any reasonable human being would do and bought an RX 6800 but that'll have to wait for the MEGA AI SERVER.

## Ubuntu Passthrough

### Figuring it Out

Not as many guides exists for this as the general "pass through shit" ones but [this](https://manjaro.site/tips-to-create-ubuntu-20-04-vm-on-proxmox-with-gpu-passthrough/) looks sufficiently dangerous. My VM is was missing a good chunk of what this had though.

1. Machine was `Dafault` instead of `q35`
2. BIOS was `Default (SeaBIOS)` instead of `OVMF` and because of this I had no `EFI Disk`

I set that all up and added the GPU as a PCIE device. The VM instantly migrated to a different node which made 0 sense. It then wouldn't migrate back because it had a local resource attached that couldn't be found. Nothing is ever easy.

To rule out the two differences I decided to go without the GPU first. It still migrated and wouldn't start, but on the host it migrated to is started. Then it migrated back and started again. Maybe I'll remove this one from HA for now...

This reminded me of another VM I tried to switch the BIOS type on after creation. I ended up nuking that one, not sure how to get the `EFI Disk` working after the fact. Maybe `SeaBIOS` is fine.

Somehow the display for VNC was "Portrait (Right)" which I think was my fault from an earlier experiment. After much inverted mouse navigation I got it back and shut down to add the GPU.

This program shows some stuff about the computer but nothing for the GPU:

```
sudo lshw-gtk
```

This works:

```
lspci  -v -s  $(lspci | grep ' VGA ' | cut -d" " -f 1)
```

Not there yet, gonna see if I can switch the BIOS. I guess here the trick is to create a new VM and then us it's `EIF Disk`but maybe if I am clever I can clone it.

The new disk was `efidisk0: vm-disks:vm-115-disk-0,efitype=4m,pre-enrolled-keys=1,size=1M`

And the old was `efidisk0: vm-disks:vm-110-disk-2,efitype=4m,pre-enrolled-keys=1,size=1M`

So I just needed to swap the.. but it didn't work so I guess onto the fresh install for now!

### Trying Again

After much fiddling with the new VM I landed with these settings:

![ubuntu gpu]({{ site.url }}/images/proxmox/ubuntu-gpu.png)

And boom we got a GPU:

![gpu passed through]({{ site.url }}/images/ubuntu/gpu-passed-through.png)

However something seemed to break in the fiddling so I may need one more...

## RX 570 

I did some dirty deeds to connect this to the MS-01 which I will post later. Once it was running I was able to map it as a PCIe passthrough for both Widows 11 and MacOS. I didn't seem to need a rom file despite not being able to set the iGPU as the primary boot option in BIOS with it installed as with it installed BIOS was just a blank screen. 

For windows I could get output from the VM direct from the GPU and pass through the USB ports to use it just like a baremetal install. However, I'd get `Error 43` in Device Manager so something wasn't right. 

For mac I could get the GPU through but the monitor would get stuck on the loading bar while nomachine let me log in and see that it was passed through: 

![rx570 details]({{ site.url }}/images/mac/rx570-details.png)

### Adding things the iGPU steps didn't

This node was setup for IOMMU for the iGPU so I was going to try and edit some stuff to see if I could resolve these problems:

Taken from above:

```
root@pve04:~# echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf
root@pve04:~# echo "options kvm ignore_msrs=1" > /etc/modprobe.d/kvm.conf
root@pve04:~# echo "blacklist amdgpu" >> /etc/modprobe.d/blacklist.conf
root@pve04:~# echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf
root@pve04:~# echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
root@pve04:~# echo "blacklist nvidia*" >> /etc/modprobe.d/blacklist.conf
root@pve04:~# echo "blacklist i915" >> /etc/modprobe.d/blacklist.conf
```

This file was missing:

```
nano /etc/modprobe.d/kvm-intel.conf
# Nested VM support
options kvm-intel nested=1
```

Then `/etc/modprobe.d/vfio-pci.conf` needed the codes..

```
root@pve04:~# lspci -n -s 01:00
01:00.0 0300: 1002:67df (rev ef)
01:00.1 0403: 1002:aaf0
```

Then bring em in 

```
nano /etc/modprobe.d/vfio-pci.conf

options vfio-pci ids=1002:67df,1002:aaf0 disable_vga=1
# Note that adding disable_vga here will probably prevent guests from booting in SeaBIOS mode
```

Then apply all those changes:

```
update-grub
update-initramfs -k all -u
reboot
```

I think the iGPU stuff is conflicting so I'm going to simplify GRUB

`nano /etc/default/grub`

First going back from:

```
GRUB_DEFAULT="gnulinux-advanced-8b2eb4ef-3076-4676-9210-df8d2ae81f5e>gnulinux-6.5.13-3-pve-advanced-8b2eb4ef-3076-4676-9210-df8d2ae81f5e"
```

To:

```
GRUB_DEFAULT=0
```

But I will leave the iGPU stuff commented out if I want to use it again. Then from:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=7 rootdelay=10"
```

To:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on rootdelay=10"
```

Then apply the changes:

```
update-grub
shutdown -r now
```

> **WARNING** every time the kernal changes the network interfaces get messed up

## RX 6800

Ended up hooking this to the MS-01...

### Configuring 

Find PCI Device:

```
lspci  -v -s  $(lspci | grep ' VGA ' | cut -d" " -f 1)
```

Find IDs for `vfio-pci.conf`

```
root@pve04:~# lspci -n -s 03:00
03:00.0 0300: 1002:73bf (rev c3)
03:00.1 0403: 1002:ab28

nano /etc/modprobe.d/vfio-pci.conf

options vfio-pci ids=1002:73bf,1002:ab28 disable_vga=1
# Note that adding disable_vga here will probably prevent guests from booting in SeaBIOS mode
```

Then apply changes (no idea which is really needed)

```
update-grub
update-initramfs -k all -u
reboot
```

### Reset Bug

Trying this too:

```
apt install pve-headers-$(uname -r)
apt install git dkms build-essential
git clone https://github.com/gnif/vendor-reset.git
cd vendor-reset
dkms install .
echo "vendor-reset" >> /etc/modules
update-initramfs -u
shutdown -r now
```

> [This guy](https://github.com/gnif/vendor-reset/issues/46#issuecomment-1412023286) says to put `vendor-reset` at the top of the modules!

> And [This guy](https://github.com/gnif/vendor-reset/issues/46#issuecomment-1295482826) says you need a script and the first guy agrees! 

Need to run this command:

```
echo 'device_specific' > /sys/bus/pci/devices/0000\:03\:00.0/reset_method
```

1. Add to script here `/var/lib/vz/gpu-vendor-reset-method.sh`
2. Add the codeblock below to `/etc/systemd/system/gpu-vendor-reset-method.service`

```
[Unit]
Description=Set the AMD GPU reset method to 'device_specific'
After=default.target

[Service]
ExecStart=/bin/bash /var/lib/vz/gpu-vendor-reset-method.sh

[Install]
WantedBy=default.target
```

3. Activate it with `systemctl enable gpu-vendor-reset-method.service`
4. Reboot
5. Check `systemctl status gpu-vendor-reset-method.service`

Check with `dmesg | grep vendor_reset`

### Wrong Kernal Making Life HArd

> Minor problem noticed rebooting - it's stil booting into the wrong Kernal and my interfaces get out of wack
>
> I want:
> Linux 6.8.4-3-pve (2024-05-02T11:55Z)
> whish should be the 0 index but it's not working: grep menu /boot/grub/grub.cfg
> WIll try setting `GRUB_DEFAULT="gnulinux-advanced-8b2eb4ef-3076-4676-9210-df8d2ae81f5e>gnulinux-6.8.4-3-pve-advanced-8b2eb4ef-3076-4676-9210-df8d2ae81f5e"` in `/etc/default/grub` instead.
> Didn't work
> I will try `proxmox-boot-tool kernel pin 6.8.4-3-pve`

I think pin was the ticket as I got:

```
root@pve04:~# proxmox-boot-tool kernel pin 6.8.4-3-pve
Overriding previously pinned version '6.5.13-3-pve' with '6.8.4-3-pve'
Setting '6.8.4-3-pve' as grub default entry and running update-grub.
```

Now I will move along and see if I can fix the reset bug:

[reset bug](https://forum.proxmox.com/threads/pci-gpu-passthrough-on-proxmox-ve-8-installation-and-configuration.130218/)

Check with `dmesg | grep vendor_reset` -> works on latest kernal

I think this is triggered when I fiddle with the drivers on windows, better safe than sorry.


### AMD Adrenaline DRV64 Error 1723

After setting the RX 6800 as the primary GPU is passes through to Windows no problem but AMD drivers are not great. 

Bunch of errors pop up and installation fails with "DRV64 Error 1723". Googling says to clean out the AMD drivers with `ccleaner` and try clean. However, after rebooting on the uninstall I'm back to Error 43. Could be the reset shit, worth a shot re-installing that given on the Kernal nonsense and rebooting.


Worst case here's a [mega hack](https://www.reddit.com/r/Proxmox/comments/v4yagb/comment/ib7wt7t/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button) to try...

```
/root/fix_gpu_pass.sh

#!/bin/bash
echo 1 > /sys/bus/pci/devices/0000\:01\:00.0/remove
echo 1 > /sys/bus/pci/rescan
```

Then create a cron job w/ the script:

```
@reboot  /root/fix_gpu_pass.sh
```

But gonna go back to the old one and save this for the weekend.

## RX 570

root@pve04:~# lspci -n -s 01:00
01:00.0 0300: 1002:67df (rev ef)
01:00.1 0403: 1002:aaf0

nano /etc/modprobe.d/vfio-pci.conf

options vfio-pci ids=1002:67df,1002:aaf0 disable_vga=1
# Note that adding disable_vga here will probably prevent guests from booting in SeaBIOS mode