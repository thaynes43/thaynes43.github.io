---
title: VolSync With Templates
permalink: /docs/home-ops/volsync-with-templates/
---

After the last train wreck VolSync adventure in [Volsync]({{ site.url }}/docs/home-ops/volsync/)

TODO List:

1. External Secrets
1. Create template
1. Follow the rules:

```yaml
# This taskfile is used to manage certain VolSync tasks for a given application, limitations are described below.
#   1. Fluxtomization, HelmRelease, PVC, ReplicationSource all have the same name (e.g. plex)
#   2. ReplicationSource and ReplicationDestination are a Restic repository
#   3. Applications are deployed as either a Kubernetes Deployment or StatefulSet
#   4. Each application only has one PVC that is being replicated
```

1. Make sure we set `targetNamespace: setme` in `ks.yaml` so template stuff has the right namespace
1. Set template values in the `Kustomization` like this:

```yaml
  postBuild:
    substitute:
      APP: *app
      VOLSYNC_CAPACITY: 5Gi
```

> **NOTE** Will have to get rid of the stuff I am switching to the templates and re-create the volumes
