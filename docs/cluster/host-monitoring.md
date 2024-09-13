---
title: Host Monitoring
permalink: /docs/host-monitoring/
---

## Glances

Seeing how how [HaynesIntelligence]({{ site.url }}/hardware/haynesintelligence/) could get motivated me to expose some sensor data so I could keep an eye on things.

With the long term goal of viewing host metrics in Grafana I first needed something that could scrape them. [Glances](https://nicolargo.github.io/glances/) looked like something simple that was worth a try. It even had a native [Grafana dashboard](https://grafana.com/grafana/dashboards/2387-glances-for-flux/).

I found a [helper script](https://tteck.github.io/Proxmox/#glances) that was easy enough to run and get things viewable at a [http://192.168.0.250:61208](http://192.168.0.250:61208).

However, but difficult to tell what the temperature sensors represented without doing some more research (TODO that):

![hi temps]({{ site.url }}/images/builds/metrics/hi-temps.png)

I decided to install on an intel host to have something to compare against. Once again the script got the job done and I was presented with [http://192.168.0.34:61208](http://192.168.0.34:61208) to locally view my senor data. At least this time I got Core temps which AMD doesn't show:

![pve01 temps]({{ site.url }}/images/builds/metrics/pve01-temps.png)

More hosts to be done later but not worth the lift until I get an Influx DB and grafana instance running in k8s to point this data to.