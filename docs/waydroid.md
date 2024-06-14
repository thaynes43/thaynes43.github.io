---
title: Waydroid
permalink: /docs/waydroid/
---

[Waydroid](https://docs.waydro.id/) seems to be the best way to emulate Android in a VM. I tried [Android-x86](https://www.android-x86.org/) first but it seems to have gone out of style.

There seems to be two ways to run Waydroid. First is on top of an Ubuntu VM and the second is a new distro called [Waydroid-Linux](https://waydro.id/index.html#wdlinux) which is in Beta as of writing this. 

## Backstory

`Waydroid-Linux` wasn't out when I first set this up so I went with the VM option. As of writing this I am setting it up for the third time. If that works I'll do it again to try `Waydroid-Linux`.

1. First I did not select GAPPS when installing it and it was way harder to add it after so I just started over
2. Second I used SeaBIOS so I couldn't figure out how to pass the GPU in after
3. This time I have `GAPPS` and `UNFI` bios so we should be cooking with gas

## Running On Ubuntu 24.04

First I got an Ubuntu VM w/ GPU passthrough going as loosely descibed in [that saga][{{ site.url }}(/docs/gpu-passthrough#ubuntu-passthrough/).

### Install Waydroid

Followed the simple steps to install Waydroid:

```
sudo apt install curl ca-certificates -y
curl https://repo.waydro.id | sudo bash

sudo apt update
sudo apt upgrade

sudo apt install waydroid -y
```

Then just launch it for the installer:

![waydroid button]({{ site.url }}/images/waydroid/waydroid-button.png)

Then make sure you select GNAPPs and wait a while cause it downloads a whole lot or maybe a little but very slowly:

![waydroid installer]({{ site.url }}/images/waydroid/waydroid-installer.png)

### Certify Play Protect

The first thing you will notice is that google play is angry. There is sparse [documentation](https://docs.waydro.id/faq/google-play-certification) on how to appease it. 

Run `sudo waydroid shell` to pop the shell and then blast in this guy:

```
ANDROID_RUNTIME_ROOT=/apex/com.android.runtime ANDROID_DATA=/data ANDROID_TZDATA_ROOT=/apex/com.android.tzdata ANDROID_I18N_ROOT=/apex/com.android.i18n sqlite3 /data/data/com.google.android.gsf/databases/gservices.db "select * from main where name = \"android_id\";"
```

That will give you an `android_id`

```
android_id|4091825162128642442
```

You will need to bring that to the headmasters at [android certification](https://www.google.com/android/uncertified).

After a bit they will grace you with the permission to use their store. Make sure you are logged into the google account you plan to use. I think you can register it to multiple accounts but I was quick on the trigger and threw the ID into a personal account I won't use on the VM.

### Waiting for Play Protect

Play Protect isn't quick, so I'm going to reboot since that is what you do when software frustrates you. That worked, but while I waited for it to reboot I read the [doc](https://docs.waydro.id/faq/google-play-certification) again and learned you are need to bounce waydroid for it to take so I guess a reboot was overkill.

Now that the google certification police left me alone I realized the Ubuntu `dock` was cramping my style so I went to `Settings` -> `Ubuntu Desktop` and ticked `Auto-hide the Dock`

Things didn't line up quite right after so I rebooted again and it finally looked great:

![fresh waydroid]({{ site.url }}/images/waydroid/fresh-waydroid.png)

### The Problem I Didn't Mention

My last Waydroid VM had a show stopping problem where videos wouldn't play, just a black screen. After much thought I came to the concision that I should try passing in an AMD GPU. I have this thought a lot, sometimes it's for NVidia, sometimes for intel even, so I didn't jump on it right away. But it worked!

![video works]({{ site.url }}/images/waydroid/video-works.png)

