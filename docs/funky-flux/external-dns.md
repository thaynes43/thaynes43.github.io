---
title: Backup Strategy for Everything (so far) 
permalink: /docs/funky-flux/external-dns/
---

[external-dns](https://github.com/kubernetes-sigs/external-dns) will let me manage DNS entries in Cloudflare via my flux repo which sounds most excellent. I assume later one we will add ingress to reverse proxy these like we are doing with NPM on unRAID right now.

## Adding to Flux-Repo

I am going to follow the [funky cookbook](https://geek-cookbook.funkypenguin.co.nz/kubernetes/external-dns/) pretty closely for this so expect a lot of copy pasta.

I took the chart from [here](https://github.com/bitnami/charts/blob/main/bitnami/external-dns/values.yaml) which was sha 773fcd7b74b7231483b12f25c65a291a52cc2e9c from 8/6/2024

One call out is we are putting it in a mode where all DNS entries will be defined in CRD but we could let services or other things add them later:

```
        sources:
          - crd
          # - service
          # - ingress
          # - contour-httpproxy
```

Now the annoying [sealed secret](https://geek-cookbook.funkypenguin.co.nz/kubernetes/sealed-secrets/):

```
  kubectl create secret generic cloudflare-api-token \
  --namespace external-dns \
  --dry-run=client \
  --from-literal=cloudflare_api_token=REDACTED -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  > sealedsecret-cloudflare-api-token.yaml
```

And off to the races:

