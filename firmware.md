---
layout: page
title: Hub Firmware
permalink: /firmware
---

# Background

Back in 2021 when I started this project the firmware on the hubs was `2.43` and it didn't check the certificate of the hosts it was connecting to so it was easy to poison the DNS and the hub would just connect to my local stack. A few months after that an updated firmware was released by SureFlap on some hubs with version `2.201` a notiable change on this was now the hub **DID** check the certificate. I am not sure if I had anything to do with that change.

<p align="center">
<img src="/assets/smile.gif" height="150">
</p>

Moving on....

So now if you want to use PetHubLocal and your hub is already upgraded to `2.201` you will need to downgrade the firmware to `2.43`. Thankfully I have made this as straigthforward as I can by building a firmware image specific to your Hub based on the Serial Number and the XOR key found in your current firmware image.

# Checking your firmware version from the config

If you check the `pethubconfig.json` once the configuration is built under the hub serial number you will have the firmware version in there.

```
    "Devices": {
        "H0xx-0xxxxxx": {
            "Hub": {
...
                "Device": {
                    "Hardware": "3",
                    "Firmware": "2.43"
                },
```

If your Firmware above is not `2.43` and `2.201` or anything higher then you will need to flash on the older firmware to make it work.

This is an area I need to document further but if you check the `pethublocal.log` it should show you that your firmware is the older version and the `2.43` firmware has been built for you.

In the configuration folder there are two groups of firmware images:

```
H0xx-0xxxxxx-2.43-nn.bin
```
and

```
H0xx-0xxxxxx-1.177-nn.bin
```

The `2.43` one is the custom firmware built to your Hub, and `1.177` is what was downloaded from SurePet and then the XOR key found as `1.177` is the version of the bootloader on the hub, which I hope SurePetCare don't upgrade to lock me out of doing this trick :pray:.

The Hub talks `HTTP` on port `80` to `hub.api.surehub.io` to download the firmware so we are in luck as it doesn't check the certificate and will download fine... hence why we can downgrade and upgrade without the issue of the Certificate getting in the way.

# Upgrade process

If you boot the hub while holding down the reset button underneath then the hub will downgrade to `2.43` and be able to run locally.

It’s **REALLY** important you don’t interrupt the firmware process as I don’t want you to brick your hub, and with everything you are taking this risk on yourself if things go wrong… which I have tried as hard as I can to avoid anything going wrong.

# Roll back to SureFlap supplied firmware

If you decide you don’t want PetHubLocal anymore you can move the `H0xx-0xxxxxx-2.43-nn.bin` files somewhere else and leave just the `H0xx-0xxxxxx-1.177-nn.bin` files in place, repeat the process again and it will flash back on the original firmware back to `2.201` or whatever version of firmware was downloaded from SurePetCare when you setup PetHubLocal.
