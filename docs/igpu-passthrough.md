---
title: GPU Passthrough w/ Intel Iris Xe
permalink: /docs/igpu-passthrough/
---

In order to do this I now had to embark on [these steps](https://www.reddit.com/r/Proxmox/comments/14fzj8l/tutorial_full_igpu_passthrough_for_jellyfin/) to pass through my iGPU. 

First he has you add some crazy shit to `/etc/default/grub`

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction initcall_blacklist=sysfb_init video=simplefb:off video=vesafb:off video=efifb:off video=vesa:off disable_vga=1 vfio_iommu_type1.allow_unsafe_interrupts=1 kvm.ignore_msrs=1 modprobe.blacklist=radeon,nouveau,nvidia,nvidiafb,nvidia-gpu,snd_hda_intel,snd_hda_codec_hdmi,i915
```
A lot of this looks like things I added to other files for the RX 6400. 

Same with these:

```
nano /etc/modules

vfio
vfio_pci
vfio_virqfd
vfio_iommu_type1

update-initramfs -u -k all
reboot
```

Once back up we can add the iGPU to the VM via `the VM` -> `Hardware` -> `Add` -> `PCI Device` and add:

![add igpu dropdown]({{ site.url }}/images/proxmox/add-igpu-dropdown.png)

![add igpu pcie]({{ site.url }}/images/proxmox/add-igpu-pcie.png)

And also set `Hardware` -> `Display` -> `none` as we did with the RX 6400 Windows VM.

> **NOTE:** Since we already installed nomachine and Parsec plus all the updates and VirtIO drivers we should be fine but if you didn't you'd need a fake display to do that first.

And VICTORYYYYYYYYYYYY! 

Sort of, the window is tiny:

![igpu tiny window]({{ site.url }}/images/windows/igpu-tiny-window.png)

But opening it in Parsec somehow snapped it out of a frozen state it went into when I tried Display Settings and switched it to 1920x1080. Now I'm flush with resolutions:

![igpu resolutions]({{ site.url }}/images/windows/igpu-resolutions.png)

Closing Parsec messed up nomachine so there's something going on there that I don't understand. Before I continue I'm going to see if passing the host CPU helps with anything:

In `pve04:/etc/pve/qemu-server/114.conf`:
* Change `machine: pc-q35-8.1` to `machine: q35`
* Change `cpu: x86-64-v2-AES` to `cpu: host,hidden=1,flags=+pcid`
* Add `args: -cpu 'host,+kvm_pv_unhalt,+kvm_pv_eoi,hv_vendor_id=NV43FIX,kvm=off'`

Threw some errors:

```
vm 114 - unable to parse value of 'machine' - format error
type: value does not match the regex pattern
TASK ERROR: q35 machine model is not enabled at /usr/share/perl5/PVE/QemuServer/PCI.pm line 486.
```

My config said q34... so retrying. For some reason I need to log in with nomachine on the tiny VM to get Parsec to detect the VM and allow me to boost the resolution. But once I was in I was greeted with a happy site:

![igpu software]({{ site.url }}/images/windows/igpu-software.png)

Since windows installed something it obviously wanted to reboot so I figured I'd see if I'd get hit with tiny nomachine again and sure enough it was super small. 

Minor problem, Device Manager is hitting a Code 43 for the iGPU!

```
Windows has stopped this device because it has reported problems. (Code 43)
```

May have something to do with [SR-IOV](https://forum.proxmox.com/threads/enabling-sr-iov-for-intel-nic-x550-t2-on-proxmox-6.56677/)

Another [guide](https://www.michaelstinkerings.org/gpu-virtualization-with-intel-12th-gen-igpu-uhd-730/) for later!

And even better, one for the [MS-01!](https://forum.level1techs.com/t/i915-sr-iov-on-i9-13900h-minisforum-ms-01-proxmox-pve-kernel-6-5-jellyfin-full-hardware-accelerated-lxc/209943)