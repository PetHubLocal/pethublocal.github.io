---
layout: home
---

PetHubLocal is a **complete** local replacement for your SurePetCare Connect series of IoT enabled Pet Devices cloud service but running all the services **locally** integrating to Home Assistant or other Home Automation as I have tried to make it as independent of Home Assistant where possible. This mean your hub will no longer connect to the cloud service so the official Sure Pet App will show the hub as offline and no data leaves your network.

This is done by poisoning the DNS entry `hub.api.surehub.io` which is hard-coded in the Hub firmware to point to a local web server instead.

There are two ways to deploy `pethublocal` either as a standalone python 3.8 or higher application or as a [Home Assistant Add-on](https://www.home-assistant.io/addons/)

# [Setup PetHubLocal](/setup)
Follow the above setup instructions to setup PetHubLocal

## Core Requirements:

- DNS Poisoning of `hub.api.surehub.io` using various methods such as PiHole, Mikrotik Router or OpenWRT router 
- Port 80 and 443 to be available on the host you want to run it on to be used by PetHubLocal for the hub boot. It's a pain I know, but there is no way around it as the hub connects on these ports.
- Mosquitto MQTT broker listening on port 8883 with TLS enabled.
- Running firmware `2.43`. Many hubs are running newer firmware build of `2.201` or higher, so you will need to downgrade the hub to `2.43` as the new firmware actually *does* check the certificate is valid :cry: so refer to [Hub Firmware](/firmware) for the downgrade process.

This is a work in progress and have parked it from late 2021 until May 2022, so there are bugs, but it works for me. :)

## Supported SurePetCare Devices

I have added support for all currently available SurePetCare devices I have, as I have them all.

- Internet Hub Connect
- Pet Door Connect
- Feeder Connect
- Cat Flap Connect
- Poseidon (Felaqua) Connect. I'm calling it Poseidon as that is a way cooler name than Felaqua.

