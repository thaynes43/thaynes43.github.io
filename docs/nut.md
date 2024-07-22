---
title: NUT
permalink: /docs/nut/
---

## Motivation 

I have intended to configure Network UPS Tools for quite some time but have put it off. `HaynesTower` has been using unRAID's built in tools to do what the rest of these devices will need and has it's own UPS. A recent power outage corrupted my proxmox install for one of the MS-01 nodes. Recovery was a breeze but it's time to get nutty so this doesn't happen again.

I will be starting out with [this guide](https://technotim.live/posts/NUT-server-guide/). Then I'll take a look at monitoring the UPS with something like what is suggested [here](https://www.thesmarthomebook.com/2022/09/02/setting-up-monitor-your-ups-proxmox-home-assistant/).

## HaynesTower UPS

As I mentioned earlier, `HaynesTower` has had a UPS set up that only unRAID is looking at thanks to it's out of the box capabilities of connecting up to a APC UPS's USB port and setting up safe power off/on. At one point I had HomeAssistant monitoring it's power consumption but it has not been able to connect to it for a while. A few tricks were needed there that I will pull out for the PVE NUT:

### Broken Home Assistant Monitoring

`/config/packages/sensors.yaml`

```
template:
  - sensor:
      - name: "UPS Watt Load"
        unique_id: "apc_ups_watt_load"
        state_class: measurement
        device_class: power
        unit_of_measurement: "W"
        state: "{{ states('sensor.ups_load')|int  * 0.01 * states('sensor.ups_nominal_output_power')|int * 0.97|round(2) }}"
```

And this is loaded in `configurations.yaml` right at the top:

```
# Loads default set of integrations. Do not remove.
default_config:

homeassistant:
  packages: !include_dir_named packages
  auth_providers:
    - type: homeassistant
    - type: trusted_networks
      trusted_networks:
        - 192.168.0.0/24
        - 127.0.0.1
```

### unRAID NUT

In the spirt of consolodating everything to the nut-server / nut-client approach I will also set up a nut-server from unRAID. I can then use my new HAOS to monitor whatever number of UPS's I end up with. 

This was as easy as installing the Network UPS Tools plugin for unRAID, disabling the built in UPS stuff under `UPS Settings`, and configuring the new plugin to be something like this:

![nut settings]({{ site.url }}/images/unraid/nut-settings.png)

I will save all tests until I finish getting the rest of the servers configured but I don't think this one will be any trouble as the plugin was a breeze to setup. 

My dashboard card also automatically updated and looks a bit more minamalist now:

![nut card]({{ site.url }}/images/unraid/nut-card.png)

### HAOS NUT Monitoring

I still could not connect my HAOS VM on unRAID to the NUT monitoring API but my fresh install running on the proxmox cluster had no problem:

![unraid nut integration]({{ site.url }}/images/haos/unraid-nut-integration.png)

The NUT was no better about reporting power consumption:

![unraid ups]({{ site.url }}/images/haos/unraid-ups.png)

However, the install of HAOS I pulled that config from is a mess and has two `sensors.yaml` files, one in packages and the other as a sub-configuration file split from `configuration.yaml`. I don't remember which was the correct way to do it so I gave the UI a shot and found it was quite simple:

![ups load sensor]({{ site.url }}/images/haos/ups-load-sensor.png)

That even linked it upo to the NUT device as one of it's sensors which my manual sensor did not do before. Hopefully we can keep the improvements up as I migrate to the proxmox hosted HAOS for November's big move.

## Fitlet3 w/ USB Connected UPS

> TODO add a page in hardware 

My [APC UPS](https://www.amazon.com/dp/B08GRY1W93?psc=1&ref=ppx_yo2ov_dt_b_product_details) that all the MS-01's are connected to is has no network capabilities and is plugged in via usb to a [fitlet3](https://fit-iot.com/web/products/fitlet3/) mini pc thing which I've been accidently calling filet3. These are currently the smallest devices I've put proxmox on but the [s100](https://store.minisforum.com/products/minisforum-s100) would work well for this too if I needed to serve something connected via USB anywhere in throughout the house.

The plan is to setup an LXC with the nut-server on the fitlet3. For monitoring I may just use home-assistant but I'll explore what's out there. Then I will install nut-client on each host and configure them to safely power down

### USB Passthrough 

The LXC can see my USB devices right off the bat. People seem to have permission issues fixed with methods like [this](https://gist.github.com/crundberg/a77b22de856e92a7e14c81f40e7a74bd). I'll have to see if I have problems before I fiddle.

To be continued... 