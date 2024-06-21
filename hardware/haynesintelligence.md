---
title: HaynesIntelligence 
permalink: /hardware/haynesintelligence/
---

I wanted to play with GPUs for LLMs. My main goal was to have a local LLM integrated into [Home Assistant](https://www.home-assistant.io/) so I could do my own thing without paying API fees or upcoming voice assistant subscriptions. I decided to do a Threadripper build with Dual 3090s.  

## Part List

### Itemized

This one wasn't going to be cheap...

| Item | Price | Quantity | Total | Note | 
| AMD Ryzen™ Threadripper™ 7960X | $1400 | 1 | $1400 | Probably the lest expensive Threadripper I could find. |
| ASRock TRX50 WS | $780 | 1 | $780 | Could support 3 GPUs + 10Gb NIC |
| NEMIX RAM 256GB  | $1240 | 1 | $1240 | ECC may have been over kill but this was a proper server | 
| CX4712 Sliger | $890 | 1 | $890 | I added cooling and rails | 
| Zotac GeForce RTX 3090 24GB Trinity OC | $810 | 2 | $1620 | eBAY specials | 
| NVIDIA NVLink HB Bridge 4-Slot for 3090 SLI | $133 | 1 | $133 | eBAY, mixed reviews on if it helps w/ training | 
| XFX Speedster SWFT319, Radeon™ RX 6800 | $370 | 1 | $370 | Had to throw this in to have a chance at hackintosh GPU passthrough |
| Seasonic Prime TX-1600 | $577 |  1 | $577 | My wall can't handle this yet but the new house will have a 30A UPS | 
| 3 Way 4 Pin PWM Fan Splitter | $9 | 2 | $18 | These 4U cases can be tricky with fans | 

The case has 10 tray less 3.5" HDD bays plus a variety of other storage space so I also had some HDDs on the mind for a ZFS array. The TRX50 motherboard had two [SFF-8654 4](https://www.amazon.com/JSER-SFF-8654-38pin-Target-Fanout/dp/B073GRZJ7X?th=1) plugs so I could easily rig up the 10 drives while still using. However, I wasn't going to pull the trigger there until I saw it so I'd know cable length for power etc. I also still need 12 drives to fill [HaynesTower]([nvme cooler]({{ site.url }}/hardware/haynestower/)) but only six of those could add to the unRAID Array itself so it's unlikely I'd get six others just for a smaller ZFS array. 

### Description

#### AMD Ryzen™ Threadripper™ 7960X

I needed a lot of PCIE lanes to run 3 GPUs, well maybe not for AI, but I didn't want to be out of luck if I pivoted to gaming VMs after a failed attempt there. The Threadripper Pro's had the most but got incredibly expensive. I grabbed a the [7960X](https://www.amd.com/en/support/downloads/drivers.html/processors/ryzen-threadripper/ryzen-threadripper-7000-series/amd-ryzen-threadripper-7960x.html) from [amazon](https://www.amazon.com/AMD-RyzenTM-ThreadripperTM-7960X-48-Thread/dp/B0CK2VGBSQ/ref=sr_1_1?crid=1ZTY9I3M8L220&dib=eyJ2IjoiMSJ9.sMacd-Cx6owFFn5Brtdl4jHMjwwgPG0JgDrLHKF_xcMDX96VpesgRN5BXB2zxm1xxSuTJcnkRvI1kuBw6qjDn-zJ3u1yulmYrh376MnYxRQ2Z5SRa5dGWdJpKre1mc8XwUSkuamfzAxJVW5xQqyslOd15HISzYsIqg6e9hGOc9BxdnCujqwhqPV9YBRSQ35Gr_hY5j5Xx31tWw1xBY1P73T7EPIPxhv_OdzyuL24CiE.-3s3td2XFmUmBlp_AYI-08N1WP1BdAWxDJXqgPFbN6A&dib_tag=se&keywords=amd+threadripper+7960x&qid=1717772540&sprefix=7960x%2Caps%2C136&sr=8-1&ufe=app_do%3Aamzn1.fos.765d4786-5719-48b9-b588-eab9385652d5) which could get me where I needed without dropping fu money.

![threadripper amazon]({{ site.url }}/images/builds/haynesai/threadripper-amazon.png)

#### ASRock TRX50 WS

After looking at every Asus worksatation motherboard I could find and constantly being reminded that people now hate Asus I was pleased to find the [TRX50](https://www.asrock.com/mb/AMD/TRX50%20WS/index.asp) which I got from [newegg](https://www.newegg.com/asrock-trx50-ws/p/N82E16813162153?Item=N82E16813162153). I'm excited that it not only can run the three GPUs but support plenty of HDDs for a future ZFS array via SlimSAS so no need to use a PCIE slot. 

![trx50ws]({{ site.url }}/images/builds/haynesai/trx50ws.png)

#### NEMIX RAM 256GB

Server memory is confusing and can get crazy expensive. I've never ran ECC before but I had the oppertunity here and I wanted this to be rock solid so, with some help from reddit, I picked up 256GB of NEMIX RAM on [newegg](https://www.newegg.com/nemix-ram-256gb/p/1X5-003Z-01FM5?Item=9SIA7S6K3T1197). This RAM is pretty sketchy but the good stuff is crazy expensive. Maybe eBAY would have been a better bet but time will tell.

![nemix ram]({{ site.url }}/images/builds/haynesai/nemix-ram.png)

#### CX4712 Sliger

This case looks awsome and I am super excited to get it. However, it is taking a lot longer than the other parts but I believe it's just from a small shop that does some custom or niche builds. I ordered right from [Sliger](https://www.sliger.com/products/rackmount/storage/cx4712/) seven days ago which means I've got three more before they recommend I inquire about the delay. 

I also accessorized the shit out of it:

* Matte White because I like to stare at computers
* The rails since this would fit nicely on top of HaynesTower at the new house when I get the 42U rack. 
* 6 [Noctua fans](https://www.amazon.com/Noctua-NF-A12x25-PWM-Premium-Quality-Quiet/dp/B07C5VG64V/ref=sr_1_3?crid=MPS5W8U8PN37&dib=eyJ2IjoiMSJ9.xIifzQO25_WlJxJeeYkIJk5flGZ9N53zXoztHrrgptwE0H6FGxHZ6YC7Lebhn5mP4MTBnBcqfx0M8mjtYQ-zkxuk0iPVFEceLF4qygeS-fGCC-UM_dzLs0p18B8UDlwX_HraZt3Q8GDTmwpgof8PTobwg02yFhVLi0e5Mf8QykWjDN-FSeadvwRmkvdCHpNE6M70cpStWLG29n9bNNZ9skO8IRstc4JMDpMMHCTRMcQ.wx8YU7zV--KDm-uTRsiHlhJXaLHMYOQ2z2mkhrNSaQA&dib_tag=se&keywords=noctua+120mm&qid=1718394861&sprefix=Noctu%2Caps%2C89&sr=8-3) for cheaper than what they sell for on Amazon
* Asetek 360mm which was an impulsive click but it's a cooler guarenteed to fit in the 4U case which saves me some trouble sizing one

![sliger cx4712]({{ site.url }}/images/builds/haynesai/sliger-cx4712.png)

#### Zotac GeForce RTX 3090 24GB Trinity OC

Everything I read said VRAM was king for LLMs and the cheapest path to that was used 3090s. I found some [ZOTAC](https://www.zotac.com/us/product/graphics_card/zotac-gaming-geforce-rtx-3090-trinity) ones on [eBay](https://www.ebay.com/itm/155181583891?epid=28044609256&itmmeta=01HZF7FJ68FQD36AX83AT8A1NC&hash=item24218b0e13:g:woIAAOSwqQpi8SfU&itmprp=enc%3AAQAJAAAA4OFABCiLAB74gcWkup0Q0Mpa1h8nqjLVHiZ87eP2LQ6r6X%2Fqg69ZneS%2F8LeIVp4c6zTZOS6Hlx8%2FtaymZ3CAOFylvXbmhSQXKY%2B3RV4ItUjRfOsEQdHkGFsrV%2BOK1oPnoMQVSuChR7LD3Eb2DVNQ7vUC1JrvJk%2BZl6W%2BHiPMrloBUdwRcA7q1SjkFZRJLtdI1JR1U0bi3PtfNh7Z8ayf3QXQ7wgVPpcbRa62iqAX7JAId6vsIHZp14wczA2%2FCaycK5yNc2ce7IGaBAmZJbtRGyN1Sfb1eesGfpPkfa9nbdq2%7Ctkp%3ABFBMtqO-5_tj) that fit the bill.

![zotec 3090]({{ site.url }}/images/builds/haynesai/zotec-3090.png)

#### NVIDIA NVLink HB Bridge 4-Slot for 3090 SLI

This wasn't strictly necessary but I figured "why not" since I was already running two 3090s next to each other. Reading feedback online never really got to the bottom of if these helped but some seemed to think it sped up training. I don't believe these are even manufactured anymore so it was another [eBay](https://www.ebay.com/itm/166809649098?itmmeta=01HZSMDEXGQSW8X09CTGSBSG7R&hash=item26d6a13fca:g:6~gAAOSwJ-xmYxUf&itmprp=enc%3AAQAJAAAA4Hv7QQDEFeBj6Ou78FSGKr3AdQBrDtYlCqvySG3VbKQMoUN78CBKCnA7smT3bqkJl0%2FJIoRNeOqSq%2Fa44UMj6pVzBctLNjLC6NITWMiQ8iAoJpTcSgqfFBlBxkiZs51ufxtp1TNxDnOg1ppsqiVj93a5dyxVxp8bKVVH5U1fEERe%2FbCIZtqq3XlNH75SKQlTusyOIt1kfz4SVUKxKvIrJiO8A1PxRZj6IaLAHBNeY0rpUl30zXVFslzp86DLuol5QFwCOIYzJGbzEjB61%2Ffsxisc839qAc%2BMUbE5VtpSlidI%7Ctkp%3ABFBM_u61tP5j) special.

![nvlink bridge]({{ site.url }}/images/builds/haynesai/nvlink-bridge.png)

#### XFX Speedster SWFT319, Radeon™ RX 6800

My goal starting out was to stick three GPUs into one build. After seeing the price of the NVidia GPUs with 24GB VRAM I decided to go with two. I then got into a fight with a hackintosh and realized, since I was going to slice and dice the AI server w/ proxmox I could use the x8 PCIE slot for a hackintosh GPU. I picked up the much more affordable RX 6800 on [amazon](https://www.amazon.com/dp/B09KW68M2G?psc=1&ref=ppx_yo2ov_dt_b_product_details)

![rx6800]({{ site.url }}/images/builds/haynesai/rx6800.png)

#### Seasonic Prime TX-1600

Another reason not to have three beefy 3090s running under load is you may need two power supplies. Since the RX 6800 is not going to be topping out much a 1600W should do the trick for my needs. I grabbed the Seasonic Prime off [Amazon](https://www.amazon.com/Seasonic-ATX-3-0-TX-1600-SSR-1600TR2/dp/B0C571LRNB/ref=asc_df_B0C571LRNB/?tag=hyprod-20&linkCode=df0&hvadid=693562313188&hvpos=&hvnetw=g&hvrand=6616365365788663353&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9001911&hvtargid=pla-2186571992283&mcid=93c4c8f53b183a8aabb7b7dd5037c639&gad_source=1&th=1).

Interestlingly I can only pull around 1440W from a 15A 120V breaker so this PSU may be able to blow a fuse. However, I will have a 30A 120V receptical and a 20A 120V receptical behind the server rack when I move in November which will be plenty of power for my needs.

![seasonic prime]({{ site.url }}/images/builds/haynesai/seasonic-prime.png)

## The Build

Coming Soon.. Still waiting on the case! 

## OS 

### Proxmox With a Twist

The proxmox boot usb I'd been using refused to boot for this build. When googling the error it sounded driver related and it was recommended I use Debian 12 and install Proxmox on top. Proxmox has an [official doc](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_12_Bookworm) on how to get that done.

#### Raid 0 with Debian

There are half a million guides on this, like [here](https://tuxfixer.com/debian-installation-with-lvm-raid-1-and-hot-spare/) I didn't have much luck with them, probably because I had no idea what I was doing. They were also all for `Raid 1` but here I'd prefer space to redundancy as I have plenty of redundent options and backups in place. I was thinking of this space more as the cache pool on [HaynesTower]]({{ site.url }}/hardware/haynestower/) (todo can link to cache pool info).

I ended up using the guided partition on one drive, then converting the largest partition it created to raid. Then I added a raid partition on the second drive. I then created an MD Raid 0 with both raid partitions and it seemed to work. 

#### No local-lvm Storage on Proxmox

I can't seem to use the partition Debian is installed on in Proxmox so I am circling back to make a smaller partition just for that and then split up the rest of the disk for Proxmox.



