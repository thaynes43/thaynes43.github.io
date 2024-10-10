---
title: Shopping List
permalink: /docs/moving-day/shopping-list/
---

## Before Moving

### Split the Cluster (ORDERED)

Now that I am deeper in the weeds I think two clusters is the way to go. Both will have 3-5 nodes running Ceph but one will be `Proxmox` and the other `Talos`. Three nodes w/ 8TB of NVMe for Ceph in each cluster is enough for what I want to host (plus massive NFS shares) and this will give me the ability to go hard on k8s over in Talos land without breaking a simpler setup on `Proxmox` VMs.

I only need one more MS-01 to start and all the fixings.

| Item | Quantity | Total $ |
| [MS-01](https://store.minisforum.com/products/minisforum-ms-01) Barebone | 1 | $679 |
| Crucial RAM 96GB | 1 | $237 |
| Crucial T500 1TB | 1 | $77 |
| [PM983a](https://www.ebay.com/itm/284524529831) 2TB | 2 | $318 | 
| [SFP+ DAC](https://store.ui.com/us/en/collections/unifi-accessory-tech-cable-sfp/products/10gbps-direct-attach-cable?variant=uacc-dac-sfp10-3m&category=ce60c14e-dd22-47a9-b051-37e1a48b8d4f) | $20 | 2 | $40 | 3m was too long |

I will also need USB4 cables if I go with a ring network - this is now possible with both clusters having three nodes. However, I can't expand past three nodes if I go this route unless I reconfigure which can be tedious. [Here](https://www.youtube.com/watch?v=Tb3KH2RbsTE&list=PLJ96mRkYRhLBPdOOaojAhuwOka5gdn-l-) is a video, more has been done since I tried last.

| Item | Quantity | Total $ |
| USB4 Cables | 3 | $42 | 

#### Ideas for Plex Nodes

I asked about HBAs [here](https://forum.45homelab.com/t/hl15-with-only-8-lanes-for-hba/1727/5) for some really nice chassis from 45 drives but they are steep in price and 4U.

### Full the Array

I need 3 more 20TB drives to hit 30 in unRAID. The price on these is way up, ServerPartDeals is sold out of X22s and [X20s](https://serverpartdeals.com/products/seagate-exos-x20-st20000nm007d-20tb-7-2k-rpm-sata-6gb-s-3-5-recertified-hard-drive) are going for $240. Can wait on this.

| Item | Quantity | Total $ |
| x22 20TB | 3 | $750 | 

### Power Density (ORDERED)

Gonna consume a lot of power in a little space! 

#### Server Rack (ORDERED)

[NetShelter 42U Unassembled](https://www.apc.com/us/en/product/AR2400FP1/netshelter-sv-42u-600mm-wide-x-1060mm-deep-enclosure-with-sides-black-single-rack-unassembled/?range=62589-netshelter-sv-enclosures&parent-subcategory-id=88954&selectedNodeId=27590146204) looks good given how tight getting the rack into the basement will be.

#### UPS (ORDERED)

I think [this one](https://www.apc.com/us/en/product/SMX3000RMLV2UNC/apc-smartups-x-line-interactive-3kva-rack-tower-convertible-2u-100v127v-3x-515r+3x-520r+1x-l530r-nema-nmc-extended-runtime/) is the right one. These are kind of hard to shop for. However, [this one](https://www.apc.com/us/en/product/SMX3KR2UNCX145/apc-smartups-x-line-interactive-3kva-rack-tower-convertible-2u-100v127v-3x-515r+3x-520r+1x-l530r-nemanmc-extended-runtime10ft-input-cord/?range=61915-smartups&parent-subcategory-id=88976&selectedNodeId=23679172486) looks a exactly the same but the cord is longer and it's cheaper... How could that be right?

First, ones that look exactly the same on paper will vary in price because one isn't TAA compliant which I believe just means they can source parts out of country. 

SMX3000RMLV2UNC - $3075:
```
APC Smart-UPS X, Line Interactive, 3kVA, Rack/tower convertible 2U, 100V-127V, 3x 5-15R+3x 5-20R+1x L5-30R NEMA, NMC, Extended runtime
```

SMX3KR2UNCX145 - $2725:
```
APC Smart-UPS X, Line Interactive, 3kVA, Rack/tower convertible 2U, 100V-127V, 3x 5-15R+3x 5-20R+1x L5-30R NEMA,NMC, Extended runtime,10ft input cord
```

> **ALERT** The cheaper one has "Network management card 2 with environmental monitoring". We don't want the old card.

We can also get an [external battery](https://www.apc.com/us/en/product/SMX120RMBP2U/apc-smartups-x-external-battery-pack-rack-tower-2u-120vdc-w-rail-kit/).

#### PDU (ORDERED)

The [UniFi PDU Hi-Density](https://store.ui.com/us/en/category/all-power-tech/collections/power-tech/products/usp-pdu-hd?variant=usp-pdu-hd) has a L5-30 which is what the above UPS could supply. It's expensive but I like the ability to toggle each receptacle individually as well as monitor power from each receptacle. 

#### Power Density BOM

| Item | Quantity | Total $ |
| 42U Rack AR2400FP1 | 1 | $1500 (tax+delivery included) | 
| 2U APC UPS SMX3000RMLV2UNC | 1 | $3075 |
| 2U APC External Battery SMX120RMBP2U | 1 | $1200 |
| UniFi PDU | 1 | $1000 |

So ~$8k for rack & power

### KVM (ORDERED)

[PiKVM V4](https://www.pishop.us/product/pikvm-v4-plus/) lets you control devices plugged into servers over the network. Even handles [wake on lan](https://docs.pikvm.org/wol/) if configured right. 

There is one catch, it only does one device. To expand you can use a more traditional KVM with a switch that the [PiKVM can control](https://docs.pikvm.org/tesmart/#:~:text=TESMART%20managed%20multiport%20KVM%20switch%C2%B6)! This is a [TESmart 4K UHD 16 Ports HDMI KVM Switch](https://www.amazon.com/dp/B07YJMXM8J?th=1).

More cables than what are included will be needed. No HDMI and only 8 USB. 

| Item | Quantity | Total $ |
| PiKVM V4 Plus | 1 | $384.95 | 
| TESmart 16x1 HDMI KVM Switch 16 Port Enterprise Grade Support 4K 60Hz | 1 | $385.99 |

## After Moving

### Office

[Dell 5K2K Thunderbolt Monitor / Hub](https://www.dell.com/en-us/shop/dell-ultrasharp-40-curved-thunderbolt-hub-monitor-u4025qw/apd/210-bmdp/monitors-monitor-accessories#techspecs_section) for my work laptop long term but short term to use in the basement until the storage unit is emptied.

[Razer Pro Type Ultra](https://www.razer.com/gaming-keyboards/Razer-Pro-Type-Ultra/RZ03-04110200-R3U1) for the office.

Or maybe the [Combo with the little mouse](https://www.amazon.com/Razer-Wireless-Mechanical-Keyboard-Portable/dp/B09KT8XW35?th=1)

### Basement Gaming Room

[Desk Counter Top](https://www.lowes.com/pd/Sparrow-Peak-FSC-Chevron-6-mm-6-ft-6-cm-6-in-x-30-in-x-1-5-in-Walnut-Stained-Straight-Acacia-Butcher-Block-Countertop/5013903825) from Loews. 


### Cameras

### Smart Home