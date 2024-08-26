---
title: VLAN Migration - Tech Debt
permalink: /docs/vlan-migration/tech-debt/
---

## What's Left

Things moved quick to get the cluster up and running, leaving behind some tech debt that needs resolving.

1. Three proxmox nodes are still on the Default network
1. HelmReleases can't be deployed because they include CRDs that need to be created by the chart
1. MINOR few secrets did not have `--format yaml` when sealed and are json
1. `Kustomizations` in the bootstrap folder are inconsistent. Ordering is different and some have `retryInterval: 1m` and `wait: true` while others don't.
1. Podinfo and Whoami were setup for tutorials but don't really work right now, either delete them or fix them.
1. Check `/etc/hosts/` and `/etc/resolve.conf` on VMs that changed subnets

### Move Everything Else to Hayneslab VLAN

#### Servers - 192.168.40.6-192.168.40.39

All hosts will fall within the range 192.168.40.6-192.168.40.39. Three in the proxmox cluster are left.

| pve01 | 192.168.40.6 | y |
| pve02 | 192.168.40.7 | y |
| pve03 | 192.168.40.8 | y |
| pve04 | 192.168.40.9 | y |
| pve05 | 192.168.40.10 | y |
| HaynesIntelligence | 192.168.40.11 | n |
| filet01-pve | 192.168.40.12 | n |
| filet02-pve | 192.168.40.13 | n |
| ---- | ---Not on Proxmox--- | ---- |
| HaynesTower | 192.168.40.39 | y |

##### HaynesIntelligence to VLAN

Should be easier because no ceph.

1. Roll out `/etc/pve/corosync.conf` changes with the anticipated IP, `192.168.40.11`, and restart it `systemctl restart corosync`. You will now be alone as a node.
1. Edit `/etc/hosts` and `/etc/resolv.conf`, change DNS and interfaces to `192.168.40.11` and THEN add VLAN for port of switch
1. `ifdown vmbr0; ifup vmbr0` for good measure
1. On node run `systemctl restart corosync` and `systemctl restart pve-cluster`
1. On every other node that are in the cluster run `systemctl restart corosync` and check with `cat /etc/pve/.members`
1. Reboot node

###### HaynesIntelligence VMs

FIRST update DNS entry or it gets weird.

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
        address 192.168.40.52/24
        gateway 192.168.40.1
```

Then refresh with `ifdown eth0; ifup eth0`

> **NOTE** Next time check if anyone is using the designated IP (looking at you Tower11). And SHUT EVERY VM DOWN OH MY GOD!

###### pre-filet01 & pve-filet02

These guys don't have the VPNLan setup yet as a secondary door to pve's dashboard but they do have three NICs so that's not bad.

When I initially set up the other VPNLan DNS entries I used a `-25` suffix to distinguish them but I think `-vpn` makes more sense. In this case the fitlet3's have 3 1Gbe NICs.

I will name the bridge `vmbr2` so it matches the other VPNLan bridges, for HA, but am skipping `vmbr1` for these (and for `HaynesIntelligence` which I had to go back and fix).

* https://pve-filet01-vpn.example:8006/
* https://pve-filet02-vpn.example:8006/

Now we need to move the hosts to:

| Node | IP |
| filet01-pve | 192.168.40.12 |
| filet02-pve | 192.168.40.13 |

###### Adjusted Plan

1. Bring down all the VMs that went crazy last time
1. Roll out `/etc/pve/corosync.conf` changes with the anticipated IP, `192.168.40.13`, new version number, and restart it `systemctl restart corosync`. You will now be alone as a node.
1. Edit `/etc/hosts` and `/etc/resolv.conf`, change DNS and interfaces to `192.168.40.13` and THEN add VLAN for port of switch
1. `ifdown vmbr0; ifup vmbr0` for good measure
1. On node run `systemctl restart corosync` and `systemctl restart pve-cluster`
1. On every other node that are in the cluster run `systemctl restart corosync` and check with `cat /etc/pve/.members`
1. Reboot node


Didn't go well, now there is no cluster. 
```bash
pvesh get /cluster/config/join --output-format json-pretty
hostname lookup 'pve04' failed - failed to get address info for: pve04: Name or service not known
```

I thought it might be because I forgot to bump the corosync version. Ran this on HaynesIntelligence and everything BUT HaynesIntelligence came back green.

```
Stop the cluster services:
systemctl stop pve-cluster
systemctl stop corosync

Force local mode to update the cluster configuration:
pmxcfs -l
```

Rebooting HaynesIntelligence than got it back online.

###### Last One Ever

`pve-filet02` with `nut01` is all that's left. 

1. Bring down all the VMs that went crazy last time
1. Roll out `/etc/pve/corosync.conf` changes with the anticipated IP, `192.168.40.13`, new version number, and restart it `systemctl restart corosync`. You will now be alone as a node.
1. Edit `/etc/hosts` and `/etc/resolv.conf`, change DNS and interfaces to `192.168.40.13` and THEN add VLAN for port of switch
1. Reboot and verify in UniFi new IP was retrieved
1. `systemctl restart corosync` once on filet01, observe that node showing up with the one we rebooted
1. Do pve01, wat for it to join pve-filet02 - good
1. Do HaynesIntelligence - now everything is frozen
1. BOOT IT

```bash
root@HaynesIntelligence:~# systemctl stop pve-cluster
root@HaynesIntelligence:~# systemctl stop corosync
root@HaynesIntelligence:~# pmxcfs -l
root@HaynesIntelligence:~# systemctl reboot
```

After booting HaynesIntelligence everything but pve04 became green. 

Then the NUT:

FIRST update DNS entry or it gets weird.

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
        address 192.168.40.80/24
        gateway 192.168.40.1
```

Reboot -> Now nut01 has this IP but `/etc/network/interfaces` is reverted.. good enough. Check USB to make sure mapping is good and do a victory lap!

###### Bring VMs Back Online

Still some work to do to get the VMs all up. First I can just boot up the ones on the correct subnet, this is mostly the k8s since I left the LXCs alone.

After I still have a big list to sift through. 

#### VMs & LXCs - 192.168.40.40-192.168.40.99

Critical functions and new VMs have been switched over but many are offline and will need static IP configuration.

| Node | IP | Configured? | 
| kubem01 | 192.168.40.40 | y |
| kubem02 | 192.168.40.41 | y |
| kubem03 | 192.168.40.42 | y |
| kubew01 | 192.168.40.43 | y |
| kubew02 | 192.168.40.44 | y |
| kubew03 | 192.168.40.45 | y |
| kubew04 | 192.168.40.46 | y |
| kubew05 | 192.168.40.47 | y |
| kube | 192.168.40.48 | y | 
| pvedash | 192.168.40.49 | y |
| haos-pve | 192.168.40.50 | y |
| cephdash | 192.168.40.51 | y |
| ---- | Skip to HaynesIntelligence | ---- |
| nas01 | 192.168.40.52 | y |
| nut02 | 192.168.40.53 | y |
| pop01 | 192.168.40.54 | y |
| WinGaming01 | 192.168.40.55 | n |
| WinGaming02 | 192.168.40.56 | n |
| ---- | Back to MS-01s | ---- |
| ubuntu01 | 192.168.40.57 | n |
| mac01 | 192.168.40.58 | y |
| mac02 | 192.168.40.59 | y |
| windows11-01 | 192.168.40.60 | n |
| windows11-02 | 192.168.40.61 | n |
| windows11-03 | 192.168.40.62 | n |
| WinGaming03 | 192.168.40.63 | n |
| ubundroid | 192.168.40.64 | n |
| waydroid01 | 192.168.40.65 | n |
| haynesdroid | 192.168.40.66 | n |
| ---- | --- filet02 --- | ---- |
| nut01 | 192.168.40.80 | y |
| ---- | ---Not on Proxmox--- | ---- |
| haos | 192.168.40.90 | y |
| pbs | 192.168.40.91 | y |
| Tower11VM | 192.168.40.92 | y |

#### MetalLB Annotations Wrong

I realized I was using the [MetalLB annotation](https://metallb.universe.tf/usage/#requesting-specific-ips) wrong where I put it under the helm release instead of the service. `loadBalancerIP` itself was working but is sketchy and [deprecated](https://github.com/kubernetes/kubernetes/pull/107235)

1. Annotate the services instead of the HelmReleases
1. Audit to see where `loadBalancerIP` is set and switch to only use the annotation
1. Just use ClusterIP for `vikunja` which will require the `truechart` for now since it's so complex

| Service | IP | LoadBalancerIP | Annotated | Doc | Verified |
| Traefik | 192.168.40.100 | no | yes | yes | yes |
| podinfo | 192.168.40.101 | no | yes | N/A | yes |
| Authentik | 192.168.40.102 | no | yes | yes | yes |
| open-webui | 192.168.40.103 | no | yes | yes | yes |
| mosquitto | 192.168.40.104 | fix! | fix! | N/A |
| zigbee2mqtt | 192.168.40.105 | N/A | yes | yes | yes |
| zwave-js-ui | 192.168.40.106 | no | yes | yes | yes |
| vikunja-redis | 192.168.40.107 | N/A | yes | yes |
| vikunja | 192.168.40.108 | N/A | yes | yes | yes |
| prowlarr | 192.168.40.109 | N/A | yes | yes | yes |
| radarr | 192.168.40.110 | N/A | yes | yes | yes |
| sabnzb-default | 192.168.40.111 | N/A | yes | yes | yes |
| sabnzb-mediarequests | 192.168.40.112 | N/A | yes |  yes | yes |
| sonarr | 192.168.40.112 | N/A | yes | yes | yes |

> **TODO** mosquitto gives no way to specify the IP. I can use a ClusterIP here and an IngressRoute to expose it to upstream dependencies. I can also use "http://mosquitto.iot-services.svc.cluster.local:9001" for in cluster routes.

[This post](https://www.reddit.com/r/kubernetes/comments/y916zw/kubernetesmetallb_are_assigned_loadbalancer_ip/) says you'll never lose an IP for a service which means I only screwed myself because I relocated, and therefore recreated, two services which snagged different IPs and conflicted with what I set prowlarr. However, still good to work out the annotation.

#### external-dns Dependency Problem

external-dns wouldn't start because the `DNSEndpoint` CRDs were not created so I'm going to split those into two kustomizations. Later I'll try the Unifi webhook sidecar and add another type of entry.

This wasn't bad at all, a template like this basically lets you add anything and tie it as a dependancy:

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: kustomization-name
  namespace: flux-system
spec:
  interval: 15m
  path: ./path/to/crds
  dependsOn:
    - name: define-what-you-want-it-to-wait-for
  prune: true
  timeout: 2m
  sourceRef:
    kind: GitRepository
    name: flux-system
```

#### Clean Up Certificates

For this I wanted everything to do with certificates under one namespace, `certificates`, and then reflector would shoot the secrets out to everyone.

I created a two step Kustomizataion to launch the stack. First `cert-manager` is installed (depending on `sealed-secrets`) and then the issuers, certs, and secrets are created. I also had reflector depend on `cert-manager` so they would be in place when it came up.

That turned out to be a harder than I thought to get working. 

1. CRDs made in the old namespace stopped the new HelmRelease from starting. I had to manually delete them.
1. ClusterIssuers had to be created before the Certificates so I had to add a third step in the dependancy chain
1. Challenges got stuck because `Error presenting challenge: error getting cloudflare secret: secrets "cloudflare-api-token-secret" not found` - need to re-seal namespace into secret, remove the stack kustomization, and add it back
1. Delete old secrets since I renamed them, though I think `flux reconcile helmrelease -n <namespace> <name of helmrelease>` would have gotten things unstuck

After all that the challenges are rolling in:

```
thaynes@kubem01:~/workspace$ k get challenges -A
NAMESPACE      NAME                                                              STATE     DOMAIN                    AGE
certificates   certificate-example.com-1-1856466766-2095207381             pending   example.com         89s
certificates   certificate-example.com-1-1856466766-2980001469                       example.com         88s
certificates   certificate-example.com-staging-1-2739062974-1720542754     valid     example.com         89s
certificates   certificate-example.com-staging-1-2739062974-2508974029               example.com         88s
certificates   certificate-local.example.com-1-112412607-1491899551        pending   local.example.com   88s
certificates   certificate-local.example.com-1-112412607-1603840806                  local.example.com   88s
certificates   certificate-local.example.com-staging-1-357645-2914321792             local.example.com   87s
certificates   certificate-local.example.com-staging-1-357645-2987784459             local.example.com   87s
```

Hope we make it across the finish line!

```
thaynes@kubem01:~/workspace$ k get challenges -A
No resources found
```

All good.

#### Traefik's Catch 22 

Traefik has the same problem external-dns where it needs to create CRDs when first dropped into a cluster before a kustomization references them. There are two that I use now but each are a bit different.

1. Middleware - this is a fairly common resource that can really be added right after traefik is deployed.
1. IngressRoute - this will depend on middleware as you specify which middleware to use in it. The big question is if IngresssRoutes should live with their respected services or together under traefik. First we need to see if traefik can deal with one for a service that does not yet exist.
1. External services - not really on traefik but I am using them to give traefik access for reverse proxy of ip/ports. 

This went well except a few missed namespaces. 

I also added `plex` in as a `DNSEndpoint` and `IngressRoute` but with a broken service name to make sure there was no dependancy on the service existing before it's created. If this was the case I'd have to put each `IngressRoute` with their services which I can see as a good idea but would rather keep the objects with the service that can use them.

First test I didn't change the host URL and it blew up.

Next try was good, it just refused to add the `IngressRoute` because the service wasn't found:

```
2024-08-13 23:51:16	2024-08-14T03:51:16Z ERR github.com/traefik/traefik/v3/pkg/provider/kubernetes/crd/kubernetes_http.go:102 > error="kubernetes service not found: traefik/haynestower" ingress=plex namespace=traefik providerName=kubernetescrd
```

After pushing the fix the plex URL is back online!

#### ExternalDNS Stuff

1. tautulli is now in the wrong namespace but I don't want it to freak out so I probably should delete all records, remove it, then add it back.
1. overseerr needs an entry incase I need to rebuild