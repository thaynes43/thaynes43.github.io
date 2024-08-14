---
title: VLAN Migration - Tech Debt
permalink: /docs/vlan-migration/tech-debt/
---

## What's Left

Things moved quick to get the cluster up and running, leaving behind some tech debt that needs resolving.

1. Three proxmox nodes are still on the Default network
1. HelmReleases can't be deployed because they include CRDs that need to be created by the chart
1. MINOR few secrets did not have `--format yaml` when sealed and are json

### Move Everything Else to Hayneslab VLAN

#### Servers - 192.168.40.6-192.168.40.39

All hosts will fall within the range 192.168.40.6-192.168.40.39. Three in the proxmox cluster are left.

| pve01 | 192.168.40.6 | y |
| pve02 | 192.168.40.7 | y |
| pve03 | 192.168.40.8 | y |
| pve04 | 192.168.40.9 | y |
| pve05 | 192.168.40.10 | y |
| filet01-pve | 192.168.40.11 | n |
| filet02-pve | 192.168.40.12 | n |
| HaynesIntelligence | 192.168.40.12 | n |
| ---- | ---Not on Proxmox--- | ---- |
| HaynesTower | 192.168.40.39 | y |

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
| todo | 192.168.40.52 | n |
| ---- | ---Not on Proxmox--- | ---- |
| haos | 192.168.40.90 | y |
| pbs | 192.168.40.91 | y |
| Tower11VM | 192.168.40.92 | n |

### Rearranged some stuff in flux-repo

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
certificates   certificate-haynesnetwork.com-1-1856466766-2095207381             pending   haynesnetwork.com         89s
certificates   certificate-haynesnetwork.com-1-1856466766-2980001469                       haynesnetwork.com         88s
certificates   certificate-haynesnetwork.com-staging-1-2739062974-1720542754     valid     haynesnetwork.com         89s
certificates   certificate-haynesnetwork.com-staging-1-2739062974-2508974029               haynesnetwork.com         88s
certificates   certificate-local.haynesnetwork.com-1-112412607-1491899551        pending   local.haynesnetwork.com   88s
certificates   certificate-local.haynesnetwork.com-1-112412607-1603840806                  local.haynesnetwork.com   88s
certificates   certificate-local.haynesnetwork.com-staging-1-357645-2914321792             local.haynesnetwork.com   87s
certificates   certificate-local.haynesnetwork.com-staging-1-357645-2987784459             local.haynesnetwork.com   87s
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
2024-08-13 23:51:16	2024-08-14T03:51:16Z ERR github.com/traefik/traefik/v3/pkg/provider/kubernetes/crd/kubernetes_http.go:102 > error="kubernetes service not found: traefik/haynestowerffffffffff" ingress=plex namespace=traefik providerName=kubernetescrd
```

After pushing the fix the plex URL is back online!

#### ExternalDNS Stuff

1. tautulli is now in the wrong namespace but I don't want it to freak out so I probably should delete all records, remove it, then add it back.
1. overseerr needs an entry incase I need to rebuild