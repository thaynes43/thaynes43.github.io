---
title: Terminal
permalink: /docs/terminal/
---

Going to up my terminal game with wsl. Need to [install WSL] and then [set up Ubuntu](https://learn.microsoft.com/en-us/windows/wsl/setup/environment#set-up-your-linux-username-and-password). Git bash is already ready to rock and role.

## Getting Started

### Kubernetes 

You need a `config` file in your `~/.kube` directory to access the cluster. After that we just need a few tools that allow us to manage it.

#### Install Kubectl

I used snap but anything is fine:

```bash
sudo snap install kubectl
```

Then followed [these steps](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) to set up autocomplete and the alias. 

#### Install Krew & Plugins

Check the [krew docks](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) on how to install this. Then install a plugin called [kubelogin](https://github.com/int128/kubelogin).

Then:

```bash
kubectl krew install oidc-login
kubectl krew install whoami
kubectl krew install browse-pvc
```

### Other CLI

#### Flux

```
curl -s https://fluxcd.io/install.sh | sudo bash
```

#### Sealed Secrets

I remembered this one though it was broken in the guide:

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/kubeseal-0.27.1-linux-amd64.tar.gz
tar -xvf kubeseal-0.27.1-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

#### Velero

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.14.0/velero-v1.14.0-linux-amd64.tar.gz
tar -xvf velero-v1.14.0-linux-amd64.tar.gz
sudo install -m 755 velero /usr/local/bin/velero
```

#### AWS

[Follow this](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```