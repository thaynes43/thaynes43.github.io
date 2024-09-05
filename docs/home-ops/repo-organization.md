---
title: Repo Organization
permalink: /docs/home-ops/repo-organization/
---

Funky man did not follow the same patters as the wise k8s-at-home founders. Now I need to move everything around. [This post](https://github.com/fluxcd/flux2/discussions/1517#discussioncomment-867558) says you can't just move the damn folder so I gotta jump through hoops.

## Moving the Damn Folder

### Edits in the Repo

> **TODO** Nice folder structure diagrams

### Getting Flux Happy

`flux uninstall`

```bash
GITHUB_TOKEN=REDACTED \
flux bootstrap github \
  --owner=thaynes43 \
  --repository=flux-repo \
  --personal \
  --path "./kubernetes/main/flux" \
```

Spits this shit out

```
► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/thaynes43/flux-repo.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ component manifests are up to date
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
✔ public key: REDACTED
✔ configured deploy key "flux-system-main-flux-system-./kubernetes/main/flux" for "https://github.com/thaynes43/flux-repo"
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("d1be94775efd7848514f64bdb56b4277bbf17766")
► pushing sync manifests to "https://github.com/thaynes43/flux-repo.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for GitRepository "flux-system/flux-system" to be reconciled
✔ GitRepository reconciled successfully
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```
  
### Debugging

At first it looked good but the smb shares got stuck. `kubectl patch pvc {PVC_NAME} -p '{"metadata":{"finalizers":null}}'` took care of the pvc's and then the pv's disappeared despite being set to retain. However, no data was lost and I could just `flux reconcile kustomization smb-storage -n flux-system` them back to life.

Then Traefik lost it's helm installations and could not upgrade! Fortunately, it has no persistence, otherwise I'd have to jump through some major hoops but for now I'm going to try and flux dance it.

Still didn't work, some fucked up shit in there. 

```bash
thaynes@kubem01:~/workspace$ helm uninstall --namespace traefik traefik-external
release "traefik-external" uninstalled

thaynes@kubem01:~/workspace$ helm uninstall --namespace traefik traefik-internal
release "traefik-internal" uninstalled
```

Closer but stuck on a token I rolled back that hung around until I blasted this shit:

```
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  2m10s                 default-scheduler  Successfully assigned traefik/traefik-external-59496c4f8-9wl88 to kubew01
  Normal   Pulled     2m10s                 kubelet            Container image "quay.io/k8tz/k8tz:0.16.2" already present on machine
  Normal   Created    2m10s                 kubelet            Created container k8tz
  Normal   Started    2m10s                 kubelet            Started container k8tz
  Normal   Pulled     10s (x10 over 2m10s)  kubelet            Container image "docker.io/traefik:v3.1.2" already present on machine
  Warning  Failed     10s (x10 over 2m10s)  kubelet            Error: configmap "kubedash-auth-token" not found
```

That was a real issue though and was missed when I removed the secret. After deleting the env reference to it we're back in business!

Except Vikunja... Vikunja is fucked too but has persistence!

```bash
thaynes@kubem01:~/workspace$ k -n vikunja get helmrelease
NAME      AGE   READY   STATUS
vikunja   34m   False   Helm upgrade failed for release vikunja/vikunja with chart vikunja@15.2.8: "vikunja" has no deployed releases
```

Same gig as Traefik (what's up with these guys!)

```bash
thaynes@kubem01:~/workspace$ helm uninstall --namespace vikunja vikunja
release "vikunja" uninstalled
```

Now maybe we can Velero the release back?

```bash
velero create restore --from-backup velero-hayneslab-backups-20240904000019 --include-namespaces vikunja --wait
```

Seems kinda stuck. Maybe I can remove it, then re-add it so it starts clean, but then restore it to get the volumes. Gonna nuke it.

Gonna just let it time out and deal with this later, not sure why this one is so stuck but it's not essential.

> **WARNING** NEVER CHANGE THE PATH