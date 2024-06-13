---
title: Hades Benchmark
permalink: /benchmarks/hades/
---

I decided to run the game [Hades](https://store.steampowered.com/app/1145360/Hades/) on a bunch of shit to see how it performed with different configs. I picked this game because it required low input lag to enjoy and should theoretically be easy to run. Unfortunately I didn't have a consistent way to collect any metrics so the actual "benchmark" is just my remarks.

## Fresh GPU Passthrough Comparisons

Hades was the first thing I ran when I passed though the ![RX 6400]({{ site.url }}/docs/gpu-passthrough/)

To quantify the difference a GPU makes we're gonna need some benchmarks. I want to try the RX 6400 vs. iGPU vs. nothing but I need some way to collect metrics that all three support. Xbox game gar should work! 

I am also going to need another VM to first test without a GPU and then passthrough the iGPU so I don't regress my working "gaming VM". Fortunately, that is super easy to make. Just need to install the guest agent [this](https://pve.proxmox.com/wiki/Qemu-guest-agent) time when the ISO is mounted. It was easy as opening the disk and installing the msi in the guest agent folder.

Rules:
1. 1280x720 resolution set to full screen
2. Parsec used for remote access w/ controller

##### RX 6400 Benchmark - Hades

Getting the fps to show up turned out to be difficult. Game bar worked once I flipped it into windowed mode and then back to full screen. Hades on the RX 6400 was showing > 50fps average in something it called "performance report":

![rx6400 hades performance report]({{ site.url }}/images/windows/rx6400-hades-performance-report.png)

Live game bar was a bit lower from the sample I took actually playing. I assume "performance report" gets thrown off from the extremely high fps I get when the game is paused. That was > 30fps:

![rx6400 hades benchmark]({{ site.url }}/images/windows/rx6400-hades-benchmark.png)

##### No GPU - Hades

This one is going to be interesting, right off the bat the virtual display only supported 1280x800 which is pretty close to the baseline. Before messing with the passed in `Display` I decided to go with it.

Once I opened the game parsec immediately lowered the bit rate to try and compensate for something it had no control over since the host was going to render at the same speed regardless. Frames were much lower, usually < 15fps:

![nogpu hades benchmark]({{ site.url }}/images/windows/nogpu-hades-benchmark.png)

It felt worst than twice as bad as the GPU benchmark because the game froze constantly which wasn't really reflected in the FPS. I ended up dying immediately in an area I was breezing through before. The screen shot did a good job at capturing the crippled bitrate: 

![nogpu dead]({{ site.url }}/images/windows/nogup-dead.png)

##### Virtio GPU - Hades

Before I attempt to passthrough the iGPU I wanted to see if any of the other Display options had an impact. Since there were too man to try all of them I read up [here](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_virtual_machines_settings) on what they did. That wasn't really helpful so I tried Virtio GPU w/ max memory (512). 

That went south fast:

![virtio gpu bsod]({{ site.url }}/images/windows/virtio-gpu-bsod.png)

However, on reboot, it went to the log in screen so I continued marching forward. Sill, no GPU showed up under Task Manager so my expectations were very low. However, this did unlock an interesting subset of resolutions so I tired to even the playing field by going down a notch:

![virtio gpu resolutions]({{ site.url }}/images/windows/virtio-gpu-resolutions.png)

For some reason 800x600 froze Parsec so I stuck with the classic 1024x768 which tasted like nostalgia. I couldn't even get fps metrics to show for this, and the picture was nearly illegible, so I just called it:

![hades virti0 gpu bad]({{ site.url }}/images/windows/hades-virti0-gpu-bad.png)

Parsec also couldn't get sound with this Display but when I opened nomachine to save myself from a blurry screen the sound popped right on.

##### VirGL GPU - Hades

After my last mishap I was sure this would fail too but since it was the mode I think the documentation referenced I figured I'd give it a fair chance. This was also constrained to 512MB of memory which I don't think was really passed in last time since no GPU was found.

Another failure to launch right off the bat:

```
TASK ERROR: missing libraries for 'virtio-gl' detected! Please install 'libgl1' and 'libegl1'
```

Easy to install though:

```
apt install libgl1
apt install libegl1
```

Booted up after that but same awkward resolutions and no GPU showing up as before. Parsec did get volume so that was nice. The game bar refused to even start this time so I wasn't going to be collecting any metrics but it looked exactly the same as above. So for whatever reason these options made it worst! 

##### iGPU - Hades

See the [iGPU Saga]({{ site.url }}/docs/igpu-passthrough/) for how I got it running Unfortunately both Parsec and nomachine behave poorly with the iGPU passed through. However, RDP worked great and the game was playable. 

![hades igpu huge]({{ site.url }}/images/windows/hades-igpu-huge.png)

Game bar didn't work for FPS but and it wasn't running nearly as well as with the RX 6400 but it could be played which was not the case for anything other than the RX 6400.

