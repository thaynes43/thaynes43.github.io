---
title: Authentication
permalink: /docs/funky-flux/authentication/
---

## Setting Up Authentik

Welcome back to your regularly scheduled programming. I think we last got Traefik setup and were moving on to authentication before we blew everything up by changing the servers to a VLAN. Well, we THINK most of that is behind us so we are moving on to [this guide](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/authentik/) for [authentik](https://goauthentik.io/).

I mostly followed the guide but plugged in `2024.6.x` for the chart since these guides are pretty dated. I also diverted a bit based on what I learned elsewhere.

### Values

#### Global

```yaml
   global:
   ...
      env:
        - name: AUTHENTIK_BOOTSTRAP_PASSWORD
          value: REDACTED
```

#### Authentik 

```yaml
    authentik:
      # -- Log level for server and worker
      log_level: info
      # -- Secret key used for cookie singing and unique user IDs,
      # don't change this after the first install
      secret_key: REDACTED
      ...
      postgresql:
        password: REDACTED
```

#### Server

```yaml
    server:
    ...
      replicas: 3
    ...
      service:
        type: LoadBalancer
    ...
      metrics:
        # -- deploy metrics service
        enabled: true #TODO OFF
        service:
          serviceMonitor:
          # -- enable a prometheus ServiceMonitor
          enabled: true
```

#### Worker

```yaml
    ## authentik worker
    worker:
      # -- The number of worker pods to run
      replicas: 5
```

#### Prometheus

```yaml
    prometheus:
      rules:
        enabled: true # TODO OFF
        # -- PrometheusRule namespace
        namespace: monitoring
```

#### Postgresql

```yaml
    postgresql:
      # -- enable the Bitnami PostgreSQL chart. Refer to https://github.com/bitnami/charts/blob/main/bitnami/postgresql/ for possible values.
      enabled: true
      auth:
        username: authentik
        database: authentik
        password: SECRETPASS
      primary:
        extendedConfiguration: |
          max_connections = 500
        persistence:
          enabled: true
          accessModes:
            - ReadWriteOnce
          storageClass: ceph-rbd 
```

#### Redis

```yaml
    redis:
      # -- enable the Bitnami Redis chart. Refer to https://github.com/bitnami/charts/blob/main/bitnami/redis/ for possible values.
      enabled: true
      architecture: standalone
      auth:
        enabled: false
      master:
        persistence:
          enabled: true
          accessModes:
            - ReadWriteOnce
          storageClass: ceph-rbd 
```

#### DNSEntry 

```yaml
apiVersion: externaldns.k8s.io/v1alpha1
kind: DNSEndpoint
metadata:
  name: "authentik.example.com"
  namespace: external-dns
spec:
  endpoints:
  - dnsName: "authentik.example.com"
    recordTTL: 180
    recordType: CNAME
    targets:
    - "example.com"
```

#### Ingress

```yaml
      ingress:
        enabled: true
        ingressClassName: traefik
        hosts:
          - authentik.example.com
        tls:
          - secretName: certificate-local.example.com
            hosts:
              - authentik.example.com  
```

> **WARNING** I did not end up using this but I don't believe it was related to the issues I was experiencing.

> **TODO** Try this again later as it may be more flexible.

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: authentik
  namespace: traefik
  annotations: 
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`authentik.example.com`)
      kind: Rule
      middlewares:
        - name: default-headers
      services:  
        - name: authentik
          port: 80
          namespace: authentik
  tls:
    secretName: certificate.example.com
```

### Troubleshooting

First problem was with how the environment variables in the guide were formatted. These are optional so if they don't work again I can roll without them.

```
  Warning  InstallFailed     6m (x10 over 12m)  helm-controller  Helm install failed for release authentik/authentik with chart authentik@2024.6.3: template: authentik/templates/worker/deployment.yaml:77:22: executing "authentik/templates/worker/deployment.yaml" at <concat .Values.global.env .Values.worker.env>: error calling concat: Cannot concat type map as list
```

Second problem seemed easy too. Authentik is now up but my Traefik has no route in it. Maybe because:

```
2024-08-14 22:24:28	2024-08-15T02:24:28Z ERR github.com/traefik/traefik/v3/pkg/provider/kubernetes/crd/kubernetes_http.go:102 > error="kubernetes service not found: traefik/authentik" ingress=authentik namespace=traefik providerName=kubernetescrd
```

And I did forget to specify the services namespaces... And I guess you MUST specify a port.

Now we are ready to rock and roll with the default `akadmin` user and the `flux` password we defined for the first log-in.

## OIDC Authentication with Authentik

Before we can use this for the dashboard we need to follow [this guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/oidc-authentication/authentik/) to enable OIDC Authentication. 

This one feels like something is missing, maybe an entire step. A page is missing and the comments are not promising. 

People also say I will need `--exec-arg=--oidc-extra-scope=profile` but it doesn't seem relevant to the step I am on. Tread lightly for the next tutorial I guess.

## Locking Down the Cluster With... The Cluster

[The next guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/oidc-authentication/k3s-authentik/) gets into the good stuff. And it looks updated for that `--exec-args` mentioned in the comments of the previous "recipe" so I think we are good to go.

### Enable OIDC

First I need to update some k3s config and reboot k3s to enable this fancy authentication layer.

`nano /etc/rancher/k3s/config.yaml`

```yaml
kube-apiserver-arg:
  - "oidc-issuer-url=https://authentik.example.com/application/o/kube-apiserver/"
  - "oidc-client-id=kube-apiserver"
  - "oidc-username-claim=email"
  - "oidc-groups-claim=groups"
  - "oidc-groups-prefix=oidc:"
```

First thing it says it to run `journalctl -u k3s -f` to watch if it worked and I see a shit ton of errors coming out of the metrics endpoints I configured. We can deal with that later:

```
Aug 15 00:02:07 kubem01 k3s[352471]: E0815 00:02:07.569544  352471 horizontal.go:270] failed to compute desired number of replicas based on listed metrics for Deployment/authentik/authentik-worker: invalid metrics (1 invalid out of 1), first error is: failed to get cpu resource metric value: failed to get cpu utilization: missing request for cpu in container worker of Pod authentik-worker-9679ddbdd-gsj9z
```

Probably just disable auto scaling.

Next to restart the damn thing with `systemctl restart k3s`. This is terrifying.

But I think it worked. Next get a token which is where I think the comment from the last post comes into play. The last post said `--exec-arg=--oidc-extra-scope=profile`  was missing and in this block it was (I added it):

```
kubectl oidc-login setup \
  --oidc-issuer-url=https://authentik.example.com/application/o/kube-apiserver/ \
  --oidc-client-id=kube-apiserver \
  --oidc-client-secret=REDACTED \
  --oidc-extra-scope=profile,email
  ```

But in the example funky man does throw in that parameter:

```
~ ❯ kubectl oidc-login setup --oidc-issuer-url=https://authentik.example.com/application/o/kube-apiserver/ --oidc-client-id=kube-apiserver --oidc-client-secret=<your secret> --oidc-extra-scope=profile,email
```

So I think we go with that!


But first 

### Krew Plugins

I need to install [krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) to then install a plugin called [kubelogin](https://github.com/int128/kubelogin).

```
sudo journalctl -u k3s -f
watch -n1 flux get kustomizations
```

It's pretty cool. `kubectl krew search` shows tons of plugins but IDK what they do. The one I need is right in my reach:

```
thaynes@kubem01:~$ kubectl krew search | grep oidc-login
oidc-login                      Log in to the OpenID Connect provider               no
```

It worked but was a bit angry:

```bash
thaynes@kubem01:~$ kubectl krew install oidc-login
Updated the local copy of plugin index.
Installing plugin: oidc-login
Installed plugin: oidc-login
\
 | Use this plugin:
 | 	kubectl oidc-login
 | Documentation:
 | 	https://github.com/int128/kubelogin
 | Caveats:
 | \
 |  | You need to setup the OIDC provider, Kubernetes API server, role binding and kubeconfig.
 | /
/
WARNING: You installed plugin "oidc-login" from the krew-index plugin repository.
   These plugins are not audited for security by the Krew maintainers.
   Run them at your own risk.
thaynes@kubem01:~$ 
```

#### New One - browse-pvc

See [browse-pvc](https://github.com/clbx/kubectl-browse-pvc).

```bash
kubectl krew install browse-pvc
```

Now back to the OCD stuff.

### oidc-login Setup

The command was super cool. It gave me a document even which I can format in a markdown codeblock! 

```bash
thaynes@kubem01:/etc/rancher/k3s$ kubectl oidc-login setup \
  --oidc-issuer-url=https://authentik.example.com/application/o/kube-apiserver/ \
  --oidc-client-id=kube-apiserver \
  --oidc-client-secret=GETFROMAUTHENTIK \
  --oidc-extra-scope=profile,email
authentication in progress...

## 2. Verify authentication

You got a token with the following claims:

{
  "iss": "https://authentik.example.com/application/o/kube-apiserver/",
  "sub": "huuugeblob",
  "aud": "kube-apiserver",
  "exp": 1723781486,
  "iat": 1723781186,
  "auth_time": 1723781186,
  "acr": "goauthentik.io/providers/oauth2/default",
  "nonce": "REDACTED",
  "email": "myemail@example.com",
  "email_verified": true,
  "name": "My Name",
  "given_name": "My Name",
  "preferred_username": "thaynes",
  "nickname": "thaynes",
  "groups": [
    "authentik Admins",
    "admin-kube-apiserver"
  ]
}

## 3. Bind a cluster role

Run the following command:

	kubectl create clusterrolebinding oidc-cluster-admin --clusterrole=cluster-admin --user='https://authentik.example.com/application/o/kube-apiserver/#huuugeblob'

## 4. Set up the Kubernetes API server

Add the following options to the kube-apiserver:

	--oidc-issuer-url=https://authentik.example.com/application/o/kube-apiserver/
	--oidc-client-id=kube-apiserver

## 5. Set up the kubeconfig

Run the following command:

	kubectl config set-credentials oidc \
	  --exec-api-version=client.authentication.k8s.io/v1beta1 \
	  --exec-command=kubectl \
	  --exec-arg=oidc-login \
	  --exec-arg=get-token \
	  --exec-arg=--oidc-issuer-url=https://authentik.example.com/application/o/kube-apiserver/ \
	  --exec-arg=--oidc-client-id=kube-apiserver \
	  --exec-arg=--oidc-client-secret=GETFROMAUTHENTIK \
	  --exec-arg=--oidc-extra-scope=profile \
	  --exec-arg=--oidc-extra-scope=email

## 6. Verify cluster access

Make sure you can access the Kubernetes cluster.

	kubectl --user=oidc get nodes

You can switch the default context to oidc.

	kubectl config set-context --current --user=oidc

You can share the kubeconfig to your team members for on-boarding.
bash: authentication: command not found
```

### Apply to K3S

To test this all out I need a second machine to try and access the cluster.

#### kubectl on Windows 

Now we have to copy the K3S config and try to access it from my desktop. But I don't have any shit installed so IDK what this guy is talking about. Guess I'll do [this](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/).

```bash
manof@HaynesHyperion MINGW64 ~
$ kubectl version --client
Client Version: v1.31.0
Kustomize Version: v5.4.2
```

Then I made a file named `config` with no extension in the directory `cd ~` brought me which was `/c/Users/manof`. 

Finally, I copy pasted the contents of `/etc/rancher/k3s/k3s.yaml` to this file and boom:

```bash
manof@HaynesHyperion MINGW64 ~
$ kubectl get nodes
NAME      STATUS   ROLES                       AGE     VERSION
kubem01   Ready    control-plane,etcd,master   2d10h   v1.30.3+k3s1
kubem02   Ready    control-plane,etcd,master   2d10h   v1.30.3+k3s1
kubem03   Ready    control-plane,etcd,master   2d10h   v1.30.3+k3s1
kubew01   Ready    <none>                      2d10h   v1.30.3+k3s1
kubew02   Ready    <none>                      2d10h   v1.30.3+k3s1
kubew03   Ready    <none>                      2d10h   v1.30.3+k3s1
kubew04   Ready    <none>                      2d10h   v1.30.3+k3s1
kubew05   Ready    <none>                      2d10h   v1.30.3+k3s1
```

I am DONE with nomachine!!! Well, maybe not since I don't have any other CLI tools or the alias set up. Or autocomplete etc. Can always just SSH too.

#### Applying the OIDC Rule

Now it's time to lock this shit down.

```bash
thaynes@kubem01:/etc/rancher/k3s$       sudo kubectl config set-credentials oidc \
          --exec-api-version=client.authentication.k8s.io/v1beta1 \
          --exec-command=kubectl \
          --exec-arg=oidc-login \
          --exec-arg=get-token \
          --exec-arg=--oidc-issuer-url=https://authentik.example.com/application/o/kube-apiserver/ \
          --exec-arg=--oidc-client-id=kube-apiserver \
          --exec-arg=--oidc-client-secret=REDACTED \
          --exec-arg=--oidc-extra-scope=profile \
          --exec-arg=--oidc-extra-scope=email
User "oidc" set.
 ```

 And now we can see that whatever the fuck that did locked me out (unless I have the master config file):

 ```bash
thaynes@kubem01:~/.kube$ kubectl --user=oidc cluster-info

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
Error from server (Forbidden): services is forbidden: User "admin@example.com" cannot list resource "services" in API group "" in the namespace "kube-system"
```

To get around this tretchery I need a `ClusterRoleBinding`. If this works I can permimently assume this poor sap's user ID with:

### Testing ODIC
> **NOTE** It took me longer to get past here than the rest of the shit in this document. Turned out I needed `  - "oidc-groups-prefix=oidc:"` in that `config.yaml` file in the k3s directory.

I shound now get in with `kubectl --user=oidc <do something>`.

```bash
thaynes@kubem01:/etc/rancher/k3s$ kubectl --user=oidc get nodes
NAME      STATUS   ROLES                       AGE     VERSION
kubem01   Ready    control-plane,etcd,master   3d10h   v1.30.3+k3s1
kubem02   Ready    control-plane,etcd,master   3d9h    v1.30.3+k3s1
kubem03   Ready    control-plane,etcd,master   3d9h    v1.30.3+k3s1
kubew01   Ready    <none>                      3d9h    v1.30.3+k3s1
kubew02   Ready    <none>                      3d9h    v1.30.3+k3s1
kubew03   Ready    <none>                      3d9h    v1.30.3+k3s1
kubew04   Ready    <none>                      3d9h    v1.30.3+k3s1
kubew05   Ready    <none>                      3d9h    v1.30.3+k3s1
```

IT FINALLY WORKS! Now I can set this mode of auth to happen all the time:

```bash
thaynes@kubem01:/etc/rancher/k3s$ kubectl config set-context --current --user=oidc
Context "default" modified.
```

But I think I can go back with:

```bash
kubectl config set-context --current --user=default
```

Since that is what I got after:

```bash
krew install whoami
thaynes@kubem01:~/.kube$ kubectl whoami
system:admin
```

I knew Krew would be helpful

And here's it live!

![kube auth]({{ site.url }}/gifs/k8s/kube-auth.png)

#### Other Resources

When the `kubectl --user=oidc <do something>` test didn't work I branched out and these docs eventually lead me to adding `  - "oidc-groups-prefix=oidc:`.

https://medium.com/@extio/kubernetes-authentication-with-oidc-simplifying-identity-management-c56ede8f2dec

https://kamrul.dev/kubernetes-oidc-authentication-gke/

https://kamrul.dev/kubernetes-oidc-authentication-gke/

> **NOTE** the dude never made a guide for this but [weave](https://github.com/weaveworks/weave-gitops) can use these creds

#### HA Cluster

I fixed a my kube config so the load balancer was used, `server: https://kube.example:6443`, but I started getting random access denied messages that cleared up by trying the same command twice. I think this is probably because I only ran though this setup on one master node. Will see if installing this stuff on the others and syncing the configs helps.

Sure enough this fixed it

## Google Workspace Sync

First thing that caught my eye was the ability to sync with Google Workspace. [This guide](https://docs.goauthentik.io/docs/providers/gws/) looks to go over the details. 

## Plex Social Log-in

Coming Soon...