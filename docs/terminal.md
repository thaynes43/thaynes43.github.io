---
title: Terminal
permalink: /docs/terminal/
---

Going to up my terminal game with wsl. Need to [install WSL] and then [set up Ubuntu](https://learn.microsoft.com/en-us/windows/wsl/setup/environment#set-up-your-linux-username-and-password). Git bash is already ready to rock and role.

## Getting Started

### Managed PC

If your org is managing your PC curl may not have certs and nothing will install. Follow [these steps](https://github.com/bayaro/windows-certs-2-wsl) to map the certs from windows.

### Homebrew

Follow the [official steps](https://brew.sh/) and get homebrew setup.

#### Siderolabs Tap

Follow [these instructions](https://github.com/siderolabs/homebrew-tap) to install `talosctl`, `omnictl`,  `kubectl-oidc_login`, and `kubectl`.

```bash
brew tap siderolabs/tap
brew install talosctl omnictl kubectl-oidc_login kubectl
```

### Kubernetes 

You need a `config` file in your `~/.kube` directory to access the cluster. After that we just need a few tools that allow us to manage it.

#### Alt Install Kubectl

If you did't use the Siderolabs tap you can install kubectl like this:

```bash
sudo snap install kubectl --classic
```

Then followed [these steps](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) to set up autocomplete and the alias. Or just do what is below:

```bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
```

#### Install Krew & Plugins

Check the [krew docks](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) on how to install this. Then install a plugin called [kubelogin](https://github.com/int128/kubelogin).

> **NOTE** Make sure to add `export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"` to `.bashrc` and not just in the active terminal

Then:

```bash
# kubectl krew install oidc-login we are using homebrew for this now
kubectl krew install whoami
kubectl krew install browse-pvc
```

### Other CLI

#### Flux

```
curl -s https://fluxcd.io/install.sh | sudo bash
```

#### Sealed Secrets - NOT USED ANYMORE

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/kubeseal-0.27.1-linux-amd64.tar.gz
tar -xvf kubeseal-0.27.1-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

#### Velero - NOT USED ANYMORE

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.14.0/velero-v1.14.0-linux-amd64.tar.gz
tar -xvf velero-v1.14.0-linux-amd64.tar.gz
sudo install -m 755 velero /usr/local/bin/velero
```

#### AWS

[Follow this](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

> **Pre-req** you may need to `sudo apt install unzip` first

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```