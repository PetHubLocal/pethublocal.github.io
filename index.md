---
layout: home
---

The local replacement for your SurePetCare Connect series of IoT enabled Pet Devices so your hub doesn't connect to the cloud service and no data leaves your network.

This is done by poisoning the DNS entry `hub.api.surehub.io` which is hard-coded in the Hub firmware to point to a local web server instead.

# [Setup PetHubLocal](/setup)
Follow the about setup instructions to setup PetHubLocal

## Core Requirements:

- DNS Poisoning of `hub.api.surehub.io`
- Port 80 and 443 to be available and able to be used by PetHubLocal for the hub boot. It's a pain I know, but it is for the best.
- Mosquitto MQTT broker listening on port 8883 with TLS enabled

This is a work in progress and have parked it from late 2021 until May 2022, so there are bugs, but it works for me. :)

## Supported SurePetCare Devices

I have added support for all currently available SurePetCare devices I have, as I have them all.

- Internet Hub Connect
- Pet Door Connect
- Feeder Connect
- Cat Flap Connect
- Poseidon (Felaqua) Connect. I'm calling it Poseidon as that is a way cooler name than Felaqua.

