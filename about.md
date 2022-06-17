---
layout: page
title: About
permalink: /about
---

This is project aims to completely reverse engineer the cloud service for the Sure Petcare or SureFlap "Connect" internet enabled pet devices and to provide a local "cloud" for the Connect Hub to connect to rather than using their cloud service. This way the data on your pets movements never leave your network and you are in total control of all your own data.

It was born out of frustration with the cloud service being unstable over Christmas 2020/January 2021 and from a privacy perspective I prefer to not to have all my animals movements being sent off to their cloud service.

This integrates to Home Assistant using MQTT Discovery so a custom Home Assistant integration isn't required and all the entities turn up automatically if MQTT is enabled in Home Assistant. It's fortunate that the Hub also talks MQTT to AWS IoT MQTT endpoint so I have designed PetHubLocal to use a Home Assistant Mosquitto MQTT Broker as the shared broker for the hub too. The issue is the hub needs to connect to a MQTT TLS endpoint with Client Certificate Authentication enabled.

It has been through various refactors and rebuilds to where it is now. Patches are always welcome.

# How it works

Below I document how the current and altered boot process works.

## Standard Hub boot process

The standard flow the hub boot process is:
- Sync Time via NTP to `pool.ntp.org`
- `HTTPS` Connect to SurePetCare Cloud service on port `443` to `hub.api.surehub.io` to retrieve credentials and client certificate
- `MQTT TLS` with Client Certificate to `a5kzy4c0c0226-ats.iot.us-east-1.amazonaws.com` AWS MOTT IoT endpoint, this hostname is returned from the above response.
- `HTTP` for firmware updating the Hub to SurePetCare Cloud service on port `80` to `hub.api.surehub.io`

![Standard-SurePet](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://pethublocal.github.io/assets/SurePet.iuml)

## Pet Hub Local Hub process

### DNS Posion - Repoint `hub.api.surehub.io` locally

For pethublocal to work you **MUST** to update your local DNS server to respond with a different IP address for:
```
hub.api.surehub.io
```
Many routers support having local DNS entries or if you are running OpenWRT it is easy, however many ISP supplied routers do not support this feature so you may need a new router to poison the DNS entry. A [Mikrotik RB931-2nD](https://mikrotik.com/product/RB931-2nD) is a low-cost option and should work fine, but there are many cheap routers out there that can do it and I am not going to discuss how to set it up, so if you don't feel comfortable setting it all up then this project isn't for you.


## Setup - Get everything setup

Download `start` and pre-cache Credentials and Firmware:

![PetHubLocal Setup](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://pethublocal.github.io/assets/PetHubLocal-Setup.iuml)

Using PetHubLocal you can login to the SurePet Browser API to retrieve `https://app.api.surehub.io/api/me/start` which contains your profile and all devices including the Hub Serial Number and MAC Address and pets. Using this forms the basis of `pethubconfig.json` which is the configuration file holding everything.

Using `start_to_pethubconfig` function in `functions.py` where it maps the `start json` into the `pethubconfig.json` format.

Also download `https://hub.api.surehub.io/api/credentials` which is requested by the Hub on boot, and includes the Client Certificate the hub needs to use to connect to AWS. The client certificate is added to `pethubconfig.json` as it is needed each time the hub boots.
Lastly download the Hub firmware from `http://hub.api.surehub.io/api/firmware` which is 77 individual files the hub downloads when flashing the Hubs firmware. This is needed if you are running newer firmware than `2.43` as you need to re-flash the Hub with the `2.43` older firmware over the current firmware version. When downloading the firmware the PetHubLocal code finds the XOR key and `long_serial` so soldering the console logs and watching the firmware update process to get the password aka `long_serial` is no longer required to use the [Client Certificate](/certificate) which is quite useful to have.

- Want to generate your own Client Certificate rather than using the AWS one
- Me In The Middle (not Man in the Middle) the traffic from the Hub to the legitmate cloud using [PolarProxy](/polarproxy)
- Just because its useful to have and who knows it might come in handy in the future as SurePet may remove this *feature* in a future firmware update

## Config file `pethubconfig.json`
This is the single configuration file everything is put into. It supports multiple hubs, stores the credentials.

```
{
    "Config": {
        "Deployment": "Setup",
        "Timezone": "Local",                                <- Print time in UTC or Local
        "Last_HA_Init": 1653171031,                         <- Last time Home Assistant entities were inited
        "Get_State": true,                                  <- Download state from devices on startup
        "Last_Updated": "2022-05-22 10:10:32",
        "Cloud": {
            "LoggedIn": true,                               <- Has logged into SurePet Cloud
            "Username": "user",                             <- Email address
            "Password": "password",                         <- Password, you may want to blank this out
            "device_id": "...",                             <- UUID Device ID generated for auth
            "Token": "...",                                 <- JWT to login to SurePet Cloud
            "StartJSON": "start-yyyymmdd-hhmmss.json"       <- Start json downloaded
        },
        "Web": {
            "Host": "0.0.0.0",                              <- IP Address to listen to for the http and https web server
            "HTTPPort": 80,                                 <- HTTP Port for Hub Firmware update and Browser interface, must be 80 for the Hub.
            "HTTPSPort": 443,                               <- HTTPS Port for Hub Credentials call, must be port 443 for the Hub.
            "Cert": "hub.pem",                              <- HTTPS certificate
            "CertKey": "hub.key"                            <- HTTPS Private Key for certificate
        },
        "MQTT": {
            "Host": "127.0.0.1"                             <- MQTT Broker that PetHubLocal connects to via port 1883, also given to the Hub in credentials
        },
        "Firmware": {
            "Cache": true,                                  <- Cache Firmware - To be added
            "Force_Download": false                         <- Force Download of Firmware / Credentials - To be added
        }
    },
    "Devices": {
        "H0xx-0xxxxxx": {                                   <- Provisioned Hub Serial Number
            "Hub": {
                "Name": "Hub Name",                         <- Hub Name
                "Product_Id": 1,
                "Serial_Number": "H0xx-0xxxxxx",
                "Mac_Address": "0000xxxxxxxxxxxx",
                "Updated_At": "..",
                "Index": "Hub",
                "Led_Mode": 0,                              <- LED Option for Off, Dimmed or Bright
                "Pairing_Mode": 0,                          <- Adoption mode to adopt new devices
                "Main_Version": "204",
                "State": "Online",
                "Device": {
                    "Hardware": "3",
                    "Firmware": "2.43"
                },
                "Registers": "01..00",
                "Uptime": "1020",
                "UUID": "...",
                "Client_Cert": "MII...A==",
                "Reconnects": "1"
            },
            "yyyyyyyyyyyyyyyy": {                           <- Device MAC Address
                "Name": "Device Name",                      <- Name
                "Product_Id": [3|4|6|8],                    <- Device Type
                "Serial_Number": "xxxx-0xxxxxx",
                "Mac_Address": "yyyyyyyyyyyyyyyy",          <- Device MAC Address
                "Index": 2,                                 <- Hub Provisioned Device Index Number 
                "Updated_At": "..",
                "Send_Counter": 20,                         <- Counter for sending messages to Device for Cat Flap, Feeder and Poseidon
                "Receive_Counter": 0,                       <- Recieve Counter
                "Last_Device_Update": xxx,                  <- Last time device was updated
                "Fast_Polling": false,
                "Main_Version": "xx",
                "Battery": "x.xxx",                         <- Battery value
                "BatteryADC": 0,                            <- Battery value if returning an ADC value
                "Last_Heard": "-453479495",
            }
        }
    },
    "Pets": {
        "ffffffffff": {                                     <- HDX Tag
            "Name": "HDX Tag Name",
            "Species": 0,
            "Activity": {
                "Where": "Inside",
                "Time": ".."
            }
        },
        "ccc.nnnnnnnnnnnn": {                               <- FDX-B Tag
            "Name": "FDX-B Tag Name",
            "Species": 1,                                   <- Species - Cat = 1, Dog = 2
            "Activity": {
                "Where": "Inside",
                "Time": ".."
            },
            "Feeding": {
                "Change": [
                    "-0.23",                                <- Last feed left weight
                    "-0.05"                                 <- Last feed right weight
                ],
                "Time": "."
            },
            "Drinking": {
                "Change": [
                    -2.51                                   <- Last drink weight
                ],
                "Time": "2021-12-29T16:07:02+00:00"
            }
        }
    }
}
```

There are more fields in the above `pethubconfig.json` but it should make sense.

### Start - Run PetHubLocal

Once you have a `pethubconfig.json` with everything in it and the MQTT Broker is configured to listen on `8883` you are ready to start.

![PetHubLocal Setup](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://pethublocal.github.io/assets/PetHubLocal.iuml)

Using [Home Assistant MQTT Discovery](https://www.home-assistant.io/docs/mqtt/discovery/) a number of MQTT Topics are published to with a JSON payload to configure Home Assistant.

Using `ha_init_entities` function in `functions.py` to create all the Home Assistant entities with topics under `homeassistant/[switch|sensor]/pethub/H0xx../config`

The following topics are published to with the MQTT retain flag set to true so they persist in Home Assistant after restarts depending on the device type:

#### Hub
- `homeassistant/sensor/pethub/H0xx-0xxxxxx_Hub/config` - Where xxx is the Hub Serial Number and will hold all Hub related state information

#### All connected devices but not the Hub
Device MAC Address = yyyyyyyyyyyyyyyy
- `homeassistant/sensor/pethub/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy/config` - Main Device state information
- `homeassistant/sensor/pethub/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy_battery/config` - Device battery state

#### Pet Door and Cat Flap
The doors have the Keep In / Out / Curfews as switches to toggle them in HA they need to be added as switches not sensors. 
- `homeassistant/switch/pethub/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy_keepin/config` - Keep In Switch and State
- `homeassistant/switch/pethub/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy_keepout/config` - Keep Out Switch and State
- `homeassistant/switch/pethub/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy_curfew/config` - Curfews Enabled Switch and State

#### Feeder

If the feeder has two bowls
- `homeassistant/sensor/pethub/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy_left_weight/config` - Left Scale weight
- `homeassistant/sensor/pethub/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy_right_weight/config` - Right Scale weight

If the feeder has a single bowl
- `homeassistant/sensor/pethub/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy_weight/config` - Scale weight

#### Poseidon aka Felaqua
- `homeassistant/sensor/pethub/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy_weight/config` - Scale weight

#### Pets
- `homeassistant/sensor/pethub/hhhhhhhhhh/config` - HDX Pet Tag which is a 5 byte hex value in the format `hhhhhhhhhh`. The "collar tags" included by SurePet in the Pet Door boxes are HDX.
- `homeassistant/sensor/pethub/ccc-nnnnnnnnnnnn/config` - FDX-B Pet Tag which is a 3 and 12 digit value separated with a `.` in the format `ccc.nnnnnnnnnnnn` which is the modern injected microchips used in animals with 3 digits for the Country Code, and 12 digits for the National Code which is unique per microchip.

#### Home Assistant MQTT Discovery Payload

Within each of the above `/config` topics a retained json message is published in the Home Assistant MQTT Discovery format, it changes slightly between the device types or pets as I set the Icon according to the device type and for switches , but you get the idea.

```
{
    "name": "Device Name",                                           <- Device or Pet name
    "ic": "mdi:door",                                                <- Device or Pet Icon 
    "uniq_id": "H0xx-0xxxxxx_yyyyyyyyyyyyyyyy",                      <- Home Assistant Unique ID 
    "stat_t": "pethub/ha/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy/state",       <- State Topic, this is single topic holding the state
    "val_tpl": "{{value_json.State}}",                               <- The JSON element in the above topic for this sensor aka State
    "json_attr_t": "pethub/ha/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy/state",  <- The "Attributes" dropdown on the entity
    "cmd_t": "pethub/ha/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy/KeepIn",       <- Only for doors with HA switch entities for HA to switch the KeepIn/Out/Curfew state
    "avty_t": "pethub/ha/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy/state",       <- Availability Topic for online/offline state, only added to devices not pets
    "avty_tpl": "{{value_json.Availability}}",                       <- Availability Template using the JSON element Availability, only for devices
    "device": {
        "ids": "yyyyyyyyyyyyyyyy",                                   <- The below groups all entities as a single device in Configuration
        "name": "Device Name",                                       <- -> Settings -> MQTT so they all appear as one panel as well
        "sw": 2,
        "mdl": "Device Type",
        "mf": "Pet Hub Local"
    }
}
```

### Pet Hub States

Now all the entities are created in Home Assistant they are expecting a `pethub` specific device entity Topic. This is a single json topic that holds the whole state of the device. These are also set to retain so that the entities are persisted and their state is known after restarts of MQTT or Home Assistant.

- `pethub/ha/H0xx-0xxxxxx_Hub/state` - The Hub State
- `pethub/ha/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy/state` - The Device State
- `pethub/ha/hhhhhhhhhh/state` - HDX Pet Tag state
- `pethub/ha/ccc-nnnnnnnnnnnn/state` - FDX-B Pet Tag state

For door devices the switches are created so the Command Topics `cmd_t` to `ON/OFF` when you change the state of the switch in Home Assistant.

- `pethub/ha/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy/KeepIn` - Change the Door KeepIn State
- `pethub/ha/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy/KeepOut` - As above, but for KeepOut
- `pethub/ha/H0xx-0xxxxxx_yyyyyyyyyyyyyyyy/Curfew` - Same again but for Curfews

#### Pet Hub State Payload

Within each of the above `/config` topics a retained json message is published in the Home Assistant MQTT Discovery format, it changes slightly between the device types or pets as I set the Icon according to the device type and for switches , but you get the idea.

##### Hubs
Fairly obvious values specific to the hub, that is also displayed under `Attributes` drop down on the entity due to `json_attr_t`
```
{
    "Availability": "online",
    "State": "Online",
    "Uptime": "xxx Mins",
    "Name": "Hub Name",
    "Reconnects": "x",
    "Serial": "H0xx-0xxxxxx",
    "MAC Address": "0000111111111111",
}
```

##### Devices
Device specific json where some of the below values are added only on if the device is a door or feeder. 
```
{
    "Availability": "online",        <- Availability topic if the device goes offline
    "State": "Closed / KeepIn",      <- State shown on the main device itself, for Feeders it's Open/Closed, for Doors it's Unlocked/KeepIn/KeepOut etc
    "Online": "Online",              <- For Poseidon showing if it is online for the device as it doesn't have a state
    "Battery": "x.xxx",              <- Battery voltage level
    "KeepIn": "OFF",                 <- Current Door KeepIn State, this changes when the KeepIn is enabled showing the switch on / off
    "KeepOut": "OFF",                <- Current Door KeepOut State, same as above but KeepOut
    "Curfew": "OFF",                 <- Current Door Curfew State, same as above but Curfew
    "Curfews": "07:00-08:45",        <- The Curfew lock and unlock values if set for the door
    "Bowl Count": 2,                 <- Feeder bowl count
    "Close Delay": "FAST",           <- Feeder close delay
    "Left Target": "50",             <- Feeder left target weight showing where the LED indicate goes to
    "Right Target": "55",            <- Feeder right target weight showing where the LED indicate goes to
    "Left Weight": "4.6",            <- Feeder left current reported weight
    "Right Weight": "3"              <- Feeder right current reported weight
}
```

##### Pets
As above only if the pet is assigned to a Door, Feeder or Poseidon
```
{
    "State": "Inside",         <- State where the Animal is if they came inside, went outside or looked in either direction 
    "Left Weight": "-0.23",    <- Last feed from left bowl
    "Right Weight": "-0.05",   <- Last feed from right bowl
    "Drinking": "-2.51"        <- Last drink from Poseidon
}
```

### Web Frontend

The frontend is a basic HTML page with SocketIO to support publishing every event to the site as well as planned end-user features to support adopting new devices, changing Curfew times for doors etc. Still a work in progress as I was working on the core code first.

