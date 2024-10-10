---
title: Hardening
permalink: /docs/smart-home/hardening/
---

Notes for how to harden IoT stuff to increase reliability.

1. Need to update firmware on both TubeZB radios just before go-live
1. Capture errors that are fixed by rebooting controllers and automate the reboot 

Z-Wave crapped out on 10/6 throwing this:

```
Logging to file:
        /usr/src/app/store/logs/zwavejs_2024-10-05.log
2024-10-05 23:42:53.381 INFO Z-WAVE-SERVER: Server closed
```