---
title: VLAN Migration - K8s Migration
permalink: /docs/vlan-migration/k8s-migration/
---

## Creating VMs

I had previously [documented](https://gist.github.com/thaynes43/6135cdde0b228900d70ab49dfe386f91#k8) how I set up debian VMs for kubernetes. The plan this time around will be slightly different. 

#### 3x Smaller Master Nodes

Specs: 4 CPU, 8 GB RAM, 64 GB Ceph Storage

![kubem specs]({{ site.url }}/images/proxmox/kubem-specs.png)

The steps were self explanatorily but I had [this checklist](https://i12bretro.github.io/tutorials/0191.html) up incase I got stuck.

#### 5x Workers Nodes

Specs: 12 CPU, 32 GB RAM, 128 GB Local Storage

![kubew specs]({{ site.url }}/images/proxmox/kubew-specs.png)

#### General Housekeeping

After logging in I added my user to sudoers:

```
su root 
nano /etc/sudoers
thaynes ALL=(ALL:ALL) ALL
```

Then I tried an `apt update` and `apt upgrade` but it was up to date.

Plus the default power settings need some work...

![power settings]({{ site.url }}/images/debian/power-settings.png)

Next up is setting DNS entries and then installing [nomachine](https://www.nomachine.com/) so I can copy paste things I find on the internet into the terminal!

##### DNS Entry Tracker

As I saw the IPs from my new VMs I realized my `MetalLB` range was compromised and static IPs would be needed to keep things tidy. I decided to keep the range `192.168.40.6-192.168.40.40` for hosts and use `192.168.40.41-192.168.40.99` for VMs.

![static ip]({{ site.url }}/images/debian/static-ip.png)

| Node | IP | Configured? | 
| kubem01 | 192.168.40.40 | f |
| kubem02 | 192.168.40.41 | f |
| kubem03 | 192.168.40.42 | f |
| kubew01 | 192.168.40.43 | f |
| kubew02 | 192.168.40.44 | f |
| kubew03 | 192.168.40.45 | f |
| kubew04 | 192.168.40.46 | y |
| kubew05 | 192.168.40.47 | f |

Mr. 43 and Mrs. 42 got into a mess but eventually it sorted itself out for this nice and neat set of VMs:

![vms ready]({{ site.url }}/images/unifi/vms-ready.png)

I will have to see if I can limit the IPs handed out via DHCP to a certain range so I don't have to static IP the hell out of the VMs and LXCs.

##### Nomachine

Easiest way I have found to do this is go on a browser in each VM and downlod the amd64 deb from [nomachine](https://www.nomachine.com/).

`sudo dpkg -i nomachine_8.13.1_1_amd64.deb` to install after.

Installation tracker:

- kubem01
- kubew01
- kubem02
- kubew02
- kubem03
- kubew03
- kubew04
- kubew05

Then I gotta log into each one which is brutal just to copy paste.

## Setting Up Kubernetes

### k3s This Time

I am going to follow [k3s](https://docs.k3s.io/datastore/ha-embedded) this time around and also see what [funky man](https://geek-cookbook.funkypenguin.co.nz/kubernetes/cluster/k3s/) did with it since I will need `--disable servicelb` to use `MetalLB` which I've already setup.

Before I start I want to make sure things are configured via hostname:

```bash
sudo hostnamectl set-hostname my-master-node
sudo hostnamectl set-hostname my-worker-node1
sudo hostnamectl set-hostname my-worker-node2
```

#### First Init of Cluster

```bash
K3S_TOKEN=SECRET
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - server \
    --disable traefik --disable servicelb \
    --cluster-init \
    --tls-san=<FIXED_IP> \
    --node-taint CriticalAddonsOnly=true:NoExecute
```

#### Join Masters

```bash
K3S_TOKEN=SECRET
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - server \
    --disable traefik --disable servicelb \
    --server https://<ip or hostname of server1>:6443 \
    --tls-san=<FIXED_IP> \
    --node-taint CriticalAddonsOnly=true:NoExecute
```
#### Join Workers

```bash
K3S_TOKEN=SECRET
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET \ 
    sh -s - agent --server https://<ip or hostname of server>:6443
```

#### Fix Credentials

Few options listed for this:

- Prefix your kubectl commands with k3s. i.e., kubectl cluster-info becomes k3s kubectl cluster-info
- Update your environment variables in your shell to set KUBECONFIG to /etc/rancher/k3s/k3s.yaml
- Copy `/etc/rancher/k3s/k3s.yaml to ~/.kube/config, which is the default location kubectl will look for

### GitOps Ceph Stuff

#### Rook Ceph

[Rook External](https://rook.io/docs/rook/v1.12/CRDs/Cluster/external-cluster/) was not easy to set up the first go at it. Funky man also has a [guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/persistence/rook-ceph/) for GitOps'in the full blown deployment so between those and my notes, plus the files on the previous cluster, I might have a chance.

### Fixing Sealed Secrets

I don't expect any Sealed Secrets to work because I did not use my own keypair as it was at the bottom of [this guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/sealed-secrets/) and they were set up when I got there. Now I can [generate one](https://github.com/bitnami-labs/sealed-secrets/blob/main/docs/bring-your-own-certificates.md) and use it. 

Will need this for the following commands:

```bash
kubeseal --fetch-cert \
--controller-name=sealed-secrets \
--controller-namespace=sealed-secrets \
> pub-cert.pem
```

> **TODO** Need to include the part about using the custom keypair in all the commands below. These were the ones I ran without it.

#### Fixing Traefik's Secret

First `nano traefik-dash-secret.yaml` and add:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: traefik-dashboard-auth
  namespace: traefik
type: Opaque
data:
  users: REDACTED
```

Then seal and copy paste to git repo:

```bash
cat traefik-dash-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-traefik-dashboard-auth.yaml
```

> **WARNING** I didn't end up using this but keeping here for reference as CF secrets are useful

First `nano traefik-cf-secret.yaml` and add:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-credentials
  namespace: traefik

type: Opaque
stringData:
  email: test@gmail.com
  apiKey: REDACTED
```

Then seal and copy paste to git repo:

```bash
cat traefik-cf-secret.yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  --format yaml > sealedsecret-cloudflare-api-credentials.yaml
```

#### Velero Sealed Secret

First `nano awssecret` and add:

```
[default]
aws_access_key_id = REDACTED
aws_secret_access_key = REDACTED
```

Then seal and copy paste to git repo:

```bash
kubectl create secret generic -n velero velero-credentials \
  --from-file=cloud=velero/awssecret \
  -o yaml --dry-run=client \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets > velero/sealedsecret-velero-credentials.yaml
```

#### External-DNS Secret

This one has no file to pass as it's one key:

```bash
  kubectl create secret generic cloudflare-api-token \
  --namespace external-dns \
  --dry-run=client \
  --from-literal=cloudflare_api_token=REDACTED -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  > sealedsecret-cloudflare-api-token.yaml
```

#### Cert Manager Secret

Basically the same as external-dns except for the name:

```bash
  kubectl create secret generic cloudflare-api-token-secret \
  --namespace cert-manager \
  --dry-run=client \
  --from-literal=api-token=REDACTED -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  > sealedsecret-cloudflare-api-token-secret.yaml
```