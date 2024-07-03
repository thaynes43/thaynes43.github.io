---
title: Gaming VM Guide 
permalink: /docs/gaming-vm-guide/
---

Most of the documents on here include missteps and mess ups but since I've already screwed the pooch on this one over in the [haynesintelligence]({{ site.url }}/hardware/haynesintelligence/) I'm going to try to create a guide for me to create a VM from scratch.

## Configuring the VM

The key here is to create a VM Windows and Anti Cheats like EasyAntiCheat don't know is a VM. Once problem I ran into before was I couldn't figure out how to trick EasyAntiCheat into forgetting my VM was a VM so I had to start over.

Start off like any VM but DO NOT START AT BOOT! Do not add VirtIO drivers either, those are a dead give away.

### Fake NIC

FAKE Intel NIC MAC with prefix `00:AA:02` using [online generator](https://dnschecker.org/mac-address-generator.php). Then configure an E1000 adapter to use it:

```
net0: e1000=00:AA:02:6A:A4:D3,bridge=vmbr0,firewall=1
```

### Hide VM Stuff

```
args: -cpu host,-hypervisor,kvm=off
scsihw: lsi
sata0: local-zfs:vm-119-disk-1,cache=writeback,size=512G
```

### BIOS Info

Take the output of `dmidecode --type 1` and fill in the BIOS settings:

```
System Information
        Manufacturer: Micro Computer (HK) Tech Limited
        Product Name: Venus Series
        Version: Default string
        Serial Number: MF216VS139QQMQA00239
        UUID: c4b82f80-18bf-11ef-93bf-c7287d1d5f00
        Wake-up Type: Power Switch
        SKU Number: Default string
        Family: Default string
```

### Initial Config

The VM is now ready for it's first boot. My config looks like the following. Note we will change a few things after we do first time setup.

```
args: -cpu host,-hypervisor,kvm=off
balloon: 0
bios: ovmf
boot: order=sata0;ide2;net0
cores: 12
cpu: host
efidisk0: vm-disks:vm-122-disk-0,efitype=4m,pre-enrolled-keys=1,size=1M
ide2: ISO-Templates:iso/en-us_windows_11_business_editions_updated_april_2022_x64_dvd_ce217593.iso,media=cdrom,size=5460642K
machine: pc-q35-9.0
memory: 32768
meta: creation-qemu=9.0.0,ctime=1719940149
name: WinGaming03
net0: e1000=00:AA:02:6A:A4:D3,bridge=vmbr0,firewall=1
numa: 0
ostype: win11
sata0: vm-disks:vm-122-disk-1,cache=writeback,size=512G
scsihw: lsi
smbios1: uuid=c4b82f80-18bf-11ef-93bf-c7287d1d5f00,manufacturer=TWljcm8gQ29tcHV0ZXIgKEhLKSBUZWNoIExpbWl0ZWQ=,product=VmVudXMgU2VyaWVz,version=RGVmYXVsdCBzdHJpbmc=,serial=TUYyMTZWUzEzOVFRTVFBMDAyMzk=,sku=RGVmYXVsdCBzdHJpbmc=,family=RGVmYXVsdCBzdHJpbmc=,base64=1
sockets: 1
tpmstate0: vm-disks:vm-122-disk-2,size=4M,version=v2.0
vmgenid: 7659a33f-749b-4abd-91fb-fe004841e6ec
```

## First Time Windows Setup

We can now boot. Everything should be standard except this time the install is online and we don't need VirtIO drivers later to get internet access. This adds a few steps to the installation process:

1. The installer will immediately download updates.
2. If you want a local account instead of a microsoft account you need to jump through a few hoops. Since I'm using gamepass I don't really mind the microsoft account. 
3. It will default to restoring from a device. I've never done this, no idea why you would unless you were replacing something that broke. 
4. They try to sell you so much shit, just skip it all. 
5. It will download even more updates, and updates for the updates.

> **NOTE** Enable location services so your timezone / sync is correct or some games will freak out

### Verify Windows is Tricked

Once you are in open up `Task Manager` -> `Performance` and make sure Virtualization is enabled meaning you can host VMs on this machine:

![not a vm]({{ site.url }}/images/builds/windows/not-a-vm.png)

If things were not set up correctly Windows will say "Virtual machine: Yes" instead:

![windows sees vm]({{ site.url }}/images/builds/windows/windows-sees-vm.png)

Now that we have a secret VM we are good to move on to passing through what we need for gaming.

## Passthrough Settings

The obvious thing to passthrough here is the GPU but I am also going to test this locally w/ a KB&M and Xbox controller via a bluetooth dongle. 

Now shut it down and 'eject' the installation media.

### Passthrough Settings

The next steps are highly dependent on your hardware and can be done through the UI. Here is where I landed for playing in my basement on some junk hardware:

```
hostpci0: 0000:03:00,pcie=1,x-vga=1  # GPU
hostpci1: 0000:00:1f.3               # Audio 
usb0: host=1-3,usb3=1                # Keyboard
usb1: host=1-8,usb3=1                # Mouse
usb2: host=1-9,usb3=1                # BT USB Dongle
```

I'm using a Bluetooth dongle which kinda sucks. It's an [ASUS BT500](https://www.asus.com/us/networking-iot-servers/adapters/all-series/usb-bt500/) that I bought for my Home Assistant VM but wasn't really using. For some reason the controller will periodically disconnect, the xbox light will blink, then it'll reconnect. I've tried two different dongles on two different Proxmox hosts ([haynesintelligence]({{ site.url }}/hardware/haynesintelligence/) has it's own one for the second gaming VM). I've also used onboard WiFi and BT that came with one of those massive antennas w/ the two screw in connectors and that worked flawlessly so I'd recommend that over this one. 

For onboard you just need to passthrough a PCIe and USB device to get the whole WiFi BT combo. Half and half won't work:

![onboard bt pcie]({{ site.url }}/images/builds/proxmox/onboard-bt-pcie.png)

![onboard bt usb]({{ site.url }}/images/builds/proxmox/onboard-bt-usb.png)

