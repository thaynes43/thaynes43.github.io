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

![dual gpus]({{ site.url }}/images/popos/dual-gpus.png)

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

[Ollama](https://ollama.com/) and on [github here](https://github.com/ollama/ollama) was wicked easy to install via the cli. I quickly grabbed [llama3](https://ollama.com/library/llama3) using `ollama run llama3` and started chatting. First question was if it was using both GPUs. It didn't know but gave me this command:

```
curl -fsSL https://ollama.com/install.sh | sh
```


```
nvidia-smi
watch -n 0.5 nvidia-smi
```

or git bash this works:

```
nvidia-smi -l 1
```

```
nvcc --version
```

```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2021 NVIDIA Corporation
Built on Thu_Nov_18_09:45:30_PST_2021
Cuda compilation tools, release 11.5, V11.5.119
Build cuda_11.5.r11.5/compiler.30672275_0
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

GPU Issue:

```
sudo rmmod nvidia_uvm && sudo modprobe nvidia_uvm
```

#### Rebooting Breaks GPU

```
In the meanwhile, if you want to switch back to using the GPU after resuming from a suspend, you can reload the nvidia_uvm:

First stop the service:
systemctl stop ollama

Reload nvidia_uvm:
sudo rmmod nvidia_uvm && sudo modprobe nvidia_uvm

Then restart the service:
systemclt start ollama

That should allow you to run a model again on the GPU.
```

This was taken from a [github bug](https://github.com/ollama/ollama/issues/3489#issuecomment-2094665760) that inspired additions to ollama's [troubleshooting guide](https://github.com/ollama/ollama/blob/main/docs/troubleshooting.md#container-fails-to-run-on-nvidia-gpu). 

TODO try this

#### So Many Models

Here are a few ready to go. Even an uncensored one!

| Model | Parameters | Size | Download |
| Llama 3 | 8B | 4.7GB | ollama run llama3 |
| Llama 3 | 70B | 40GB | ollama run llama3:70b |
| Phi 3 Mini | 3.8B | 2.3GB	| ollama run phi3 | 
| Phi 3 Medium | 14B | 7.9GB	ollama run phi3:medium |
| Gemma | 2B | 1.4GB | llama run gemma:2b |
| Gemma | 7B | 4.8GB | ollama run gemma:7b |
| Mistral | 7B | 4.1GB | ollama run mistral |
| Moondream | 2 | 1.4B | 829MB | ollama run moondream |
| Neural Chat | 7B | 4.1GB | ollama run neural-chat |
| Starling | 7B | 4.1GB | ollama run starling-lm |
| Code Llama | 7B | 3.8GB | ollama run codellama |
| Llama 2 Uncensored | 7B | 3.8GB | ollama run llama2-uncensored |
| LLaVA | 7B | 4.5GB | ollama run llava |
| Solar | 10.7B | 6.1GB | ollama run solar |

And the complete list is [here](https://ollama.com/library).

Found a huge one! `ollama run qwen:110b` pulled 62 GB. 

And some [spicy ones](https://erichartford.com/uncensored-models).

### CUDA Upgrade

This [guide](https://support.system76.com/articles/cuda/) from the creators of Pop!_OS gives a good walkthrough on upgrading CUDA though it seems a bit dated:

```
thaynes@pop01:~$ docker run -it --rm --runtime=nvidia --gpus all nvidia/cuda:12.1.0-devel-ubuntu22.04 bash

==========
== CUDA ==
==========

CUDA Version 12.1.0

Container image Copyright (c) 2016-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.

This container image and its contents are governed by the NVIDIA Deep Learning Container License.
By pulling and using the container, you accept the terms and conditions of this license:
https://developer.nvidia.com/ngc/nvidia-deep-learning-container-license

A copy of this license is made available in this container at /NGC-DL-CONTAINER-LICENSE for your convenience.

*************************
** DEPRECATION NOTICE! **
*************************
THIS IMAGE IS DEPRECATED and is scheduled for DELETION.
    https://gitlab.com/nvidia/container-images/cuda/blob/master/doc/support-policy.md
```

Will upgrade later but [pytorch](https://pytorch.org/) doesn't seem easy for 12.4.

Aaand now I have a docker container running CUDA 12.1 which is not what I was going for. 

Further down we see:

```
sudo apt install system76-cuda-latest
sudo apt install system76-cudnn-11.2
nvcc -V
```

But I have the latest that offers and it's 11.5 which has no pytorch version associated with it!

[Here](https://www.reddit.com/r/pop_os/comments/15fjklg/how_to_get_specific_versions_of_cuda_and_pytorch/) is someone as confused as I am about this CUDA docker. Seems like there is a [PyTorch container](https://docs.nvidia.com/deeplearning/frameworks/pytorch-release-notes/rel-23-03.html) with what I need. [Here](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch) is how to run it.

## Docker

It looks like I need to run [PyTorch on docker](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch) which is fine but if I'm going to do that I might as well also run [ollama's image](https://hub.docker.com/r/ollama/ollama). 

PyTorch can be ran with this:

```
docker run --gpus all -it --rm nvcr.io/nvidia/pytorch:24.06-py3
```

It tells me I should have done this:

```
docker run --gpus all --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 -it --rm nvcr.io/nvidia/pytorch:24.06-py3
```

And that I am running a container that supports CUDA 12.5 but I have 12.4 and that's just silly nonsense and I really should run a more comparable one or at least know a thing or two about it [here](https://docs.nvidia.com/deploy/cuda-compatibility/).

I also need to map some storage with:

```
docker run --gpus all -it --rm -v local_dir:container_dir nvcr.io/nvidia/pytorch:xx.xx-py3
```

But the good news is I've got PyTorch w/ CUDA support without any heavy fiddling:

```
root@d127755b81d1:/workspace# python
Python 3.10.12 (main, Nov 20 2023, 15:14:05) [GCC 11.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import torch
>>> print(torch.cuda.is_available())
True
```

So storage plus Then I can rock and roll.

tune download meta-llama/Meta-Llama-3-8B \
    --output-dir /workspace/downloads \
    --hf-token <redacted>



## Open-WebUI

[open-webui](https://github.com/open-webui/open-webui) is the go to UI for ollama and other local LLMs stuff. This looked like a great thing to run on my cluster which is intended to host endpoints people would hit vs. back end compute / storage servers. The charts were off in their own [repo](https://github.com/open-webui/helm-charts) but didn't look bad and I had been neglecting all the hard work I did setting up k8s and ceph-csi.

## Retrieval Augmented Generation

Retrieval Augmented Generation keeps coming up as something I'll need if I want this thing to be smart. [This article](https://towardsdatascience.com/running-llama-2-on-cpu-inference-for-document-q-a-3d636037a3d8) looks like a good start but it's PAYWALLED! Found a ton more.

## Other Things

I found this cool [guide](https://github.com/mikeroyal/Pop_OS-Guide) that details a bunch of other things to do with POP OS. 