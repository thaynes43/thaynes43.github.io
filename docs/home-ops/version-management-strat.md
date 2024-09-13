---
title: Version Management
permalink: /docs/home-ops/version-management/
---

We are a bit late to the game on this one but we need to figure out how to update shit. [This guide](https://www.howtogeek.com/devops/how-to-get-started-managing-a-kubernetes-cluster-with-portainer/) looks pretty slick for portainer but I need to do some research to figure out what the `GitOps` way is.

Well, the git ops way is just with a bot that updates the repo!

## Which Bot?

So far I have found two options that look good. Renovate and Dependabot, though I think both can also work. [This guide](https://docs.renovatebot.com/bot-comparison/) has a really in depth comparison of the two bots. Renovate having a dashboard is a huge plus for me. Dependabot has a bunch of venerability checks that may be valuable. 

### Renovate

- [Docs](https://docs.renovatebot.com/)
- [GitHub](https://github.com/renovatebot/renovate)

Some renovate stuff:

1. If you use regex you need https://github.com/joryirving/home-ops/blob/main/.github/renovate/customManagers.json5

### Dependabot

GitHub's Dependabot seems to be able to update the versions for you which is documented [here](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuring-dependabot-version-updates). I seem to see a lot of this going on. 

The starter guide is [here](https://docs.github.com/en/code-security/getting-started/dependabot-quickstart-guide).

