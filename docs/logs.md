---
title: Logs
permalink: /docs/logs/
---

## Grafana Loki

Starting with [this writeup](https://grafana.com/blog/2023/04/12/how-to-collect-and-query-kubernetes-logs-with-grafana-loki-grafana-and-grafana-agent/) we are given a good idea at why this is a good idea. Plus I need to debug some stuff that I lose logs for when pods restart.

### Roadblocks

Now that I am using [flux](https://fluxcd.io/flux/get-started/) all the steps need to be adapted to follow how this repository structures the helm stuff. I may want to run though a [few more examples](https://geek-cookbook.funkypenguin.co.nz/kubernetes/) or pivot to some ansible shit.