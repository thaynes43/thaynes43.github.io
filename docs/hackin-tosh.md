---
title: Backup Strategy for Everything (so far) 
permalink: /docs/hackin-tosh/
---

## Motivation

I wanted to set up a hackintosh VM for it's iCloud sync abilities. I had been using my Windows 11 desktop PC for this and the iCloud program was causing BSODs. My alternative plan, or maybe I something I will do as well, is to write a program using [icloud-photos-downloader](https://github.com/icloud-photos-downloader/icloud_photos_downloader). Once I download the photos I want to use [ImageMagic](https://github.com/dlemstra/Magick.NET) to convert the `HEIC` ones and upload them to unRAID so they can be seen on Plex. I'm also open to a alternative to Plex for Photos since it's pretty bad.

## Installing the OS

At first I tried to follow [these steps](https://www.nicksherlock.com/2022/10/installing-macos-13-ventura-on-proxmox/) using my own .img seeing how I have a mac mini. Unfortunately, it refused to boot after the OS install was complete.

I pivoted to [these sketchier steps](https://klabsdev.com/definitive-guide-to-running-macos-in-proxmox/) but went with Ventura instead of Sonoma.

I immediately hit an issue where it refused to boot the ISO. I was getting an error like `proxmox failed to load Boot0001 access denied`. Disabling secure boot fixed me up.

1. Mash escape while booting the VM
2. In the UEFI Bios menu go to Device Manager and find something about Secure Boot
3. Un-tick the x so it's disabled
4. Save and reset

After this I was able to get into the steps of the second tutorial. Note that you can use the mouse most of the time but the initial menu is keyboard only (even though a mouse pointer shows up).

And it worked!

![select language]({{ site.url }}/images/hosting/select-language.png)

Don't use the scroll wheel when selecting the language, use the arrow keys once close...

The rest is clicking either "Not Now" or "Setup Later" when available since we need to do a bit more work to link this to our iCloud ID.

Once in I get this error but it walks me through some ways to identify the keyboard. Everything seems fine...

![keyboard assistant broken]({{ site.url }}/images/hosting/keyboard-assistant-broken.png)

> **NOTE:** I installed ventura here because that was the same version as the image I pulled from the mac mini. I may make another for Sonoma in the future.

## Installing nomachine

Before I go any further I want a better way to access the VM. There are many but I have ended up using nomachine for most VMs since it's easy to install and then they instantly show up in the client. Spice is nice too but it's a bit more of a hassle. 

Safari wasn't working well so I installed chrome first. Then I went to `nomachine.com` and clicked the huge "Download now" button. The installer had a UI and was straightforward. Make sure to enable the permissions it prompts for.

Back on my desktop I can now select the mac VM with it's default name that I should change later...

![nomachine new mac]({{ site.url }}/images/hosting/nomachine-new-mac.png)

## Fixing Things Up

https://github.com/thenickdude/KVM-Opencore/releases?ref=klabsdev.com -> download something like `OpenCoreEFIFolder-v21.zip` from release

https://github.com/corpnewt/MountEFI?ref=klabsdev.com -> clone

`git clone` on the http link popped up a nice window for installing `git` instead of just saying it wasn't found so I went ahead and did that. Then I ran the `git clone` command again to pull the repo.

Running these commands:

```
cd ~/Downloads/MountEFI-update
chmod +x MountEFI.command
./MountEFI.command
```

Got me this popup:

![mount efi]({{ site.url }}/images/hosting/mount-efi.png)

Following the guide I selected the MacOS drive and entered my password. It then mounted an empty drive which was not what the guide said but I ran with it and copied the EFI folder from `OpenCoreEFIFolder-v21.zip` in. Then I shut the VM down.

Finally I removed OpenCore ISO from the hardware and was able to boot back in without it!

## Getting to iCloud

Coming Soon!