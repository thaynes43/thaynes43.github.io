---
title: Flux
permalink: /docs/funky-flux/invidious/
---

## What Is This?

[Invidious](https://invidious.io/) seems to be a way to host your own front end to YouTube. It seems kind of [intense](https://docs.invidious.io/installation/) to host but YouTube is a common thing in my house and might be some fun for the family.

I found this though our funky friend [here](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/invidious/) which doesn't look to bad.

One thing that sticks out in this guide is the following:

```yaml
  podAnnotations: 
    backup.velero.io/backup-volumes: backup
    pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "PGPASSWORD=$POSTGRES_PASSWORD pg_dump -U postgres -d $POSTGRES_DB -h 127.0.0.1 > /scratch/backup.sql"]'
    pre.hook.backup.velero.io/timeout: 3m
    post.hook.restore.velero.io/command: '["/bin/bash", "-c", "[ -f \"/scratch/backup.sql\" ] && PGPASSWORD=$POSTGRES_PASSWORD psql -U postgres -h 127.0.0.1 -d $POSTGRES_DB -f /scratch/backup.sql && rm -f /scratch/backup.sql;"]'
```

I have a few things that use postgres already so I'm wondering if this is missed by just letting velero do it's thing. 

> **TODO** Look into whatever he is doing here