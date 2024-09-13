---
title: VLAN Migration - Build Cluster
permalink: /docs/vlan-migration/build-k3s-cluster/
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

As I saw the IPs from my new VMs I realized my `MetalLB` range was compromised and static IPs would be needed to keep things tidy. I decided to keep the range `192.168.40.6-192.168.40.39` for hosts and use `192.168.40.40-192.168.40.99` for VMs.

![static ip]({{ site.url }}/images/debian/static-ip.png)

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

###### Additional Entries

| pvedash | 192.168.40.49 | y |
| haos-pve | 192.168.40.50 | y |
| cephdash | 192.168.40.51 | y |

**TOWERVMS**

Using the end of my range for things not on proxmox

| haos | 192.168.40.90 | y |
| pbs | 192.168.40.91 | y |

Debian static IP

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
        address 192.168.40.48/24
        gateway 192.168.40.1
```

    server haynesintelligence haynesintelligence.example:8006 check ssl verify none cookie haynesintelligence

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
sudo hostnamectl
```

#### HA Load Balancer

[This guide](https://www.the38.page/posts/high-availability-k3s-cluster-setup/) is exactly what I'm going for and a lot more detailed than [funky mans](https://geek-cookbook.funkypenguin.co.nz/kubernetes/cluster/k3s/) (and no  emoji's that make this stuff look AI generated).

Setting up the config was easy:

```bash
# Uncomment this next line if you are NOT running nginx in docker
# By loading the ngx_stream_module module, Nginx can handle TCP/UDP traffic in addition to HTTP traffic
load_module /usr/lib/nginx/modules/ngx_stream_module.so;
events {}

stream {
  upstream kube {
    server kubem01.example:6443; # Change to the IP of the K3s first master VM
    server kubem02.example:6443; # Change to the IP of the K3s second master VM
    server kubem03.example:6443; # Change to the IP of the K3s third master VM
  }

  server {
    listen 6443;
    proxy_pass kube;
  }
}
```

But of course `nginx -t` threw errors:

1. I had to install `sudo apt install nginx-extras`
1. kubem03 was missing a DNS entry

But once sorted it ran fine:

```
root@kube:~# ss -antpl
State        Recv-Q       Send-Q              Local Address:Port               Peer Address:Port       Process                                                        
LISTEN       0            100                     127.0.0.1:25                      0.0.0.0:*           users:(("master",pid=3904,fd=13))                             
LISTEN       0            511                       0.0.0.0:6443                    0.0.0.0:*           users:(("nginx",pid=6246,fd=3),("nginx",pid=6245,fd=3))       
LISTEN       0            128                       0.0.0.0:22                      0.0.0.0:*           users:(("sshd",pid=4606,fd=3))                                
LISTEN       0            100                         [::1]:25                         [::]:*           users:(("master",pid=3904,fd=14))                             
LISTEN       0            128                          [::]:22                         [::]:*           users:(("sshd",pid=4606,fd=4))                                
```

#### First Init of Cluster

First `sudo apt get curl` on everything...

**EXAMPLE**

```bash
K3S_TOKEN=SECRET
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - server \
    --disable traefik --disable servicelb \
    --cluster-init \
    --tls-san=<LOAD_BALANCER_HOST_OR_IP> \
    --node-taint CriticalAddonsOnly=true:NoExecute
```

**ACTUAL**

```bash
sudo curl -sfL https://get.k3s.io | K3S_TOKEN=REDACTED sh -s - server \
    --disable traefik --disable servicelb \
    --node-taint CriticalAddonsOnly=true:NoExecute \
    --cluster-init \
    --tls-san=kube.example \
    --tls-san=kubem01.example \
    --tls-san=kubem02.example \
    --tls-san=kubem03.example              
```

`thaynes@kubem01:~$ sudo /usr/local/bin/k3s-uninstall.sh` when it goes to hell...

And it's good!

```
thaynes@kubem01:~$ sudo k3s kubectl get nodes
NAME      STATUS   ROLES                       AGE   VERSION
kubem01   Ready    control-plane,etcd,master   31s   v1.30.3+k3s1
```

Extract token from `sudo cat /var/lib/rancher/k3s/server/node-token`:

#### Join Masters

**EXAMPLE**

```bash
K3S_TOKEN=SECRET
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - server \
    --disable traefik --disable servicelb \
    --server https://<ip or hostname of server1>:6443 \
    --tls-san=<LOAD_BALANCER_HOST_OR_IP> \
    --node-taint CriticalAddonsOnly=true:NoExecute
```

**ACTUAL**

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="REDACTED::server:REDACTED" sh -s - server \
    --disable traefik \
    --disable servicelb \
    --node-taint CriticalAddonsOnly=true:NoExecute \
    --server https://kubem01.example:6443 \
    --tls-san=kube.example \
    --tls-san=kubem01.example \
    --tls-san=kubem02.example \
    --tls-san=kubem03.example
```

#### Join Workers

**EXAMPLE**

```bash
K3S_TOKEN=SECRET
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET \ 
    sh -s - agent --server https://<ip or hostname of server>:6443
```

**ACTUAL**

These guys need a token:

```
sudo cat /var/lib/rancher/k3s/server/node-token
REDACTED::server:REDACTED
```

```
curl -sfL https://get.k3s.io | K3S_URL=https://kube.example:6443 K3S_TOKEN="REDACTED::server:REDACTED" sh -
```

#### Wrapping Up K8S

After that I copied the k3s config to `~/.kube/config` and configured it to use the load balancer IP. Then make it official with `sudo chmod 600 ~/.kube/config && export KUBECONFIG=~/.kube/config`.

> **NOTE** This IP seems to get overwritten when `k3s.yaml` changes, so if you don't do the KUBECONFIG env var you can probably just change everything in the k3s.yaml too with `sudo nano /etc/rancher/k3s.yaml`.

I also [enabled kubectl auto complete](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#optional-kubectl-configurations-and-plugins) for both the command and the alias `k`.

### k3s config.yaml

Later I learned that some extra configs were needed. These can be added when creating the cluster. just create a file `/etc/rancher/k3s/config.yaml` and add the configuration overrides needed. For now I am using:

```yaml
write-kubeconfig-mode: "0644"
kube-apiserver-arg:
  - "oidc-issuer-url=https://authentik.example.com/application/o/kube-apiserver/"
  - "oidc-client-id=kube-apiserver"
  - "oidc-username-claim=email"
  - "oidc-groups-claim=groups"
  - "oidc-groups-prefix=oidc:"
```

k3s.yaml is still giving me permissions trouble, [this doc](https://thriveread.com/unable-to-read-etc-rancher-k3s-k3s-yaml-error/) has a slew of messy ideas.


### Installing Previously Installed Dependencies 

Time to get cooking. Fortunately these are mostly fresh in my mind.

#### Installing Sealed Secrets CLI

I remembered this one though it was broken in the guide:

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/kubeseal-0.27.1-linux-amd64.tar.gz
tar -xvf kubeseal-0.27.1-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

#### Installing Velero CLI

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.14.0/velero-v1.14.0-linux-amd64.tar.gz
tar -xvf velero-v1.14.0-linux-amd64.tar.gz
sudo install -m 755 velero /usr/local/bin/velero
```

#### Installing Flux

```
curl -s https://fluxcd.io/install.sh | sudo bash
```

### Installing Things Discovered Later

Just to keep everything on one page if, incase I need a new cluster, anything installed later will be added below:

#### Install Krew & Plugins

Check the [krew docks](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) on how to install this. Then install a plugin called [kubelogin](https://github.com/int128/kubelogin).

Then:

```bash
kubectl krew install oidc-login
kubectl krew install whoami
kubectl krew install browse-pvc
```

#### .bashrc

For reference here is what I accumulated in .bashrc doing this. I recommend adding when each doc recommends and just referencing this to see if anything was missed.

```bash
# Custom K8s Stuff
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
source /usr/share/bash-completion/bash_completion
export KUBECONFIG=~/.kube/config
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

#### AWS Cli

[Follow this](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```