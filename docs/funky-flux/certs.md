---
title: Certs
permalink: /docs/funky-flux/certs/
---

We're going to need some certs to get rid of the annoying messages for http sites. For this we will install [cert-manager](https://geek-cookbook.funkypenguin.co.nz/kubernetes/ssl-certificates/cert-manager/)

## Adding cert-manager to Cluster

This has to just be an easy copy paste here. Once thing I changed and will change in the others is:

Delete:

```yaml
  valuesFrom:
  - kind: ConfigMap
    name: cert-manager-helm-chart-value-overrides
    valuesKey: values.yaml # This is the default, but best to be explicit for clarity
```

Add:

```yaml
  values: # pasted contents of upstream values.yaml below, indented 4 spaces
    # ... all of the values ...
```

I also updated the chart to [1.15.x](https://artifacthub.io/packages/helm/cert-manager/cert-manager/1.15.2) as `1.15.2` looks to be the latest.

## Configuring cert-manager for DNS01 from Cloudflare  Cert Issuer 

First part was easy but now it's heating up a bit. [This guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/ssl-certificates/letsencrypt-issuers/) now goes into how to setup cert-manager to use Cloudflare but it doesn't go into making the sealed secret until a later guide so I'll bring that forward.

Also, when following this I noticed a dependency flag was added for `sealed-secrets`:

```yaml 
  dependsOn:
  - name: sealed-secrets
```

However, everything using this prior did not do this so I'll give it a shot everywhere that needs a sealed secret. Details for the Cloudflare token were [here](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/).

I guess we didn't need the secret yet but I ended up running this and throwing the output file (sealedsecret-cloudflare-api-token-secret.yaml) in the cert-manager folder since it felt weird to be missing the secret that was configured.

```bash
  kubectl create secret generic cloudflare-api-token-secret \
  --namespace cert-manager \
  --dry-run=client \
  --from-literal=api-token=REDACTED -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem \
  > sealedsecret-cloudflare-api-token-secret.yaml
```

Finally, we test to see if it's good:

```
kubectl describe clusterissuer letsencrypt-prod
...
Status:
  Acme:
    Last Private Key Hash:  f4g8fTo5jjGBpxg4vwYcSPkGW2UQVFuKF0kEoLBTSP4=
    Last Registered Email:  test@gmail.com
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/1879879286
  Conditions:
    Last Transition Time:  2024-08-08T03:32:15Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

Looks ready to rock and roll!

## Creating a Wildcard Certificate

### Staging

After creating the secret, which we already did, we get right into making a "staging" cert which doesn't have the same rate limiting as the prod endpoint.

> The name of the cert in the example is wrong

To get the name I ran:

```
kubectl get certificate -A
```

Then I panicked for 2-4min waiting for the cert. Finally all systems were nominal:

```
kubectl describe certificate -n letsencrypt-wildcard-cert letsencrypt-wildcard-cert-example.com-staging
...
  Conditions:
    Last Transition Time:  2024-08-08T03:49:25Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2024-11-06T02:50:51Z
  Not Before:              2024-08-08T02:50:52Z
  Renewal Time:            2024-10-07T02:50:51Z
  Revision:                1
```

As I waited I checked the logs for `cert-manager` namespaced pods. They were terrifying! Bunch of "re-queueing", "failed to update", "no longer exists" logs but we nailed it:

```json
{"log":"I0808 03:49:25.544288       1 acme.go:233] \"certificate issued\" logger=\"cert-manager.controller.sign\" resource_name=\"letsencrypt-wildcard-cert-example.com-staging-1\" resource_namespace=\"letsencrypt-wildcard-cert\" resource_kind=\"CertificateRequest\" resource_version=\"v1\" related_resource_name=\"letsencrypt-wildcard-cert-example.com-staging-1-2727934737\" related_resource_namespace=\"letsencrypt-wildcard-cert\" related_resource_kind=\"Order\" related_resource_version=\"v1\"\n","stream":"stderr","time":"2024-08-08T03:49:25.544425649Z"}
```

### Prod

Next was to test the prod issuer, `letsencrypt-prod`, which I will not rush this time.

```
kubectl describe certificate -n letsencrypt-wildcard-cert letsencrypt-wildcard-cert-example.com
...
Status:
  Conditions:
    Last Transition Time:  2024-08-08T03:59:29Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2024-11-06T03:00:57Z
  Not Before:              2024-08-08T03:00:58Z
  Renewal Time:            2024-10-07T03:00:57Z
  Revision:                1

```

## Replicating the Secrets (Which Point to Certs)

### Secret Replicator

Now this is some jank shit but we need to replicate the certs (which are secrets) across namespaces so everyone can use them. For this we have the secret replicator tool. The [repo](https://github.com/kiwigrid/secret-replicator) is even archived! 

Once again I am going to skip the separate ConfigMap file since this dude decided that sucked and go right to the helmrelease with the configs.

It didn't work:

```
secret-replicator   52s   False   Helm install failed for release secret-replicator/secret-replicator with chart secret-replicator@0.6.0: template: secret-replicator/templates/serviceaccount.yaml:1:18: executing "secret-replicator/templates/serviceaccount.yaml" at <.Values.rbac.enabled>: nil pointer evaluating interface {}.enabled
```

Going to the official chart [here](https://github.com/kiwigrid/helm-charts/blob/master/charts/secret-replicator/values.yaml) I can see that funky penguin was trying to fix something and I don't think `latest` is a good thing to  pull.

I went with chart `0.5.0` and tag `0.1.1`, the versions before funky man's changes and things came up:

```
thaynes@kubevip:~/workspace/cert-manager$ kubectl get pods -n secret-replicator
NAME                                 READY   STATUS    RESTARTS   AGE
secret-replicator-7b9f696d99-m6sm4   1/1     Running   0          18s
```

Now we should see that our secrets cloned themselves:

```
kubectl get secrets -A | grep letsencrypt-wildcard-cer
letsencrypt-wildcard-cert   letsencrypt-wildcard-cert-example.com                      kubernetes.io/tls                     2      24m
letsencrypt-wildcard-cert   letsencrypt-wildcard-cert-example.com-staging              kubernetes.io/tls                     2      34m
```

Aaand they didn't...

Cert manager has a [guide](https://cert-manager.io/docs/devops-tips/syncing-secrets-across-namespaces/) on a more up to date way to replicate secrets. Plus the [last commit](https://github.com/kiwigrid/secret-replicator/commit/c16872661b5c1894097920d71ac2a13c803cb467) in this repo says not to use it so I'm going to ROLL BACK!


### Roll Back Secret Replicator

Funky man never says how to do this so I'm going to start be deleting the shit that was added for `secret-replicator`.

```git
$ git status
On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        deleted:    bootstrap/helmrepositories/helmrepository-kiwigrid.yaml
        deleted:    bootstrap/kustomizations/kustomization-secret-replicator.yaml
        deleted:    bootstrap/namespaces/namespace-secret-replicator.yaml
        deleted:    secret-replicator/helmrelease-secret-replicator.yaml
```

And like a boss everything is gone! That was easy.

### What to Use Now?

I am now presented with two choices. Either [kubernetes-reflector](https://github.com/emberstack/kubernetes-reflector) or [kubernetes-replicator](https://github.com/mittwald/kubernetes-replicator). 

I am going to go with reflector because it seems easy enough given the [cert-manager docs](kubectl get secrets -A | grep letsencrypt-wildcard-cer)

For my certs I just added reflection to `podinfo` to start since we'll be testing that end to end.

```yaml
  secretTemplate:
    annotations:
      reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
      reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "podinfo"  # Control destination namespaces
      reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true" # Auto create reflection for matching namespaces
      reflector.v1.k8s.emberstack.com/reflection-auto-namespaces: "podinfo" # Control auto-reflection namespaces
```

The rest were pretty generic files I copied from podinfo. 

However, nothing worked. Something got borked:

```
flux-system                     main@sha1:dbbb3b3f      False           False   Kustomization/flux-system/reflector dry-run failed: failed to create typed patch object (flux-system/reflector; kustomize.toolkit.f
luxcd.io/v1, Kind=Kustomization): .spec.validation: field not declared in schema
```

It does not like `validation: server` which was in podman, maybe because I used `v1` instead of `v1beta1` so I will just delete that and maybe look what it does later. I am also going to remove the values specified in the helmrelease and just rock the defaults all the way here.

In my thrashing I also set the helm repo name to `reflector` when this chart is in the `emberstack` repo. Hopefully this was the last mistake. 

And it worked!

```
thaynes@kubevip:~/workspace/cert-manager$ kubectl get secrets -A | grep letsencrypt-wildcard-cer
letsencrypt-wildcard-cert   letsencrypt-wildcard-cert-example.com                      kubernetes.io/tls                     2      76m
letsencrypt-wildcard-cert   letsencrypt-wildcard-cert-example.com-staging              kubernetes.io/tls                     2      86m
podinfo                     letsencrypt-wildcard-cert-example.com                      kubernetes.io/tls                     2      5m56s
podinfo                     letsencrypt-wildcard-cert-example.com-staging              kubernetes.io/tls                     2      5m56s
```

But now I have a weird version of `sealed-secrets` failing to start that isn't the tag I specified!!

```
thaynes@kubevip:~/workspace/cert-manager$ kubectl describe deployment sealed-secrets -n sealed-secrets
Name:                   sealed-secrets
Namespace:              sealed-secrets
CreationTimestamp:      Sat, 03 Aug 2024 21:22:37 -0400
Labels:                 app.kubernetes.io/instance=sealed-secrets
                        app.kubernetes.io/managed-by=Helm
                        app.kubernetes.io/name=sealed-secrets
                        app.kubernetes.io/version=v0.16.0
                        helm.sh/chart=sealed-secrets-1.16.1
                        helm.toolkit.fluxcd.io/name=sealed-secrets
                        helm.toolkit.fluxcd.io/namespace=sealed-secrets
Annotations:            deployment.kubernetes.io/revision: 2
                        meta.helm.sh/release-name: sealed-secrets
                        meta.helm.sh/release-namespace: sealed-secrets
Selector:               app.kubernetes.io/instance=sealed-secrets,app.kubernetes.io/name=sealed-secrets
Replicas:               1 desired | 1 updated | 2 total | 1 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:           app.kubernetes.io/instance=sealed-secrets
                    app.kubernetes.io/name=sealed-secrets
  Service Account:  sealed-secrets
  Containers:
   sealed-secrets:
    Image:      quay.io/bitnami/sealed-secrets-controller:v0.16.0
    Port:       8080/TCP
    Host Port:  0/TCP
    Command:
      controller
    Args:
      --key-prefix
      sealed-secrets-key
    Liveness:     http-get http://:8080/healthz delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:    http-get http://:8080/healthz delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /tmp from tmp (rw)
  Volumes:
   tmp:
    Type:          EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:        
    SizeLimit:     <unset>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    False   ProgressDeadlineExceeded
OldReplicaSets:  sealed-secrets-7fbdddf68 (1/1 replicas created)
NewReplicaSet:   sealed-secrets-f88f5fccb (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  33m   deployment-controller  Scaled up replica set sealed-secrets-f88f5fccb to 1
```

Fixed it with fiddling, was crazy.