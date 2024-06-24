---
title: Pop!_OS
permalink: /docs/pop-os/
---

## HaynesIntelligence on Pop!_OS

Pop!_OS was recommended for Ollama and it's Debian based, feels a lot like Ubuntu, so I didn't hesitate to try it. I beefed this mother up like crazy as it'll be the single minded focus of the beef machine hosting it. 

### Creating the VM

For the VM I did my standard GPU passthrough settings:

* UEFI Bios
* q35 machine type
* host cpu

It didn't work the first try. If an ISO doesn't start disabling secure boot in Proxmox UEFI this has become my default attempt to fix it which usually works. It worked.

Next up was pressing next a bunch and saying no I don't want to encrypt my fake drives. 

After agreeing to more default values I was quickly into the OS. A nag window wanted me to update it so I figured why not.

After updating I renamed the PC to match the VM's name and installed [nomachine](https://www.nomachine.com/). Then I brought it down to pass the two GPUs in.

### GPU Passthrough

The [ISO](https://iso.pop-os.org/22.04/amd64/nvidia/41/pop-os_22.04_amd64_nvidia_41.iso) I installed for Pop!_OS even had Nvidia in the name so I was feeling pretty good that this wouldn't be another hackintosh adventure. Sure enough they went right in. 

![dual gpus]({{ site.url }}/images/builds/popos/dual-gpus.png)

Oddly my settings were a bit different and I had a hard time changing the resolution but sometimes a dummy plug can make a world of a difference so I went for one of those. That's when I realized it was still connected to my ancient basement monitor which likely explains the limited resolution options but it's still getting dummy plugs.

Nomachine wasn't doing great changing resolutions with the GPU installed so I decided to do something extreme. I enabled a virtual display but kept the passthrough. VirtIO GPU was snappy with 512MB and I then the 3090s were dedicated to llama 3

#### Few things I Installed

```
sudo apt install system76-driver-nvidia
```

Some add-ons:

```
gnome-shell-extension-prefs
```

GPU Check:

### Ollama

[Ollama](https://ollama.com/) was wicked easy to install via the cli. I quickly grabbed [llama3](https://ollama.com/library/llama3) using `ollama run llama3` and started chatting. First question was if it was using both GPUs. It didn't know but gave me this command:


```
nvidia-smi
```

Then it said to set:

```
CUDA_VISIBLE_DEVICES
``` 

and:

```
OPENCL_CACHED_MEM
```

But I think it's fine, need to train to see for sure. Next I tried `mistral`

```
ollama run mistral
```

#### So Many

Here are a few ready to go. Even an uncensored one!

Model	Parameters	Size	Download
Llama 3	8B	4.7GB	ollama run llama3
Llama 3	70B	40GB	ollama run llama3:70b
Phi 3 Mini	3.8B	2.3GB	ollama run phi3
Phi 3 Medium	14B	7.9GB	ollama run phi3:medium
Gemma	2B	1.4GB	ollama run gemma:2b
Gemma	7B	4.8GB	ollama run gemma:7b
Mistral	7B	4.1GB	ollama run mistral
Moondream 2	1.4B	829MB	ollama run moondream
Neural Chat	7B	4.1GB	ollama run neural-chat
Starling	7B	4.1GB	ollama run starling-lm
Code Llama	7B	3.8GB	ollama run codellama
Llama 2 Uncensored	7B	3.8GB	ollama run llama2-uncensored
LLaVA	7B	4.5GB	ollama run llava
Solar	10.7B	6.1GB	ollama run solar


#### Retrieval Augmented Generation

Retrieval Augmented Generation keeps coming up as something I'll need if I want this thing to be smart. [This article](https://towardsdatascience.com/running-llama-2-on-cpu-inference-for-document-q-a-3d636037a3d8) looks like a good start.