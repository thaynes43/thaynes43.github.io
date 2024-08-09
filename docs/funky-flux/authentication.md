---
title: Authentication
permalink: /docs/funky-flux/authentication/
---

On my journey to setup the k8s dashboard with proper auth I've so far configured an ![external dns]({{ site.url }}/docs/funky-flux/external-dns/) and ![ssl certs]({{ site.url }}/docs/funky-flux/certs/). To log in nicely I need a few more things.

## MetalLB Load Balancer

The [load balancer guide](https://geek-cookbook.funkypenguin.co.nz/kubernetes/loadbalancer/metallb/) is be using [MetalLB](https://metallb.io/installation/) so I am going to roll with it.

I followed everything pretty closely except I added a single `metallb-service` folder with `configs` and `controllers` sub folders so things were a bit neater. I also added both kustomizations to one file like with the monitoring stuff.

Everything came up fine:

```
NAME                                  READY   STATUS    RESTARTS   AGE   IP              NODE       NOMINATED NODE   READINESS GATES
metallb-controller-77cb7f5d88-snbq5   1/1     Running   0          73s   10.244.2.51     debian03   <none>           <none>
metallb-speaker-2zvn4                 4/4     Running   0          73s   192.168.0.208   debian03   <none>           <none>
metallb-speaker-57nnb                 4/4     Running   0          73s   192.168.0.190   kubevip    <none>           <none>
metallb-speaker-gjv7k                 4/4     Running   0          73s   192.168.0.27    debian02   <none>           <none>
metallb-speaker-kgs8d                 4/4     Running   0          73s   192.168.0.6     debian01   <none>           <none>
```

Even a pod on my control-plane which I don't usually see.

Next I was instructed to change podinfo from `ClusterIP` to `LoadBalancer` so Metallb would assign it an ip.

Well, it did:

```
kubectl get services -n podinfo
podinfo                podinfo                                          LoadBalancer   10.100.130.118   192.168.0.1   9898:30839/TCP,9999:31962/TCP   6d1h
```

This fucked shit up bigtime. `192.168.0.1` is my router and I immediately lost my network connection. Tethering to my phone fix it but I'll need IPs that are not in use. However, it doesn't seem like I can use a different subnet than my cluster but I can at least try...

I made a vlan and assigned Metallib `- 192.168.4.6-192.168.4.254`. I don't understand how you'd avoid IP conflicts the other way unless I threw the entire cluster on the vLAN which I could do with some extra effort.

```
thaynes@kubevip:~/workspace/rook-setup$ kubectl get services -n podinfo
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
podinfo   LoadBalancer   10.100.130.118   192.168.4.6   9898:32176/TCP,9999:30146/TCP   6d1h
```

Well, it got the ip! And I can hit it from `http://kubevip:32176/` but the IP assigned doesn't seem to do anything, my router doesn't see it.

Just to see what it looks like on the cluster's IP I set the range to things not in use, `192.168.0.35-192.168.0.40`.

```
thaynes@kubevip:~/workspace/rook-setup$ kubectl get services -n podinfo
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                         AGE
podinfo   LoadBalancer   10.100.130.118   192.168.0.36   9898:32176/TCP,9999:30146/TCP   6d1h
```

The behavior is exactly the same so maybe all good and I just need ingress next.

## Ingress