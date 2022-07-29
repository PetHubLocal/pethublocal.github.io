---
layout: page
title: Hub Firmware
permalink: /firmware
---

# Background

Back in 2021 when I started this project the firmware on the hubs was `2.43` and it didn't check the certificate of the hosts it was connecting to so it was easy to poison the DNS and the hub would just connect to PetHubLocal. A few months after I published on the Home Assistant community about building PetHubLocal SurePetCare released an updated firmware version `2.201` a notable change on this was now the hub **DOES** check the certificate. I am not sure if I had anything to do with that change.

<p align="center">
<img src="/assets/smile.gif" height="150">
</p>

Moving on....

If you want to use PetHubLocal you will need to be running firmware to `2.43`.

# Checking your firmware version from the config

If you check the `pethubconfig.json` once the configuration is built under the hub serial number you will have the firmware version in there.

```
    "Devices": {
        "H0xx-0xxxxxx": {
            "Hub": {
...
                "Device": {
                    "Firmware": "2.43"
```

Or from the app API response to https://app.api.surehub.io/api/me/start which is retrieve by the app

```
{
    "data": {
        "devices": [
            {
                "product_id": 1,
                "serial_number": "H0xx-0xxxxxx",
...
                "status": {
                    "version": {
                        "device": {
                            "firmware": 2.43
```


If your Firmware above is not `2.43` and `2.201` or anything higher then you will need to flash on the older firmware to make it work. Thankfully I have made this as straight forward a process as possible as during the setup when downloading the current firmware from SurePetCare the script finds the XOR key for that firmware as the project includes the DeXORed `2.43` firmware and builds a hub specific image with the XOR key just found.

After the configuration is built using `pethublocal setup` the current directory should have two groups of firmware images:

```
H0xx-0xxxxxx-2.43-nn.bin
```
and

```
H0xx-0xxxxxx-1.177-nn.bin
```

Where `H0xx-0xxxxxx` is your Hub Serial Number and `2.43` version of files is the custom firmware built to your Hub, and `1.177` is what was downloaded from SurePet and then the XOR key found as `1.177` is the version of the bootloader on the hub, which I hope SurePetCare don't upgrade to lock me out of doing this :pray:

The Hub requests the firmware over `HTTP` on port `80` to `hub.api.surehub.io` so we are in luck as it doesn't check the certificate and will download fine from PetHubLocal... hence why we can downgrade and upgrade without the issue of the need to validate a Certificate over HTTPS.

# Downgrade firmware to 2.43

If you unplug the power for the hub then hold down the reset button underneath the hub and plug it back in again and release the reset button when the ears are solid red as then the hub will download the locally built firmware from `hub.api.surehub.io` which is pointing to your PetHubLocal right?? PetHubLocal checks the local directory based on the Hub Serial Number and serves up the already built version `2.43` if the `H0xx-0xxxxxx-2.43-nn.bin` files exist.

The upgrade / downgrade takes about 5 minutes so just leave it alone.

It’s **REALLY** important you don’t interrupt the firmware process as I don’t want you to brick your hub, and with everything you are taking this risk on yourself if things go wrong… which I have tried as hard as I can to avoid anything going wrong.

# Roll back to SureFlap supplied firmware

If you decide you don’t want PetHubLocal and want to return to the firmware downloaded from the cloud you can move the `H0xx-0xxxxxx-2.43-nn.bin` files somewhere else and leave just the `H0xx-0xxxxxx-1.177-nn.bin` files in place, repeat the firmware upgrade process again with the reset button and it will flash back on the `H0xx-0xxxxxx-1.177-nn.bin` firmware back to `2.201` or whatever version of firmware was downloaded from the clouds `hub.api.surehub.io` when you setup PetHubLocal.

Again it's really important not to interrupt the firmware update process as you don't want to brick the Hub.

# iHB vs iHB v2

So, the folks at SurePetCare have been busy and built a completely new version of the hub called the "iHB v2" I have not seen one in the wild but it is on the FCC Website

## iHB V1
My photos of the V1 Hub:

<p align="center">iHB V1 Front<br/> 
<img src="/assets/Front.jpg" height="350"><br/>
iHB V1 Back<br/>
<img src="/assets/Back.jpg" height="350">
</p>

## iHB V2
FCC Site: https://fccid.io/XO9-IHB002

<p align="center">V2 Front<br/> 
<img src="/assets/iHBV2-Front.jpg" height="350"><br/>
V2 Back<br/>
<img src="/assets/iHBV2-Back.jpg" height="350">
V2 Mounted<br/>
<img src="/assets/iHBV2-Mount.jpg" height="350">
</p>

So it's completely different but looks exactly the same from the outside, and seems to run an Arm Cortext MIMXRT1021 so... all bets are off with this if you have one.
