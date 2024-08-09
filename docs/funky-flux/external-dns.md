---
title: External DNS
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

```git
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   bootstrap/helmrepositories/helmrepository-bitnami.yaml
        new file:   bootstrap/kustomizations/kustomization-external-dns.yaml
        new file:   bootstrap/namespaces/namespace-external-dns.yaml
        new file:   external-dns/helmrelease-external-dns.yaml
        new file:   external-dns/sealedsecret-cloudflare-api-token.yaml
```

Helm did not like the chart version:

```
2024-08-08T00:51:16.939Z error HelmChart/external-dns-external-dns.flux-system - reconciliation stalled invalid chart reference: failed to get chart version for remote reference: no 'external-dns' chart with version matching '5.1.x' found
```

New version, `8.3.x` worked, but it didn't like this:

```
  Warning  InstallFailed        49s   helm-controller  Helm install failed for release external-dns/external-dns with chart external-dns@8.3.4: 1 error occurred:
           * CustomResourceDefinition.apiextensions.k8s.io "dnsendpoints.externaldns.k8s.io" is invalid: metadata.annotations[api-approved.kubernetes.io]: Required value: protected groups must have approval annotation "api-approved.kubernetes.io", see https://github.com/kubernetes/enhancements/pull/1111
```

And this is a real issue with external-dns. See [here](https://github.com/kubernetes-sigs/external-dns/issues/4657) for the guy asking for the release to be cut since the fix is there already! Next I tried `7.3.2` and it's much better:

```
thaynes@kubevip:~/workspace/external-dns$ helm list -a -n external-dns
NAME        	NAMESPACE   	REVISION	UPDATED                                	STATUS  	CHART             	APP VERSION
external-dns	external-dns	2       	2024-08-08 01:13:21.507616746 +0000 UTC	deployed	external-dns-7.3.2	0.14.1     
```

```
thaynes@kubevip:~/workspace/external-dns$ flux get kustomizations external-dns
NAME        	REVISION          	SUSPENDED	READY	MESSAGE                              
external-dns	main@sha1:3a10b46a	False    	True 	Applied revision: main@sha1:3a10b46a	
```

```
thaynes@kubevip:~/workspace/external-dns$ flux get helmreleases -n external-dns external-dns
NAME        	REVISION	SUSPENDED	READY	MESSAGE                                                                                       
external-dns	7.3.2   	False    	True 	Helm upgrade succeeded for release external-dns/external-dns.v2 with chart external-dns@7.3.2	
```

```
thaynes@kubevip:~/workspace/external-dns$ kubectl get pods -n external-dns -l app.kubernetes.io/name=external-dns
NAME                            READY   STATUS    RESTARTS   AGE
external-dns-6979678bd9-4ppck   1/1     Running   0          3m20s
```

All systems nominal!
