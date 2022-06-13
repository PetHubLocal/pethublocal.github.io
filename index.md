---
layout: home
---

The local replacement for your SurePetCare Connect series of IoT enabled Pet Devices so your hub doesn't connect to the cloud service and no data leaves your network.

This is done by poisoning the DNS entry `hub.api.surehub.io` which is hard-coded in the Hub firmware to point to a local web server instead.

There are two ways to deploy `pethublocal` either as an [Home Assistant Add-on](https://www.home-assistant.io/addons/)

# [Setup PetHubLocal](/setup)
Follow the above setup instructions to setup PetHubLocal

## Core Requirements:

- DNS Poisoning of `hub.api.surehub.io`
- Port 80 and 443 to be available and able to be used by PetHubLocal for the hub boot. It's a pain I know, but it is for the best.
- Mosquitto MQTT broker listening on port 8883 with TLS enabled
- Running firmware `2.43` and not `2.201` or higher as many hubs seem to have been upgraded to new firmware that actually *does* check the certificate is valid :cry: so please refer to [Hub Firmware](/firmware) for more details on the process

This is a work in progress and have parked it from late 2021 until May 2022, so there are bugs, but it works for me. :)

## Supported SurePetCare Devices

I have added support for all currently available SurePetCare devices I have, as I have them all.

- Internet Hub Connect
- Pet Door Connect
- Feeder Connect
- Cat Flap Connect
- Poseidon (Felaqua) Connect. I'm calling it Poseidon as that is a way cooler name than Felaqua.

