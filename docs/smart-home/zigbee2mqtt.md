---
title: Zigbee2MQTT
permalink: /docs/smart-home/zigbee2mqtt/
---

## Determining the Zigbee Channel

Zigbee2MQTT is a great way to connect Zigbee deices from any brand to one coordinator. I'm currently using two 2.4 GHz channels for my current home so I will need to squeeze this in:

| Device | Channel |
| rpi4 | 11 |
| huge | 25 |
| **tubeszb-zigbee01** | 15 |
| **tubeszb-zigbee02** | 20 |

Also important to make sure my WiFi channels don't overlap:

![24ghz-channels]({{ site.url }}/images/unifi/24ghz-channels.png)

Channel 11 overlaps with those, so it's good we are migrating away from it. I'm going to go with channel 20. 

## Flux Files

We will use the existing `iot-services` namespace for this service.

There appears to be a [semi-official helm chart](https://github.com/Koenkk/zigbee2mqtt-chart) which was added after a recent feature request. It looks to be hosted on truecharts with instructions for install [here](https://staging.artifacthub.io/packages/helm/truecharts/zigbee2mqtt).

### HelmRepository

`helmrepository-zigbee2mqtt-chart.yaml`
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: zigbee2mqtt-chart
  namespace: flux-system
spec:
  interval: 15m
  url: https://github.com/Koenkk/zigbee2mqtt-chart
```

Where you would run:

```bash
helm install my-zigbee2mqtt zigbee2mqtt-chart/zigbee2mqtt --version 1.37.1
```

I can't tell if this repo hosts the chart though. It seems to be just the code base for it. There are also some concerning PRs and issues to watch out for here.

### Bootstrap Kustomization

This will live with the `mosquitto` Kustomization:

`nano kustomization-iot-services.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: zigbee2mqtt
  namespace: flux-system
spec:
  dependsOn:
    - name: mosquitto
  interval: 15m
  retryInterval: 1m  
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  healthChecks:
    - apiVersion: apps/v1
      kind: HelmRelease
      name: zigbee2mqtt
      namespace: iot-services
  path: ./iot-services/zigbee2mqtt
```

### HelmRelease

`helmrelease-zigbee2mqtt.yaml`

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: zigbee2mqtt
  namespace: iot-services
spec:
  chart:
    spec:
      chart: zigbee2mqtt
      version: 1.37.x # https://github.com/Koenkk/zigbee2mqtt-chart/releases
      sourceRef:
        kind: HelmRepository
        name: zigbee2mqtt-chart
        namespace: flux-system
  interval: 15m
  timeout: 5m
  releaseName: zigbee2mqtt
  values: # https://github.com/Koenkk/zigbee2mqtt-chart/blob/main/charts/zigbee2mqtt/values.yaml
    image:
      tag: "1.39.1" # https://github.com/Koenkk/zigbee2mqtt/releases/
    service:
      annotations:
        metallb.universe.tf/loadBalancerIPs: 192.168.40.105
      type: LoadBalancer
      port: 8080
    statefulset:
      storage:
        enabled: true
        storageClassName: ceph-rbd
    zigbee2mqtt:
      mqtt:
        server: "mqtt://mosquitto.haynesnetwork:1883"
      serial:
        port: "tcp://tubeszb-zigbee01.haynesnetwork:6638" # https://github.com/tube0013/tube_gateways?tab=readme-ov-file#network-coordinators-1
        adapter: ember
      advanced:
        # Changing requires re-pairing of all devices! (Note: use a ZLL channel: 11, 15, 20, or 25 to avoid Problems)
        # rpi4 = 11; Hue = 25; tubeszb-zigbee02 = 20
        channel: 15
        # Can go up to 19.9 dBm for MGM240PB32VNN per https://tubeszb.com/product/efr32-mgm24-poe-coordinator/
        transmit_power: 5   
```

### All Values

For reference here are the values of the chart at version 1.37.1:

```yaml
# -- override the release name
nameOverride: null
# -- override the name of the objects generated
fullnameOverride: null
customLabels: {}
image:
  # -- Image repository for the `zigbee2mqtt` container.
  repository: koenkk/zigbee2mqtt
  # -- Version for the `zigbee2mqtt` container.
  tag: "1.37.1"
  # -- Container pull policy
  pullPolicy: IfNotPresent
  # -- Container additional secrets to pull image
  imagePullSecrets: {}
service:
  # -- annotations for the service created
  annotations: {}
  # -- type of Service to be created
  type: LoadBalancer
  # -- port in which the service will be listening
  port: 8080
statefulset:
  storage:
    enabled: false
    size: 1Gi
    # -- the name for the storage class to be used in the persistent volume claim
    storageClassName: freenas-nfs-csi
    existingVolume: ""
    ## Persistent Volume selectors
    ## https://kubernetes.io/docs/concepts/storage/persistent-volumes/#selector
    matchLabels: {}
    matchExpressions: {}
  # -- pod dns policy
  dnsPolicy: ClusterFirst
  # -- CPU/Memory configuration for the pods
  resources:
    requests:
      memory: 600Mi
      cpu: 200m
    limits:
      memory: 600Mi
      cpu: 200m
  # -- Node taint tolerations for the pods
  tolerations: {}
  # -- Select specific kube node, this will allow enforcing zigbee2mqtt running
  # only on the node with the USB adapter connected
  nodeSelector: {}
zigbee2mqtt:
  homeassistant:
    enabled: true
    discovery_topic: 'homeassistant'
    status_topic: 'hass/status'
    legacy_entity_attributes: true
    legacy_triggers: false
  # -- Optional: allow new devices to join.
  permit_join: false
  availability:
    active:
      # -- Time after which an active device will be marked as offline in
      # minutes (default = 10 minutes)
      timeout: 10
    passive:
      # -- Time after which a passive device will be marked as offline in
      # minutes (default = 1500 minutes aka 25 hours)
      timeout: 1500
  timezone: UTC
  external_converters: []
  mqtt:
    # -- Required: MQTT server URL (use mqtts:// for SSL/TLS connection)
    server: "mqtt://localhost:1883"
    # -- Optional: MQTT base topic for Zigbee2MQTT MQTT messages (default: zigbee2mqtt)
    # base_topic: zigbee2mqtt
    # -- Optional: absolute path to SSL/TLS certificate of CA used to sign server and client certificates (default: nothing)
    # ca: '/etc/ssl/mqtt-ca.crt'
    # -- Optional: absolute paths to SSL/TLS key and certificate for client-authentication (default: nothing)
    # key: '/etc/ssl/mqtt-client.key'
    # cert: '/etc/ssl/mqtt-client.crt'
    # -- Optional: MQTT server authentication user (default: nothing)
    # user: my_user
    # -- Optional: MQTT server authentication password (default: nothing)
    # password: my_password
    # -- Optional: MQTT client ID (default: nothing)
    # client_id: 'MY_CLIENT_ID'
    # -- Optional: disable self-signed SSL certificates (default: true)
    # reject_unauthorized: true
    # -- Optional: Include device information to mqtt messages (default: false)
    # include_device_information: true
    # -- Optional: MQTT keepalive in seconds (default: 60)
    # keepalive: 60
    # -- Optional: MQTT protocol version (default: 4), set this to 5 if you
    # use the 'retention' device specific configuration
    # version: 4
    # -- Optional: Disable retain for all send messages. ONLY enable if you MQTT broker doesn't
    # support retained message (e.g. AWS IoT core, Azure IoT Hub, Google Cloud IoT core, IBM Watson IoT Platform).
    # Enabling will break the Home Assistant integration. (default: false)
    # force_disable_retain: false
  serial:
    # -- Required: location of the adapter (e.g. CC2531).
    # USB adapters - use format "port: /dev/ttyACM0"
    # To autodetect the USB port, set 'port: null'.
    # Ethernet adapters - use format "port: tcp://192.168.1.12:6638"
    port: "/dev/ttyACM0"
    # -- Optional: disable LED of the adapter if supported (default: false)
    disable_led: false
    # -- Optional: adapter type, not needed unless you are experiencing problems (default: shown below, options: zstack, deconz, ezsp)
    # adapter: null
    # -- Optional: Baud rate speed for serial port, this can be anything firmware support but default is 115200 for Z-Stack and EZSP, 38400 for Deconz, however note that some EZSP firmware need 57600.
    baudrate: 115200
    # -- Optional: RTS / CTS Hardware Flow Control for serial port (default: false)
    rtscts: false
  # -- Optional: OTA update settings
  # See https://www.zigbee2mqtt.io/guide/usage/ota_updates.html for more info
  ota:
    # -- Optional: use IKEA TRADFRI OTA test server, see OTA updates documentation (default: false)
    ikea_ota_use_test_url: false
    # -- Minimum time between OTA update checks
    update_check_interval: 1440
    # -- Disable automatic update checks
    disable_automatic_update_check: false
  frontend:
    # -- Mandatory, default 8080
    port: 8080
    # -- Optional, empty by default to listen on both IPv4 and IPv6. Opens a unix socket when given a path instead of an address (e.g. '/run/zigbee2mqtt/zigbee2mqtt.sock')
    # Don't set this if you use Docker or the Home Assistant add-on unless you're sure the chosen IP is available inside the container
    host: 0.0.0.0
    # -- Optional, enables authentication, disabled by default, cleartext (no hashing required)
    auth_token: null
    # -- Optional, url on which the frontend can be reached, currently only used for the Home Assistant device configuration page
    url: null
  advanced:
    # -- Optional: ZigBee pan ID (default: shown below)
    # Setting pan_id: GENERATE will make Zigbee2MQTT generate a new panID on next startup
    # pan_id: null
    #  Optional: Zigbee extended pan ID, GENERATE will make Zigbee2MQTT generate a new extended panID on next startup (default: shown below)
    # ext_pan_id: null
    # --  Optional: ZigBee channel, changing requires re-pairing of all devices. (Note: use a ZLL channel: 11, 15, 20, or 25 to avoid Problems)
    # (default: 11)
    channel: 11
    # --  Optional: network encryption key
    # GENERATE will make Zigbee2MQTT generate a new network key on next startup
    # Note: changing requires repairing of all devices (default: shown below)
    # network_key: null
    log_output:
      - console
    log_level: info
    timestamp_format: 'YYYY-MM-DD HH:mm:ss'
    # -- Optional: state caching, MQTT message payload will contain all attributes, not only changed ones.
    # -- Has to be true when integrating via Home Assistant (default: true)
    cache_state: true
    # -- Optional: persist cached state, only used when cache_state: true (default: true)
    cache_state_persistent: true
    # -- Optional: send cached state on startup, only used when cache_state_persistent: true (default: true)
    cache_state_send_on_startup: true
    # -- Optional: Add a last_seen attribute to MQTT messages, contains date/time of last Zigbee message
    # possible values are: disable (default), ISO_8601, ISO_8601_local, epoch (default: disable)
    last_seen: 'disable'
    # -- Optional: Add an elapsed attribute to MQTT messages, contains milliseconds since the previous msg (default: false)
    elapsed: false
    # -- Optional: Enables report feature, this feature is DEPRECATED since reporting is now setup by default
    # when binding devices. Docs can still be found here: https://github.com/Koenkk/zigbee2mqtt.io/blob/master/docs/information/report.md
    report: true
    # -- Optional: disables the legacy api (default: shown below)
    legacy_api: false
    # -- Optional: MQTT output type: json, attribute or attribute_and_json (default: shown below)
    # Examples when 'state' of a device is published
    # json: topic: 'zigbee2mqtt/my_bulb' payload '{"state": "ON"}'
    # attribute: topic 'zigbee2mqtt/my_bulb/state' payload 'ON"
    # attribute_and_json:
    # -- Optional: configure adapter concurrency (e.g. 2 for CC2531 or 16 for CC26X2R1) (default: null, uses recommended value)
    adapter_concurrent: null
    # -- Optional: Transmit power setting in dBm (default: 5).
    # This will set the transmit power for devices that bring an inbuilt amplifier.
    # It can't go over the maximum of the respective hardware and might be limited
    # by firmware (for example to migrate heat, or by using an unsupported firmware).
    # For the CC2652R(B) this is 5 dBm, CC2652P/CC1352P-2 20 dBm.
    transmit_power: 5
    # -- Optional: Set the adapter delay, only used for Conbee/Raspbee adapters (default 0).
    # In case you are having issues try `200`.
    # For more information see https://github.com/Koenkk/zigbee2mqtt/issues/4884
    adapter_delay: 0
# -- Ingress configuration. Zigbee2mqtt does use webssockets, which is not part of the Ingress standart settings.
# most of the popular ingresses supports them through annotations. Please check https://www.zigbee2mqtt.io/guide/installation/08_kubernetes.html
# for examples.
ingress:
  # -- When enabled a new Ingress will be created
  enabled: false
  # -- The ingress class name for the ingress
  ingressClassName: contour
  # -- Additional labels for the ingres
  labels: {}
  # -- Ingress implementation specific (potentially) for most use cases Prefix should be ok
  pathType: Prefix
  # Additional annotations for the ingress. ExternalDNS, and CertManager are tipically integrated here
  annotations: { }
  # -- list of hosts that should be allowed for the zigbee2mqtt service
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
        - path: /api
          pathType: ImplementationSpecific
  # -- configuration for tls service (ig any)
  tls:
    - secretName: some-tls-secret
      hosts:
        - yourdomain.com
```

## Testing

Testing went as expected. These references got me through:

- https://www.zigbee2mqtt.io/guide/configuration/adapter-settings.html
- https://www.zigbee2mqtt.io/guide/adapters/emberznet.html
- https://tubeszb.com/product/efr32-mgm24-poe-coordinator/ 

Ultimately I was just missing a few settings so it could communicate with my adaptor.

This was the error I was struggling with for a few iterations:

```
[2024-08-19 15:43:30] error: 	z2m: Error: Failed to connect to the adapter (Error: SRSP - SYS - ping after 6000ms)
    at ZStackAdapter.start (/app/node_modules/zigbee-herdsman/src/adapter/z-stack/adapter/zStackAdapter.ts:119:27)
    at Controller.start (/app/node_modules/zigbee-herdsman/src/controller/controller.ts:127:29)
    at Zigbee.start (/app/lib/zigbee.ts:63:27)
    at Controller.start (/app/lib/controller.ts:139:27)
    at start (/app/index.js:154:5)
```

Once things worked it crashed once for this:

```
Error: EROFS: read-only file system, open '/app/data/configuration.yaml'
    at Object.openSync (node:fs:596:3)
    at Object.writeFileSync (node:fs:2322:35)
    at Object.writeIfChanged (/app/lib/util/yaml.ts:25:12)
    at write (/app/lib/util/settings.ts:272:10)
    at Object.set (/app/lib/util/settings.ts:497:5)
    at Controller.start (/app/lib/controller.ts:151:22)
    at processTicksAndRejections (node:internal/process/task_queues:95:5)
    at runNextTicks (node:internal/process/task_queues:64:3)
    at processImmediate (node:internal/timers:447:9)
    at start (/app/index.js:154:5)
```

But the next restart it went fine!

![fresh-z2m]({{ site.url }}/images/web/fresh-z2m.png)

## Blocked For Now

The PRs and bugs on the helm chart repo did end up getting me. The [read only filesystem bug](https://github.com/Koenkk/zigbee2mqtt-chart/issues/4) was a killer so I bumped the thread to see if it would be fixed. For now I'm going to wait and see since I'm not in a rush and this is the official chart. 

## Test Adding a Device

Now to test adding a device.

## Setting Up Two or Having a Backup?

I have two of these adapters. My initial plan was to use one upstairs and the other on the main floor while I used the hue bridge in the basement for it's entertainment zones. However, I want to see if it's easy to switch between the two with the same Z2M instance without doing some crazy re-paring every device thing. 

### Adapter Swap

## Ingress and DNS Entries

At this point I assume we've decided on one or two so we can move forward with mapping this to a nice, secure, local domain with SSO.

## Pass to Home Assistant via MQTT Integration