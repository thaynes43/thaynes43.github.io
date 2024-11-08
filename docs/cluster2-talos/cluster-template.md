---
title: Cluster Template
permalink: /docs/moving-day/cluster-template/
---

> **TODO** This is a page within the baremetal document and should be linked

## WTF DO I DO

Notes are from following along with the instructions [here](https://github.com/onedr0p/cluster-template). There is also an example of how to template with omni [here](https://github.com/siderolabs/omni-docs/blob/main/content/docs/reference/cluster-templates.md).  

Devcontainer seemed like an obvious win for home use since Docker Desktop is free there but not at work... Next up is configuring the template which appears tricky but I have a lot of config in Omni already.

This can be whatever I want:

```yaml
bootstrap_cluster_name: "haynes-cluster"
```

As for `bootstrap_schematic_id: ""` I am now stuck.

First we need the config for talosctl:

```yaml
context: haynes
contexts:
    haynes:
        endpoints:
            - https://haynes.omni.siderolabs.io
        auth:
            siderov1:
                identity: manofoz@gmail.com
```

```bash
thaynes@HaynesHyperion:~/.talos$ talosctl config merge config
renamed talosconfig context "haynes" -> "haynes-1"
```

## omnictl 

Trying [omnictl](https://omni.siderolabs.com/how-to-guides/install-and-configure-omnictl) first. They even have a [cluster example](https://github.com/siderolabs/contrib/tree/main/examples/omni) with Argo. 

Going to try `brew install siderolabs/tap/sidero-tools`

See [tap](https://github.com/siderolabs/homebrew-tap)

Now we need the config for omnictl:

```yaml
contexts:
    default:
        url: https://haynes.omni.siderolabs.io
        auth:
            siderov1:
                identity: manofoz@gmail.com
context: default
```

Need to "merge" that into the config:

```bash
thaynes`@HaynesHyperion:~/.talos/omni$ omnictl config merge config
renamed omniconfig context default -> default-1
thaynes@HaynesHyperion:~$ omnictl get cluster
NAMESPACE   TYPE      ID              VERSION
default     Cluster   talos-default   13
```

### Template

You can dump a template via `thaynes@HaynesHyperion:~$ omnictl cluster template export -c talos-default -o haynes-ops-template.yaml`

```yaml
kind: Cluster
name: haynes-ops
kubernetes:
  version: v1.31.1
talos:
  version: v1.8.0
---
kind: ControlPlane
machines:
  - 88d0b080-43be-11ef-9fe8-3b0f229ef000
patches:
  - idOverride: 400-talos-default-control-planes
    inline:
      cluster:
        allowSchedulingOnControlPlanes: true
---
kind: Workers
---
kind: Machine
systemExtensions:
  - siderolabs/intel-ucode
  - siderolabs/nut-client
  - siderolabs/nvidia-container-toolkit-lts
  - siderolabs/nvidia-open-gpu-kernel-modules-lts
  - siderolabs/thunderbolt
name: 88d0b080-43be-11ef-9fe8-3b0f229ef000
patches:
  - idOverride: 400-cm-88d0b080-43be-11ef-9fe8-3b0f229ef000
    inline:
      machine:
        kernel:
          modules:
            - name: nvidia
              parameters:
                - NVreg_OpenRmEnableUnsupportedGpus=1
            - name: nvidia_uvm
            - name: nvidia_drm
            - name: nvidia_modeset
        network:
          hostname: talosm01
        sysctls:
          net.core.bpf_jit_harden: 1
```

You can import the change via:

```bash
omnictl cluster template sync -f haynes-ops-template.yaml
```

Any verify with:

```bash
omnictl cluster template status -f my-cluster-exported-template.yaml
```

But my settings for nut-client are gone!

```yaml
apiVersion: v1alpha1
kind: ExtensionServiceConfig
name: nut-client
configFiles:
  - content: |-
        MONITOR HaynesTowerUPS@haynestower.haynesnetwork:3493 1 thaynes REDACTED secondary
        SHUTDOWNCMD "/sbin/poweroff"
    mountPath: /usr/local/etc/nut/upsmon.conf
```

Maybe something like [backup templates](https://omni.siderolabs.com/how-to-guides/etcd-backups#cluster-templates) will help.


OK It's like this I think:

```yaml
kind: Cluster
name: haynes-ops
kubernetes:
  version: v1.31.1
talos:
  version: v1.8.0
---
kind: ControlPlane
machines:
  - 88d0b080-43be-11ef-9fe8-3b0f229ef000
patches:
  - idOverride: 400-talos-default-control-planes
    inline:
      cluster:
        allowSchedulingOnControlPlanes: true
---
kind: Workers
---
kind: Machine
systemExtensions:
  - siderolabs/intel-ucode
  - siderolabs/nut-client
  - siderolabs/nvidia-container-toolkit-lts
  - siderolabs/nvidia-open-gpu-kernel-modules-lts
  - siderolabs/thunderbolt
name: 88d0b080-43be-11ef-9fe8-3b0f229ef000
patches:
  - idOverride: 400-cm-88d0b080-43be-11ef-9fe8-3b0f229ef000
    inline:
      machine:
        kernel:
          modules:
            - name: nvidia
              parameters:
                - NVreg_OpenRmEnableUnsupportedGpus=1
            - name: nvidia_uvm
            - name: nvidia_drm
            - name: nvidia_modeset
        network:
          hostname: talosm01
        sysctls:
          net.core.bpf_jit_harden: 1
  - idOverride: 500-1b0a466b-81f0-4b2b-9701-5d48259c33d0
    annotations:
      name: User defined patch
    inline:
      apiVersion: v1alpha1
      configFiles:
        - content: |-
            MONITOR HaynesTowerUPS@haynestower.haynesnetwork:3493 1 thaynes REDACTED secondary
            SHUTDOWNCMD "/sbin/poweroff"
          mountPath: /usr/local/etc/nut/upsmon.conf
      kind: ExtensionServiceConfig
      name: nut-client
```

Nuke:
```bash
omnictl delete cluster talos-default
```

And boom:

```bash
torn down Clusters.omni.sidero.dev talos-default
destroyed Clusters.omni.sidero.dev talos-default
```

The machine and machine patches I setup persisted while all the cluster stuff is gone. I think I can just live with that since the password is there.

```bash
omnictl cluster template validate -f cluster.yaml
omnictl cluster template sync -f cluster.yaml --verbose
omnictl cluster template status -f cluster.yaml
```

## Act 2 - Starting with a dev container

Makes sense if I want to develop against the repo to have a dev container to do it. I don't need much more than what is in the template, just:

1. Need omnictl
1. Need my talos/omni/kubernetes config files

I added omni, now to try it:

```
>Dev Containers: Reopen
```

No dice:

```
[12536 ms] Start: Run in container: /bin/sh -c bash /workspaces/haynes-ops/.devcontainer/postCreateCommand.sh
[13056 ms] postCreateCommand failed with exit code 127. Skipping any further user-provided commands.
```

[12555 ms] postCreateCommand failed with exit code 127. Skipping any further user-provided commands.
[12557 ms] Error: Command failed: /bin/sh -c bash /workspaces/haynes-ops/.devcontainer/postCreateCommand.sh
[12557 ms]     at G7 (c:\Users\manof\.vscode\extensions\ms-vscode-remote.remote-containers-0.388.0\dist\spec-node\devContainersSpecCLI.js:235:130)
[12557 ms]     at async tm (c:\Users\manof\.vscode\extensions\ms-vscode-remote.remote-containers-0.388.0\dist\spec-node\devContainersSpecCLI.js:227:4483)
[12557 ms]     at async $w (c:\Users\manof\.vscode\extensions\ms-vscode-remote.remote-containers-0.388.0\dist\spec-node\devContainersSpecCLI.js:227:3828)
[12557 ms]     at async em (c:\Users\manof\.vscode\extensions\ms-vscode-remote.remote-containers-0.388.0\dist\spec-node\devContainersSpecCLI.js:227:3032)
[12557 ms]     at async GrA (c:\Users\manof\.vscode\extensions\ms-vscode-remote.remote-containers-0.388.0\dist\spec-node\devContainersSpecCLI.js:666:2752)
[12558 ms]     at async LrA (c:\Users\manof\.vscode\extensions\ms-vscode-remote.remote-containers-0.388.0\dist\spec-node\devContainersSpecCLI.js:665:8554)
[12558 ms]     at async c:\Users\manof\.vscode\extensions\ms-vscode-remote.remote-containers-0.388.0\dist\spec-node\devContainersSpecCLI.js:482:1190
[12567 ms] Exit code 1
[12567 ms] Command failed: C:\Users\manof\AppData\Local\Programs\Microsoft VS Code\Code.exe c:\Users\manof\.vscode\extensions\ms-vscode-remote.remote-containers-0.388.0\dist\spec-node\devContainersSpecCLI.js run-user-commands --user-data-folder c:\Users\manof\AppData\Roaming\Code\User\globalStorage\ms-vscode-remote.remote-containers\data --container-session-data-folder /tmp/devcontainers-23663f7f-83bf-4b1f-bb35-e62f28cb5fce1728442884984 --workspace-folder d:\labspace\haynes-ops --id-label devcontainer.local_folder=d:\labspace\haynes-ops --id-label devcontainer.config_file=d:\labspace\haynes-ops\.devcontainer\devcontainer.json --container-id 409f160677ef1fda99ee8a029365f3918b095c546b153af37d11bbf427337465 --log-level debug --log-format json --config d:\labspace\haynes-ops\.devcontainer\devcontainer.json --default-user-env-probe loginInteractiveShell --skip-non-blocking-commands false --prebuild false --stop-for-personalization true --remote-env REMOTE_CONTAINERS_IPC=/tmp/vscode-remote-containers-ipc-1faec904-6f88-4d5d-880d-d55050bee7a8.sock --remote-env DISPLAY=:0 --remote-env REMOTE_CONTAINERS_DISPLAY_SOCK=/tmp/.X11-unix/X0 --remote-env REMOTE_CONTAINERS=true --mount-workspace-git-root --terminal-columns 137 --terminal-rows 13 --dotfiles-target-path ~/dotfiles


### Act 2 - Update

Lots of progress with the dev container now. It is building as a github action and publishing to my ghco. I can hit the kubernetes cluster via kubectl no problem but omni is challenging.

I had to add `"int128/kubelogin!!?as=kubectl-oidc_login&type=script" \` to what was installed in the container to access my cluster since it requires auth. However, omnictl is released with other binaries and `"siderolabs/omni!!?as=omnictl&type=script"` pulls the wrong binaries. I found [this PR](https://github.com/jpillora/installer/pull/45#issuecomment-2403866231) which would let me specify which binaries using that tool but it appears to have gone stale. This is lower priority for now as I want to move onto flux.

Flux bootstrapping appears to require SOPS for handling it's dirty secrets. Next up we must dig in and understand wtf sops is.

### Act 2 - Update 2

devcontainer has omnictl now but no envvar for config and `Error: open /home/vscode/.config/omni/config: no such file or directory`. 