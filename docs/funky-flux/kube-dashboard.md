---
title: Kubernetes Dashboard
permalink: /docs/funky-flux/kube-dashboard/
---

We have finally made it to [this guide](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/dashboard/) after quite the journey through dependencies.

## But Wait, There's More

Well, we have not made it. First we need an OAuth2 Proxy as defined in [this guide](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/oauth2-proxy/).

I don't understand why you'd need a separate `oauth2-proxy` Kustomization if no dependancy exists between it and the dashboard. Since it appears we spin one proxy up per app we want to use the ODIC stuff for I'm going to just do one Kustomization under flux for now.

### metadata

```yaml
metadata:
  name: oauth2-proxy
  namespace: kubernetes-dashboard
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.40.103
```

### config config

```yaml
config:
  clientID: kube-apiserver
  clientSecret: FROMAUTHENTIK
  cookieSecret: RANDOMSECRET
  configFile: |-
    email_domains = [ "*" ]
    upstreams = [ "http://kubernetes-dashboard" ]
    provider = "oidc"
    oidc_issuer_url = "https://authentik.example.com/application/o/kube-apiserver/"    
```

> **Note** provider & oidc_issuer_url were added after funky man did his doc

### Random extraArgs

YOLO'ing these in 

```yaml
    extraArgs:
      provider: oidc
      provider-display-name: "Authentik"
      skip-provider-button: "true"
      pass-authorization-header: "true" 
      cookie-refresh: 15m 
```

### Load Balancer

```yaml
    service:
      type: LoadBalancer
      loadBalancerIP: "192.168.40.103"
```

### Ingres

> **Note** you cannot have `enabled: true` and the `path` is singular unlike Authentik.

```yaml
    ingress:
      enabled: true
      className: traefik      
      path: /
      pathType: Prefix
      hosts:
        - kubedash.local.example.com
      labels: {}
      tls:
        - secretName: certificate-local.example.com
          hosts:
            - kubedash.local.example.com 
```

### DNS

> **TODO** We will hopefully replace the manual part with the `external-ens` UniFi webook

For now I manually added the DNS A entry in Unifi (from `kubedash.local.example.com` to `192.168.40.100`).

## Initial Setup

Now that the proxy is in place things seem pretty straight forward. 

From the [helm repo page](https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard/6.0.8) I can see the latest chart is `7.5.0` so I will ignore his complaints about breaking changes on master.

Unfortunately the complaints were valid and the dashboard has been completely split into multiple pods. None of the instructions in this guide or any others are valid. Good thing we leaned heavily on this OAuth2 Proxy.

`--enable-insecure-login` appears to be the only thing needed but I don't even think it is. I'm going to roll with: 

```yaml
values: values.yaml
```

## Testing

Didn't work

```
thaynes@kubem01:~$ kubectl logs -n kubernetes-dashboard oauth2-proxy-5b496f9ff5-xzwf4
Defaulted container "oauth2-proxy" out of: oauth2-proxy, wait-for-redis (init)
[2024/08/16 21:18:23] [main.go:58] ERROR: Failed to initialise OAuth2 Proxy: error initialising provider: could not create provider data: error building OIDC ProviderVerifier: invalid provider verifier options: missing required setting: issuer-url
```

Getting closer, no redirect though:

```json
	{"auth_via": "session", "domain_url": "authentik.example.com", "event": "Invalid redirect uri (regex comparison)", "host": "authentik.example.com", "level": "warning", "logger": "authentik.providers.oauth2.views.authorize", "pid": 18133, "redirect_uri_expected": ["http://localhost:8000"], "redirect_uri_given": "https://kubedash.local.example.com/oauth2/callback", "request_id": "ba67711357fc4baa8bf5f9dbfadb7f5e", "schema_name": "public", "timestamp": "2024-08-16T21:47:53.910641"}
```

Going to try:

```yaml
    extraArgs:
    ...
      redirect-url: "http://localhost:8000"
```

At some point I need to sync up on using `extraArgs` vs `configFile`.

Nope, need to add one in Authentik and then use it!

authentik-kubedash-redirect.png

```yaml
    extraArgs:
    ...
      redirect-url: "https://kubedash.example.com/oauth2/callback/"
```

Getting closer but now it just bounces right back:

```json
	{"action": "authorize_application", "auth_via": "session", "client_ip": "10.42.7.1", "context": {"authorized_application": {"app": "authentik_core", "model_name": "application", "name": "Hayneslab Kubernetes", "pk": "aa831c187bfd42cda3bddd95473c7455"}, "flow": "dfa84f6b3a1547119e49c2929d59d409", "http_request": {"args": {"approval_prompt": "force", "client_id": "kube-apiserver", "redirect_uri": "https://kubedash.example.com/oauth2/callback/", "response_type": "code", "scope": "openid email profile", "state": "_QNr3TEb1CXbYK9ccp0ubCEpYAP2Gjw9cELv0-NcAWI:/"}, "method": "GET", "path": "/api/v3/flows/executor/default-provider-authorization-implicit-consent/", "request_id": "6cfae8591e4a4553b37d9a817dafb85e", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36 Edg/127.0.0.0"}, "scopes": "openid profile email"}, "domain_url": "authentik.example.com", "event": "Created Event", "host": "authentik.example.com", "level": "info", "logger": "authentik.events.models", "pid": 27183, "request_id": "6cfae8591e4a4553b37d9a817dafb85e", "schema_name": "public", "timestamp": "2024-08-17T04:10:11.608400", "user": {"email": "admin@example.com", "pk": 9, "username": "example"}}
```

> **IDEA** After setting some other stuff up I think I would have had better success if I set up a new Provider w/ it's own redirect for this OAuth2-Proxy

## PIVOT

I started digging around looking at stuff on [medium](https://medium.com/@mike.schouw/how-to-run-oauth2-proxy-with-traefik-in-kubernetes-using-helm-and-terraform-85c39dddcd44) and even some [bug reports](https://github.com/oauth2-proxy/oauth2-proxy/issues/1019) but this wasn't working. Then I found a [new approach](https://blog.cubieserver.de/2023/auth-proxy-with-authentik-and-traefik/) that looks a lot easier in the long run even. It just doesn't use ODIC but I really don't know what consequences that'll have yet. 

### Outpost Proxy for Kube Dash

Turns out I can just configure Authentik to take the place of `OAuth2-proxy` and it'll handle the auth and redirection. A few quick forms and I thought I was all set:

![authentik-kubedash-proxy-provider]({{ site.url }}/images/web/authentik-kubedash-proxy-provider.png)

![authentik-kubedash-proxy-app]({{ site.url }}/images/web/authentik-kubedash-proxy-app.png)

![authentik-add-proxy-to-outpost]({{ site.url }}/images/web/authentik-add-proxy-to-outpost.png)

However, I missed adding a new Outpost and got confused for a bit. But doing this made the confusion melt away:

![authentik-add-outpost]({{ site.url }}/images/web/authentik-add-outpost.png)

[This guy](https://artur.rocks/i-set-up-authentik-outpost-for-k3s-with-traefik/) seems to deploy his externally from just adding them in Authentik but just adding then dropped the new service in my cluster:

```bash
thaynes@kubem01:~$ k get services,pods,middleware -n authentik
NAME                                    TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
service/ak-outpost-auth-proxy-outpost   ClusterIP      10.43.249.155   <none>           9000/TCP,9300/TCP,9443/TCP   10m
service/authentik-postgresql            ClusterIP      10.43.157.234   <none>           5432/TCP                     2d11h
service/authentik-postgresql-hl         ClusterIP      None            <none>           5432/TCP                     2d11h
service/authentik-redis-headless        ClusterIP      None            <none>           6379/TCP                     2d11h
service/authentik-redis-master          ClusterIP      10.43.205.16    <none>           6379/TCP                     2d11h
service/authentik-server                LoadBalancer   10.43.206.143   192.168.40.102   80:31520/TCP,443:32696/TCP   2d11h

NAME                                                 READY   STATUS    RESTARTS      AGE
pod/ak-outpost-auth-proxy-outpost-7c944f6949-7mbzc   1/1     Running   0             10m
pod/authentik-postgresql-0                           1/1     Running   0             2d11h
pod/authentik-redis-master-0                         1/1     Running   0             2d11h
pod/authentik-server-84b66d969-d65rb                 1/1     Running   1 (33h ago)   2d11h
pod/authentik-server-84b66d969-lt2bt                 1/1     Running   1 (33h ago)   2d11h
pod/authentik-server-84b66d969-zkpks                 1/1     Running   2 (33h ago)   2d11h
pod/authentik-worker-9679ddbdd-czt9k                 1/1     Running   0             33h
pod/authentik-worker-9679ddbdd-gsj9z                 1/1     Running   0             2d11h
pod/authentik-worker-9679ddbdd-n9qbn                 1/1     Running   0             2d11h
pod/authentik-worker-9679ddbdd-qlf5t                 1/1     Running   0             2d11h
pod/authentik-worker-9679ddbdd-zzxpd                 1/1     Running   0             33h

NAME                                                  AGE
middleware.traefik.io/ak-outpost-auth-proxy-outpost   10m
```

The outpost even added it's own middleware! And it has the correct address, so we are cooking with gas now.

### How I Configured Mine

```yaml
apiVersion: traefik.io/v1alpha1 # traefik.containo.us/v1alpha1 depreciated in Traefik v3
kind: Middleware
metadata:
  name: authentik-auth-proxy
  namespace: traefik
spec:
  forwardAuth:
    # address of the identity provider (IdP)
    address: http://ak-outpost-auth-proxy-outpost.authentik:9000/outpost.goauthentik.io/auth/traefik
    trustForwardHeader: true
    # headers that are copied from the IdP response and set on forwarded request to the backend application
    authResponseHeaders:
    - X-authentik-username
    - X-authentik-groups
    - X-authentik-email
    - X-authentik-name
    - X-authentik-uid
    - X-authentik-jwt
    - X-authentik-meta-jwks
    - X-authentik-meta-outpost
    - X-authentik-meta-provider
    - X-authentik-meta-app
    - X-authentik-meta-version
```

### Kubectl Describe the Outposts:

```yaml
    Forward Auth:
    Address:  http://ak-outpost-auth-proxy-outpost.authentik:9000/outpost.goauthentik.io/auth/traefik
    Auth Response Headers:
      X-authentik-username
      X-authentik-groups
      X-authentik-email
      X-authentik-name
      X-authentik-uid
      X-authentik-jwt
      X-authentik-meta-jwks
      X-authentik-meta-outpost
      X-authentik-meta-provider
      X-authentik-meta-app
      X-authentik-meta-version
    Auth Response Headers Regex:  
    Trust Forward Header:         true
```
> **TODO** Migrate to the auto generated middleware incase it updates when authentik updates but test with what I have for now

IT'S ALIVE!!!! But now Kubernetes Dashboard wants some sort of a token:

![kubedash-wants-token]({{ site.url }}/images/web/kubedash-wants-token.png)

## Passing The Bearer Token

For my next challenge I need some Bearer token, but that needs ot be linked to a service account. I'm going to make an account to keep this seperate from other things:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubedash-admin
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubedash-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubedash-admin
  namespace: kubernetes-dashboard
```

Then I grab the token like this:

```bash
kubectl -n kubernetes-dashboard create token kubedash-admin
```

Then I can seal it up in a secret:

```bash
  kubectl create secret generic kubedash-admin-token \
  --namespace kubernetes-dashboard \
  --dry-run=client \
  --from-literal=token=HUGETOKEN -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem --format yaml \
  > sealedsecret-kubedash-admin-token.yaml
```

Then add it as extraEnv to SOMETHING:

```yaml
extraEnv:
  - name: TOKEN
    valueFrom:
      secretKeyRef:
        name:  kubedash-admin-token
        key: token

protocolHttp: true # TODO May not need?
```

I could also pass the token somehow in my `IngressRoute` but let's try all that first. 

However, that didn't work. So I just copy pasted the token to log in. I don't know if I'll use this dashboard much so I don't want to invest much more time here. I have a log in screen now and sometimes I may need to paste in a huge token.

I can now circle back and put better auth around my traefik dashboard!

### Even That Didn't Work

I guess that token only lasted 1hr so I'm gonna shoot for something a bit longer. First we need a secret:

`nano kubedash-loglived.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kubedash-admin
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: kubedash-admin
type: kubernetes.io/service-account-token
```

This time we just force it in there to get the token. We can add the crd to the cluster if we really want but it's a hack no matter what.

`kubectl create -f kubedash-loglived.yaml`

Extract!

```bash
kubectl get secret kubedash-admin -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
```


Re seal the long lived one:

```bash
  kubectl create secret generic kubedash-admin-token \
  --namespace kubernetes-dashboard \
  --dry-run=client \
  --from-literal=token=HUGETOKEN -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem --format yaml \
  > sealedsecret-kubedash-admin-token.yaml
```

NOPE

### Pass It As a Header

#### Either do this:

```bash
kubectl create secret generic kubedash-admin-token-header \
  --namespace traefik \
  --dry-run=client \
  --from-literal=authorization="Bearer HUGETOKEN" -o json \
  | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --cert pub-cert.pem --format yaml \
  > sealedsecret-kubedash-admin-token-header.yaml
```

#### Or roll the dice with this:

```bash
echo -n "Bearer HUGETOKEN" | base64
```

```
apiVersion: v1
kind: Secret
metadata:
  name: dashboard-token-secret
  namespace: kube-system
type: Opaque
data:
  authorization: CRAZYHUGETOKEN
```

#### Pass Secret Via ENV


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubedash-auth-token
  namespace: traefik
data:
  auth-token: |
    $(kubectl get secret kubedash-admin-token-header -o jsonpath='{.data.authorization}' | base64 --decode)
```

##### Load ENV in Traefik Helm

```yaml
env:
    - name: BEARER_TOKEN
      valueFrom:
        configMapKeyRef:
        name: kubedash-auth-token
        key: auth-token
```

##### Pass Via Middleware

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: forward-bearer-token-auth-header
  namespace: kube-system
spec:
  headers:
    customRequestHeaders:
      Authorization: "${BEARER_TOKEN}"
```

> **WARNING** Still have not got any of that to work so I just plugged the long lived one in and will likely not worry about it ever again.