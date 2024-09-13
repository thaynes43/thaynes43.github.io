---
title: Backlog
permalink: /backlog/
---

## Backlog

First I should really

1. Finish migrating the documentation I wrote on my [gist](https://gist.github.com/thaynes43/6135cdde0b228900d70ab49dfe386f91) with the help of the [good gist](https://gist.github.com/scyto/76e94832927a89d977ea989da157e9dc)
2. Finish porting over the Notepad++ saga
3. Document the hardware in the lab
4. Host a real tool for managing a backlog

The rest of this page is things I may or may not take on but don't want to lose track of. 

### Helm

* Check out [homelab charts](https://github.com/k8s-home-lab/helm-charts) inspired by [k8s at home](https://k8s-at-home.com/) that I missed out on!
* Make my own chart for Cloudflare-DDNS (see ![my doc]({{ site.url }}/docs/cloudflare-ddns/))
* Make my own chart for something that has no chart
* See what [Helmfile](https://helmfile.readthedocs.io/en/latest/) is (and figure out what template they used for pages!)
* Check out this [sweet repo](https://github.com/onedr0p/home-ops/tree/main) for what I guess they are calling HomeOps
* Explore [TrueCharts](https://truecharts.org/) cause it has a shit ton but some terrible ads, but [this is better](https://github.com/truecharts/charts/tree/master/charts/stables)

### Kube

* See what [kubevip](https://kube-vip.io/docs/installation/) is all about!
* Use the [Fail2Ban](https://plugins.traefik.io/plugins/628c9ebcffc0cd18356a979f/fail2-ban) Middleware for traefik and explore what other middleware is offered
* Open WebUI w/ SSO
* Finish or delete `whoami` Helmrelease
* Finish or delete `podinfo` Helmrelease
* Delete the nginx example deployment / ingress / DNS entry
* Setup `external-dns` to add UniFi DNS entries
* Get IoT Started
* Migrate media management & download clients

### HAOS

* Get SSO working with HAOS too
* IngressRoute for HAOS-PVE so we can hit it from the web
* Database we can backup and migrate data from the unRAID VM into?
* This list is endless

#### Lens

See [lens](https://docs.k8slens.dev/) later...

#### Comments for the Site

* [disqus](https://disqus.com/) can be used to hold comments people make on these pages
* [minimal-mistakes](https://mmistakes.github.io/minimal-mistakes/docs/configuration/#disqu) needs to be configured to take advantage of it
* Need to evaluate other alternatives as well

#### Use or Delete Pages

* Currently set up to support a blog style pages
* Can do pagination with some additional config
* Can migrate posts on `HaynesTower` wordpress to here

### Infrastructure

#### Temp Monitoring 

* In Proxmox on the dashboards looks good via [this guide](https://sluijsjes.nl/2024/05/20/how-to-show-your-cpu-temperature-in-the-proxmox-web-interface/)
* Something in grafana later would be good too for historical data

#### Finish Backup Strat

* See [backup-strat]({{ site.url }}/docs/backup-strat/)

#### Figure Out Radosgw

Installed on three nodes, not configured for the other two but is probably fine. See [here](https://devopstales.github.io/home/proxmox-ceph-radosgw/)

#### Document and Update HAProxy

* This ended up being trivial for Cephdash but brutal for proxmox. This [post](https://forum.proxmox.com/threads/pxve-nginx-websocket-proxy.20886/#post-139486) was the closest I got to a guide
* Doesn't work with Spice

#### Stick a GPU in an MS-01

* Going to try an [XFX Speedster RX 6400](https://www.amazon.com/gp/product/B09Y7358KJ/ref=ppx_yo_dt_b_asin_title_o00_s00?ie=UTF8&psc=1) and hope it fits
* Maybe get a [game VM](https://forum.proxmox.com/threads/windows-11-vm-for-gaming-setup-guide.137718/) running.

#### iGPU Passthrough

* First I need to expose it to Proxmox using [this guide](https://www.reddit.com/r/Proxmox/comments/14fzj8l/tutorial_full_igpu_passthrough_for_jellyfin/)
* After I can try it w/ the macos VM using [OpenCore patching](https://dortania.github.io/OpenCore-Post-Install/gpu-patching/intel-patching/#terminology)
* Then I can try it with [Waydroid](https://docs.waydro.id/faq/get-waydroid-to-work-through-a-vm)

#### Flux

* [Flux](https://fluxcd.io/flux/) is a pre-req to the backup strat if I want to follow the docs I referenced there
* This blog has a good [tutorial](https://geek-cookbook.funkypenguin.co.nz/kubernetes/deployment/flux/operate/)
* [podinfo](https://github.com/stefanprodan/podinfo?tab=readme-ov-file) seems like a "Hello World" for best practices

#### Getting Grafana working w/ Ceph Dashboard 

* See [here](https://docs.ceph.com/en/quincy/mgr/dashboard/#enabling-the-embedding-of-grafana-dashboards)

### Network

#### Better Web Traffic Statistics

* Web Traffic Analysis using [ntopng](https://www.ntop.org/products/traffic-analysis/ntop/)
* Can run directly on DMSE with [notpng-udm](https://github.com/tusc/ntopng-udm?tab=readme-ov-file) but needs a special [boot script](https://github.com/unifi-utilities/unifios-utilities/tree/main/on-boot-script) to start automatically. 

#### InnerSpace w/ Floor Plan

Once we move I will set up InnerSpace for the new house's coverage and use the plans I have now to map everything out. See [here](https://www.reddit.com/r/Ubiquiti/comments/18n6qme/innerspace_and_making_a_floor_plan/) for making custom floor plans.

### Smart Home / Car

#### Integrate Bhyve in HA

* Bhyve app is terrible. I at least want one button to rain delay all the timers. [This integration](https://github.com/sebr/bhyve-home-assistant) works but is also brutal. Might not be worth the effort since I won't need the timers when I move.

#### Reconfigure Alexa for Amazon Home

* Though I want to replace Alexa w/ [HA Assist and an LLM](https://www.home-assistant.io/blog/2024/06/05/release-20246/#dipping-our-toes-in-the-world-of-ai-using-llms) my current approach here is a mess. Need different accounts for different devices so people stop interrupting each other's Spotify.

#### TeslaMate

* [TeslaMate](https://docs.teslamate.org/docs/installation/docker) looks great but I'm going to have to dig into each dependency and see if it's worth isolating it all to it's own namespace or using existing `mosquitto` / `grafana` instances since a lot more than this needs those.

#### Automatic Tesla Cam Backups

* [TeslaUSB](https://github.com/marcone/teslausb?tab=readme-ov-file) can do this but I'm running a larger SSD and probably don't need this since I can pull it manually if/when it's full.

### AI

#### AI Server

* I bought parts for this, need to document the hardware

#### Local LLM

* [Pop!_OS](https://pop.system76.com/) recommended for hosting [ollama](https://ollama.com/)
* [STABLE DIFFUSION](https://github.com/Stability-AI/StableDiffusion)!

#### AI LINKS

* https://medium.com/@vndee.huynh/build-your-own-rag-and-run-it-locally-langchain-ollama-streamlit-181d42805895
  * https://github.com/vndee/local-rag-example
* https://github.com/open-webui/helm-charts/tree/main/charts/open-webui
* https://github.com/tloen/alpaca-lora
  * https://github.com/tloen/alpaca-lora/issues/8#issuecomment-1477490259
* https://pytorch.org/
* https://github.com/yurijmikhalevich/rclip
* https://immich.app/
* https://r2r-docs.sciphi.ai/cookbooks/local-rag
* https://www.reddit.com/r/LocalLLaMA/comments/1dltq6o/best_rag_libraries_for_chunking_embedding_and/
* https://github.com/hiyouga/LLaMA-Factory
* https://huggingface.co/datasets
* https://github.com/modelscope/modelscope
* https://github.com/Lawfam/LLM-Collab -> MAKE EM TALK TO EACH OTHER

### C# / App Development

#### Convert HEIC Photos and Transfer to Plex

* [icloud-photos-downloader](https://github.com/icloud-photos-downloader/icloud_photos_downloader) or the hackintosh to pull from iCloud
* [ImageMagic](https://github.com/dlemstra/Magick.NET) to convert the `HEIC` 
  * CLI app that can convert and upload to a staging spot where I can then curate what makes it to Plex

#### Finish Plex Library Catalogue 

* Real rough app [here](https://github.com/thaynes43/plex-library-catalogue) could use fixing up and a way to run it daily
* Abstract how it gets the data and implement a version that does not require [Tautulli](https://tautulli.com/) 

#### Hook HaynesTower into Proxmox Cluster

* Maybe a bad idea but I could host a proxmox VM following the steps for a [pbs VM](https://forum.proxmox.com/threads/how-to-setup-pbs-as-a-vm-in-unraid-and-uses-virtiofs-to-passthrough-shares.120271/) so I could use Proxmox to manage VMs on HaynesTower instead of unRAID (and migrate them easier)

#### Hook HaynesHyperion into Proxmox Cluster

* This is likely a bad idea but I can play around with the extra horsepower HaynesHyperion offers though the same cluster management if I install a proxmox VM there.

### Self Hosted

These are just things I want to try and host

#### Vikunja

* First on the list for family organizers would be [vikunja](https://vikunja.io/). Looks like a great candidate for the smart home dashboard.

#### Nextcloud

* [Nextcloud](https://nextcloud.com/) is highly recommended and has a ton of features but I want to wait until things are more stable to host something that would need to be rock solid like replacing a cloud provider


#### Joplin 

* [joplin](https://joplinapp.org/) for notes might be a good for mobile. 
* Need [SeaweedFS](https://github.com/seaweedfs/seaweedfs) or something for self hosted back end storage. The app seems to do the rest. 

### Tour de OS

Found a [cool site](https://distrowatch.com/dwres.php?resource=major) that lists and describes distros. 

* [HOLOISO](https://github.com/HoloISO/releases) for a SteamOS 3 (deck OS) esq implementations 
* [Pop!_OS](https://pop.system76.com/) for AI
* [Bliss OS](https://sourceforge.net/projects/blissos-dev/) which is what Waydroid-Linux points to and seems better than the containerized install on Ubuntu
* [Madness](https://en.wikipedia.org/wiki/TempleOS)

### World Server

[AzerothCore](https://www.azerothcore.org/wiki/home) looks pretty suck tbh.

### DUMP

http://192.168.0.200:8085/#Homepage
http://homeassistant.local:8123/adaptive-dash/hallways
http://192.168.0.200:32400/web/index.html#!/
http://192.168.0.62:8080/#/login?returnUrl=%2Fdashboard
https://pvedash/#v1:0:=qemu%2F100:4:=zfs:=contentImages:::8:=consolejs:21
https://192.168.0.14:8007/#pbsDashboard

#### NOT PINNED

https://docs.nvidia.com/deeplearning/frameworks/user-guide/index.html#runcont
https://github.com/thaynes43/thaynes43.github.io
https://us.moshi.chat/
http://homeassistant.local:8123/config/devices/device/e38c74aea575987ca65cc36877dccb1e
https://openai.com/index/hello-gpt-4o/
https://platform.openai.com/playground/chat?models=gpt-4o
https://chatgpt.com/gpts
https://tteck.github.io/Proxmox/#turnkey-lxc-appliances
https://www.turnkeylinux.org/
https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch
https://github.com/meta-llama/llama-recipes#install-with-pip
https://github.com/OpenPipe/OpenPipe/tree/main/examples/classify-recipes
https://llama.meta.com/docs/how-to-guides/fine-tuning/
https://pytorch.org/torchtune/main/tutorials/e2e_flow.html
https://github.com/pytorch/torchtune/blob/main/recipes/configs/llama2/7B_lora_single_device.yaml
https://github.com/pytorch/torchtune/blob/main/recipes/configs/llama3/8B_lora.yaml
https://pytorch.org/torchtune/stable/api_ref_datasets.html#datasets
https://pytorch.org/torchtune/stable/api_ref_datasets.html#datasets
https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch
https://jupyter.org/install
https://forums.developer.nvidia.com/t/using-a-jupyter-notebook-within-a-docker-container/60188/3
https://pytorch.org/torchtune/stable/tutorials/e2e_flow.html#e2e-flow
http://192.168.0.250:61208/
https://blog.kye.dev/proxmox-cockpit
https://docs.ceph.com/en/latest/rados/operations/add-or-rm-osds/
https://docs.openwebui.com/tutorial/openedai-speech-integration/
https://forums.unraid.net/topic/140380-supermicro-cse-847-36-bay-4u-sas-3-barebone-chassis-2x-pws-1k28p-sq/#comment-1414637
https://gist.github.com/thaynes43/6135cdde0b228900d70ab49dfe386f91
https://gist.github.com/scyto/76e94832927a89d977ea989da157e9dc
https://docs.ceph.com/en/quincy/start/hardware-recommendations/
https://forums.servethehome.com/index.php?threads/minisforum-ms-01-pcie-card-and-ram-compatibility-thread.42785/page-49
https://community.home-assistant.io/t/openassist-chatgpt-can-now-control-entities/579000
https://www.home-assistant.io/integrations/openai_conversation/
http://homeassistant.local:8123/config/voice-assistants/assistants
https://music-assistant.io/
https://docs.ceph.com/en/quincy/mgr/dashboard/#enabling-the-embedding-of-grafana-dashboards
https://cockpit-project.org/
https://huggingface.co/datasets/wikimedia/wikipedia
https://docs.openwebui.com/features/
https://docs.trychroma.com/
https://www.reddit.com/r/ollama/comments/1dtcmxz/local_rag_powered_by_ollama/
https://r2r-docs.sciphi.ai/installation <--- NEXT
https://dortania.github.io/Anti-Hackintosh-Buyers-Guide/GPU.html
