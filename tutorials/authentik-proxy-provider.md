---
title: Authentik Proxy Provider
permalink: /tutorials/authentik-proxy-provider/
---

This tutorial is designed to show step by step how to add a new Proxy Provider in Authentik and then create an `IngressRoute` for Traefik to use with `Middleware` so it goes through Authentik for authentication.

We will use Zigbee2Mqtt as the application to proxy.

## Dependencies

* You must already have a working instance of Authentik
* You must already have an Outpost deployed from Authentik that is not the embedded outpost (or know how to get it to work with that cause I don't)
* You must already have Traefik set up
* You have an application thats needs to have a proxy

## Authentik Configuration

### Create Proxy Provider

In Authentik use the side bar to navigate to `Applications` -> `Providers` and then click `Create` on the main frame.

Now select the `Proxy Provider` type and click `Create`:

![select-proxy-provider]({{ site.url }}/images/tutorials/authentik-proxy-provider/select-proxy-provider.png)

The next page needs some basic information, In this case we are just forwarding to a page that previously had no access control.

> **NOTE** I am not exposing this "External host" to the internet so I added a `.local` prefix before my domain. If I were to expose this I'd simply remove that and add the DNS entry in Cloudflare instead of my router.

![create-proxy-provider]({{ site.url }}/images/tutorials/authentik-proxy-provider/create-proxy-provider.png)

### Create Application

Next we create the application. To do so navigate the sidebar to `Applications` -> `Applications` and click `Create` on the main frame. 

Give this a name you don't mind seeing because it'll be in the non-admin screen of Authentik. You can also expand "UI settings" and select a logo. So far I have been linking to ones I've googled but I've also downloaded them so I can host my own logos later and not risk them going away.

![create-application]({{ site.url }}/images/tutorials/authentik-proxy-provider/create-application.png)

### Updating the Outpost

All you have to do here is double click on the application we just created to add it to the outpost.

> **NOTE** You can probably do this with the embedded outpost but this was much easier as it added `Middleware` to kubernetes which had everything I needed.

![update-outpost]({{ site.url }}/images/tutorials/authentik-proxy-provider/update-outpost.png)

## Routing Configuration

### Adding DNS Entry

Next you need to map `https://z2m.local.example.com` to the external IP of your load balancer. In my case that is traefik:

| NAME | TYPE | CLUSTER-IP | EXTERNAL-IP | PORT(S) | AGE |
| traefik | LoadBalancer | 10.43.34.58 | 192.168.40.100 | 80:32514/TCP,443:31648/TCP,443:31648/UDP | 7d4h |

![dns-entry]({{ site.url }}/images/tutorials/authentik-proxy-provider/dns-entry.png)

This would be very similar for other DNS providers. I don't want to expose Zigbee2MQTT to the internet.

### Adding IngressRoute

The final piece to tie it all together is the `IngresRoute` that Traefik is going to use to request authorization from Authentik before sending traffic along to the site. 

The outpost should have created Middleware which you can find like this:

```
thaynes@kubem01:~/workspace$ k get middleware -n authentik 
NAME                            AGE
ak-outpost-auth-proxy-outpost   2d11h
```

For me this wasn't in the namespace Traefik was using but you can specify. However, I am using my own that is based off of this file so you will see that the name doesn't match. 

`ingressroute-zigbee2mqtt.yaml`
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: zigbee2mqtt
  namespace: traefik
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`www.z2m.local.example.com`)
      kind: Rule
      services:
        - kind: Service
          name: zigbee2mqtt
          namespace: iot-services
          port: 8080
      middlewares:      
        - name: authentik-auth-proxy
          namespace: traefik 
    - kind: Rule
      match: Host(`z2m.local.example.com`) && PathPrefix(`/`)
      services:
        - kind: Service
          name: zigbee2mqtt
          namespace: iot-services
          port: 8080
      middlewares:     
        - name: authentik-auth-proxy
          namespace: traefik 
  tls:
    secretName: certificate-local.example.com
```

## Outcome

