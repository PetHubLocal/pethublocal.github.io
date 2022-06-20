---
layout: page
title: Device Documentation
permalink: /devices
---

Documenting MQTT messages from the hub

1. [Background](#background)
1. [Hub](#hub)
2. [Pet Door](#pet-door)
4. [Cat Flap](#cat-flap)
4. [Feeder](#feeder)
5. [Felaqua (Poseidon)](#poseidon)

# Background

Observations and learnings from the SurePetCare Connect series. Focused on the network flows from the Hub to the cloud including HTTPS request for Credentials and the MQTT AWS as well as a hardware teardown of the hub.

FCC Main Page: https://fccid.io/XO9

# Hub

Hub V2 FCC https://fccid.io/XO9-IHB002 (Don't currently have one of these hubs)


The hub is where all my effort has focused, as most likely you already own a the Hub so why not use it rather than my other attempt at building a whole hub replacement.

## Credentials

The Hub boot process:

![Standard-SurePet](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://pethublocal.github.io/assets/SurePet.iuml)

- Powers on and ears alternate flash red showing it is booting.
- Syncs NTP time to `pool.ntp.org`
- Does a *x-www-form-urlencoded* **POST** to `https://hub.api.surehub.io/api/credentials` with Serial Number and Mac Address to retrieve the credentials.

With the *semi* recent build of the Hub Firmware to `2.201` the hub now checks the certificate it is connecting to is legimate, which is annoying since just posioning DNS was sufficient, but now you must also downgrade firmware to `2.43` which the old firmware build process is included. So after the initial build has happened downloading the `start` information the script then downloads all firmware and builds the firmware.

### Example CURL to retrieve credentials
```
fw=2.43
serialnumber=H0xx-0xxxxxx
macaddress=0000xxxxxxxxxxxx
curl -v -k -d "serial_number=$serialnumber&mac_address=$macaddress&product_id=1&firmware_version=$fw" -H "curl/7.22.0 (x86_64-pc-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3" -H "Content-Type: application/x-www-form-urlencoded" -X POST -o $serialnumber-$macaddress-$fw.bin https://hub.api.surehub.io/api/credentials
```

The response comes back as `text/html; charset=utf-8` with the payload:
If the HTTP Header `X-Upgrade: 1` is included in the HTTP response then it forces a Firmware Update.

```
v02:ssssss:uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu:::1:v2/production/uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu:a5kzy4c0c0226-ats.iot.us-east-1.amazonaws.com:MIIKL...Client Cert...AgIIAA==
```

The credentials response contains a colon `:` delimited file which is used to configure the hub each time it boots, it also means that to run completely offline you need a backup of the credentials file.

The decoded credentials are also printed to the console output when the hub boots.

```
-------- Credentials Decoded --------
	Host:		a5kzy4c0c0226-ats.iot.us-east-1.amazonaws.com
	Client ID:	uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu
	Base Topic:	v2/production/uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu
	Version:	v02
	ID:		ssssssuuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu
	Username:	
	Password:	
	Network Type:	1
	Certificate:	MIIKL...Client Cert...AgIIAA==
 	Length:		2611
	Cert Hash:	0x..
	Key Hash:	0x..
-------- End Credentials --------
```

The fields 

| Field | Example Value | Description |
| - | - | - |
| 0 | v02 | Version, which is always 'v02' |
| 1 | ssssss | ID which is serial number after the `-` with no leading zeros IE H010-0123456 the serial would be 123456 |
| 2 | uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu | Client ID which is just an AWS issued UUID for the MQTT Client ID |
| 3 |  | MQTT Username, which is blank but can be set if your local MQTT broker requires authentication |
| 4 |  | MQTT Password, which is also blank but can be set as required |
| 5 | 1 | Network Type, always set to 1 |
| 6 | v2/production/uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu | Base Topic, this is updated to `pethub/hub/H0xx-0xxxxxx` for pethublocal |
| 7 | xx | Certificate a base64 encoded PKCS12 AWS issued Client Certificate with a **password** that is hard-coded into the hub flash |

Once the credentials file has been decoded then the ears change from alternating red, to alternating green.

## MQTT Messages

- The Base Topic in the `credentials` response in field #6 from the cloud points to AWS IoT Base topic, they use a AWS generated `UUID` for both the Client ID and part of the Base Topic `v2/production/uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu`
- All messages are sent with MQTT QoS `1`, rather than `0`
- Whenever a hub sends a message the cloud replays exactly the same message back to the hub. I have no idea why, but it does. The Hub doesn't stop working if it doesn't retrieve a replay of the message back so I don't do that in PetHubLocal.

Messages from the hub:
- Base Topic + `/messages` - Hub messages
- Base Topic + `/messages/yyyyyyyyyyyyyyyy` - Device messages where `yyyyyyyyyyyyyyyy` is the MAC Address of the device.

Publish messages `mosquitto_pub`:
- `mosquitto_pub -q 1 -t 'pethub/hub/H0xx-0xxxxxx/messages' -m '62852aad 1000 ...'`
- `mosquitto_pub -q 1 -t 'pethub/hub/H0xx-0xxxxxx/messages/yyyyyyyyyyyyyyyy' -m '62852aad 1000 ...'`

## Last Will
The Last Will Message is a non-standard messages. When the Hub Connects it publishes the last will to the `*base topic* + /messages` topic with the message:

- `Hub has gone offline (having been online since dd hh mm ss)`

Where dd is the day of the month. and hours, mins and seconds in UTC time, not zero padded. This way SurePet / AWS knows when the hub goes offline if you unplug the hub or the network gets interrupted the last will message will be sent.

## Common message characteristics  
- The messages are space delimited with a fairly standard message format with a few quirks which is to be expected.
- Time - All time is in `UTC` with the exception of the Pet Doors which set the curfew in Local Time as that is what is displayed on the LCD screen.
- The command vs status message messages are determined by the counter being either `0xx0` for a status message which is a single byte counter for `xx` or `1000` for the command message to send an action to the hub.
- Message Counters - Are either single or double bytes.
- Two classes of devices, Hubs + Pet Doors and everything else.
  - The Hub and Pet Door send *registers* so have memory / register offset plus the register value that is changing or being set. These are sent as `132` messages. 
  - All other devices, Cat Flap, Feeder and Posedion all send `126` and `127` message type payloads where `126` is a status multi-message with message length and message type and there can be multiple messages sent in a single payload, and `127` is a command message with just message type, and there is no length as only one message can be sent in a command message.
- You will also notice the Last Will message doesn't have a timestamp or counter. Since, you know, why have a standard message format when you don't need to.

| Offset | Example | Description |
| - | - | - |
| 1 | `626e685b` | Event Timestamp in UTC hex format |
| 2 | `0010` or `1000` | Status or Command message where status is `0xx0` and command is `1000` for messages from devices or the hub, or making changes |
| 3 | `2` or `3` or `8` or `10` or `126` or `127` or `132` | The message type depending on which device is generating it |
| 4+ | ... | Depends on the payload |

## Hub message characteristics
Specifics of the Hub message ontop of [Common message characteristics](#common-message-characteristics)
Hub Message Type:
- `2` Command to set multiple bytes to the Hub
- `3` Command to query registers
- `10` Status to report Hub Uptime hourly
- `132` Status register update

## Hub Example messages
| Type | Example | Description |
| - | - | - |
| CMD - Flash Hub LEDs | `6283d4fb 1000 2 18 1 80` | Send command to set register `18` one `1` byte to `80`, which causes the Ears to flash green three times, then go to off |
| CMD - Query Registers | `6283d4fb 1000 3 0 205` | Send command to query the Hub Registers from offset `0` to offset `205` |
| Status - Hub Uptime | `6283d4fb 0020 10 00003480 01 12 34 56 6283d4fb 1` | Counter `0020`, Uptime Type `10`, Decimal minutes up `00003480`, UTC time `01 12 34 56` - `dd hh mm ss`, Reconnect counter `1` increments each time hub reconnects to MQTT |
| Status - Hub Online | `6283d4fb 0000 Hub online at 17 17 1 47` | First boot message when the hub boots, the counter is `0000` and the time stamp is `mm hh mm ss` in string format |
| Status - Hub Register Update | `6283d4fb 0010 132 0 0 8 01 cd 00 02 00 2b 00 03` | First register message, `0010` Status counter, `132` register, counter `0`, offset `0` and length of `8` with the registers `0 through to 7` being sent |

## Hub 132 Register messages
| Offset | Example | Description |
| - | - | - |
| 3 | `132` | `132` register status update |
| 4 | `1` | Single byte decimal counter that goes from `0` to `254` before repeating, not zero padded |
| 5 | `0` to `205` | Starting decimal register offset, this is the starting offset of the register not zero padded |
| 6 | `1` to `8` | Payload length, from the above register, the proceeding bytes are the reigster(s) changing not zero padded, maximum length of 8 for each message |
| 7+ | `xx` | Payload data in Hex byte, this is zero padded for the hex value |

With the above understanding of the different message formats, the registers sent in the 132 messages are documented below from my understanding.

| Register | Description |
| -- | -- |
| 3-6 | Firmware Version, High value 3&4, Low value 5&6 so 2.43 = 02 00 2B 00 |
| 7 | Hardware Version |
| 15 | Adoption Mode to adopt new devices - Disabled (0), Enabled (2), Adopt using bottom Button (0x82) |
| 18 | Flash LEDs - Off (0), Bright (1), Dimmed (4), Flash then Off (0x80), Flash then Bright (0x81), Flash then Dimmed (0x84) |
| 23 | MIWI Channel Noise Floor |
| 24 | RF Channel Register, Channel 1 - 15 (0f), Channel 2 - 20 (14), Channel 2 - 26 (1a) should be always 0x0f | 
| 31-34 | Device uptime |
| 44 | Number of registered devices |
| 45-52 | Registered Device 0 Mac Address - 8 Bytes |
| 53-60 | Registered Device 0 State - 8 Bytes |
| 61-68 | Registered Device 1 Mac |
| 69-76 | Registered Device 1 State |
| .. | Device 2 - 8 |
| 189-196 | Registered Device 9 Mac |
| 197-204 | Registered Device 9 State |

### Device Status
The 8 Bytes for each device after the Mac address can be further broken down as:
| Byte | Description |
| -- | -- |
| 0 | Status, First bit is Valid True/False, Second bit Online / Offline, Bit 3+ Product ID ie 3 for Pet Door |
| 1-3 | RSSI Value I guess, still haven't figured this out |
| 4-7 | Last heard, signed integer which is displayed on the serial console as well. Some timestamp  |

## Hub Teardown
Hub pins

| Component | Description |
| -- | -- |
| CPU |PIC32MX695F512H - PIC CPU in 64 Pin (H version) TQFP form factor |
| Radio | MRF24J40 - Microchip Zigbee / MiWi radio controller |
| Ethernet | ENC424J600 Ethernet controller |
 
### Datasheets
Below is links to the datasheets found on the Microchip site, these links may break so the PDFs are also downloaded and included here.

#### PIC32MX
There are two revisions of the PIC32MX and I find the v2 better.

Datasheets
- https://ww1.microchip.com/downloads/en/DeviceDoc/PIC32MX_Datasheet_v2_61143B.pdf - PIC32MX_Datasheet_v2_61143B.pdf
- https://ww1.microchip.com/downloads/en/devicedoc/pic32mx_datasheet_v1_61143a.pdf - Also V1 if required (for completeness)
- https://www.microchip.com/content/dam/mchp/documents/MCU32/ProductDocuments/DataSheets/PIC32MX5XX6XX7XX_Family)Datasheet_DS60001156K.pdf - There is also a specific PIC32MX6xx/7xx series datasheet which is much the same information as the above two datasheets:
- https://www.microchip.com/en-us/products/microcontrollers-and-microprocessors/32-bit-mcus/pic32-32-bit-mcus/pic32mx - Main page

#### MRF24J40MA
The radio is a MRF24J40 but it is in the MA package so you will need the main datasheet for the information about the CPU and the MA datasheet for the board pin information:
MRF24J40: https://ww1.microchip.com/downloads/en/DeviceDoc/39776C.pdf
MRF24J40MA: https://ww1.microchip.com/downloads/en/DeviceDoc/MRF24J40MA-Data-Sheet-70000329C.pdf
Main page: https://www.microchip.com/mrf24j40

### High resolution photos
High resolution photos of both sides of the hub:
<p align="center">Front<br/>
<img src="/assets/Front.jpg" height="350"/><br/>
Back<br/>
<img src="/assets/Back.jpg" height="350"><br/>
CPU<br/>
<img src="/assets/CPU.jpg" height="350">
</p>

### Hub pins
The pads on both sides of the board connect to the other side, so the below photos show the pin connections on both sides of the boards.
This may not be 100% correct so please check this before connecting anything.

The serial port speed is `57600 8/N/1` with flow control disabled.

<p align="center">Front with pins<br/> 
<img src="/assets/Front-WithPins.jpg" height="350"><br/>
Back  with pins<br/>
<img src="/assets/Back-WithPins1.jpg" height="350">
</p>

| Pin Number | CPU Pin | Description |
| -- | -- | -- |
| 1 | 17 | PGC2 used for ICSP debugger |
| 2 | 7 | /MCLR via 100R |
| 3 | 18 | PGD2 used for ICSP debugger |
| 4 | 31 | U2RX - UART RX for the serial console |
| 5 | 32 | U2TX - UART TX for the serial console |
| 6 | 10, 26, 38 | Vdd - 3.3v power supply |
| 7 | 9,25,41 | Vss - Ground for UART and ICSP |
| 8 | - | Vin 5Vdc barrel plug |
| 9 | - | USB connector (unpopulated) USB+? |
| 10 | 36 | USB D- (unpopulated L3) |
| 11 | 37 | USB D+ (unpopulated L3) |
| 12 | 61 | PMD0/RE0 Connected to Reset Button |
| 13 | 9,25,41 | Vss - Ground for UART and ICSP |
| 14 | - | NC |

Side connector pin mapping:

| Pin | CPU Pin | Description |
| -- | -- | -- |
| 6 | 10, 26, 38 | Vdd - 3.3v Positive supply |
| - | NC |
| 7 | 9,25,41 | Vss - Ground |
| 1 | 17 | PGC2 used for ICSP debugger |
| 3 | 18 | PGD2 used for ICSP debugger |
| 2 | 7 | /MCLR via 100R |
| 4 | 31 | U2RX - UART RX for the Console output |
| 5 | 32 | U2TX - UART TX for the Console output |

# Console Boot Message:
The hub doesn't output too much during boot until you enable debug messages:
Standard boot message:
```
    SureFlap Hub 12:45:09 Jan 17 2020
    Build Number 43
    ---------------------------------
            Serial: 48 30 3x 30 2d 30 3x 3x 3x 3x 3x 3x 00
            As text: H0x0-0xxxxxx

    Stack Top: 0xa001ff88

    MAC address = xx:xx:xx:xx:xx:xx:xx:xx
    Read channel f from EEPROM
    Warning trying to change channel to f
    Set PANID to 3421
            [------------- Paired Devices ------------]
            [-----------------------------------------]
```
The above MAC address is the Zigbee Wireless MAC address not the ethernet MAC address

## Debug Menu
The Debug menu can be enabled if you send an upper case "A"
```
    A - Enable debug console
    c/r - After debug messages are enabled hitting enter gives the below debug menu.

    SureFlap Hub Debug Menu
    -----------------------
    e - Dump list of application errors
    h - dump entire hub register entry table
    p - dump pairing table
    l - Toggle Ethernet RJ45 LEDs
    s - Dump set_reg_queue
    t - Spam RF requests to test buffering
    z - disconnect and zero connection table

    Please select
    Other commands:
    miwi_channel_noise_floor addr=23

    RF Channel Register = 24
    1 - Set Channel to 15 (0f)
    2 - Set Channel to 20 (14)
    3 - Set Channel to 26 (1a)
    r - Doesn't give an error, I think it does a ping over MQTT.
    v - Unknown
```

- `h` is the same output as sending a `TS 1000 3 0 205` message over MQTT.
- `z` is the same as pressing the button underneath to start paring.
- `p` Pairing table you get, this corresponds to the 8 status register bytes:

```
    [00]: valid=0 online=0 type=0 last_heard=0 >>00:00:00:00:00:00:00:00<<   -]
    [01]: valid=0 online=0 type=0 last_heard=0 >>00:00:00:00:00:00:00:00<<   -]
    [02]: valid=0 online=0 type=0 last_heard=0 >>00:00:00:00:00:00:00:00<<   -]
    [03]: valid=0 online=0 type=0 last_heard=0 >>00:00:00:00:00:00:00:00<<   -]
    [04]: valid=0 online=0 type=0 last_heard=0 >>00:00:00:00:00:00:00:00<<   -]
    [05]: valid=0 online=0 type=0 last_heard=0 >>00:00:00:00:00:00:00:00<<   -]
    [06]: valid=0 online=0 type=0 last_heard=0 >>00:00:00:00:00:00:00:00<<   -]
    [07]: valid=0 online=0 type=0 last_heard=0 >>00:00:00:00:00:00:00:00<<   -]
    [08]: valid=0 online=0 type=0 last_heard=0 >>00:00:00:00:00:00:00:00<<   -]
    [09]: valid=0 online=0 type=0 last_heard=0 >>00:00:00:00:00:00:00:00<<   -]
```

If you enable debug when the hub first boots you get a lot more information:

```
    | Talking to: hub.api.surehub.io | Socket Obtained | Socket Opened | Socket Secured |
    --- Processing Response: 17 Bytes Total
    1 bytes remain, after 2 reads.Found status = 200
    0 bytes remain, after 3 reads.  | Tstate 2 |
    --- Processing Done

    --- Processing Response: 275 Bytes Total
    0 bytes remain, after 35 reads..Content Length = 3573           Start receiving non-chunked data, length=3573
            | Tstate 4 |
    --- Processing Done

    --- Processing Response: 3573 Bytes Total
    0 bytes remain, after 447 reads....     | Tstate 5 |    TCP_RESPONSE_COMPLETE
    --- Processing Done

    -------- Credentials Decoded --------
            Host:           **8th Field in Creds file**
            Client ID:      **3rd Field in Creds file**
            Base Topic:     **7th Field in Creds file**
            Version:        **1st Field in Creds file**
            ID:             xxxxxx (Serial Number which is also 2nd field ) and ClientID as a single string
            Username:       **4th Field in Creds file, should be empty**
            Password:       **5th Field in Creds file, also empty**
            Network Type:   1 **6th Field in Creds file**
            Certificate:    **9th Field in Creds file**
            
            Length:         xxx
            Cert Hash:      0x xxxxxxxx
            Key Hash:       0x xxxxxxxx

    -------- End Credentials --------

    Connected
    LED new mode a 5 Closing Socket
    Web state=3 Calling connect... Connection made!
    Connection sequence done.
    RF reset
    Web state=5 Subscribing to Hub Messages: Success!
    LED new mode 5 5 Web state=9 TCP Bytes Put: 2852, Seconds: 60
    Set LED to 0
    LED new mode 0 5 Unknown command 71
```

# Pet Door
Product ID = 3
FCC Link: https://fccid.io/XO9-IMPD00003

The Pet Door is obvious larger designed to take either a large cat, or a small dog and my 12.5kg dog which is a Lowchen happily uses the Pet Door.
A major disadvantage of all the Connect devices is they all require batteries and only the Pet Door supports re-chargeable batteries from my understanding. With semi-typical usage the battery should last about 3-4 months. Sometimes more, but if the batteries are lasting less than 3 months something is wrong and you should find out why.

Pet Door advantages:
- Supports a larger animals, but even a largeish cat should be fine with the cat flap.
- Uses 'C' batteries and also supports re-chargables if you select Custom Mode 2.
- You can select the Custom Mode via the buttons on the pet door, or send them via the cloud which setting custom modes can't be done with other devices and needs to be sent from the cloud by logging a case with SureFlap and hoping they will do it for you.
- Can unlock both directions with Non Selective Entry mode so door mechanism doesn't need to unlock for each entry improving battery life so you should get 6+ months life with Energizer Max Alkalines. I do this for home as I don't have random cats who come inside due to our dog. 
- Buttons on the front and LCD on top of the pet door to visually check the situation, lock and unlock the door without using the cloud service.

Pet Door disadvantages:
- Doesn't have the "dual scan" capabilities of the dual scan cat door so can't have selective exit depending on the animal. It would be nice to do this but the tunnel would need to be quite large on the interior side to be able to detect and scan the animal
- A completely different message format to all the other devices... Why SurePet.. why did you do this? Or my guess was the Pet Door the first device they designed and whoever was the developer thought it was a good idea to follow the "hub" 'registers' approach. It's dumb and annoying to developer for.
- Only supports one curfew
- Curfew is another state alongside the locked / keepin state. Whereas on the Cat Flap curfews are independent.

## Pet door message characteristics
Specifics of the Hub message ontop of [Common message characteristics](#common-message-characteristics) and there are only two message types from the Pet Door
- `8` The 8 messages are generated by the Pet Door when an animal comes in or leaves through the door.
- `132` Status register update all values are in big endian byte order, starts at `0` and goes to `630`.

## Pet door example messages
| Type | Example | Description |
| - | - | - |
| CMD - Lock Door | `6283d4fb 1000 2 36 1 03` | Lock the door in both directions |
| CMD - Query Registers | `6283d4fb 1000 3 0 630` | Send command to query the Pet Door Registers from offset `0` to offset `630` |
| Status - Pet 0 Came Inside | `6283d4fb 0bc0 8 00 07 2f 81 00 01 6e 44 02 22 6e 01 e5` | Message 8 with first `00` being pet number at 7:47 local time `07 2f` came inside but pet door thought they were already inside `81` and the rest I don't know what it means |
| Status - Pet 0 Came Inside | `6283d4fb 0bd0 132 130 525 3 07 2f 81` | Also Pet 0 register `525` with `3` byte at `07 2f` 7:47 came inside `81` when already thought inside  |

## Hub 132 Register messages
| Offset | Example | Description |
| - | - | - |
| 3 | `132` | `132` register status update |
| 4 | `1` | Single byte decimal counter that goes from `0` to `254` before repeating, not zero padded |
| 5 | `0` to `630` | Starting register decimal offset, this is the starting offset of the register not zero padded |
| 6 | `1` to `8` | Payload length, from the above register, the proceeding bytes are the reigster(s) changing not zero padded, maximum length of 8 for each message |
| 7+ | `xx` | Payload data in Hex byte, this is zero padded for the hex value |

With the above understanding of the different message formats, the registers sent in the 132 messages are documented below from my understanding.

## 132 Door Registers

| Register | Description |
| -- | -- |
| 33 | Battery ADC Value. Battery full at 0xbd and door dies at around 0x61/0x5f. ADC Start at 2.1075 not sure if this is consistent or just my door, and ADC step = 0.0225 |
| 34-35 | Door Time in local HH:MM in hex of the local time, so yes at DST Surepet updates every pet door with the new time |
| 36 | Lock State 0-4 The lock state of the door, UNLOCKED = 0, KEEPIN = 1, KEEPOUT = 2, LOCKED = 3, CURFEW = 4 |
| 40 | Inbound lock state for keep pets out. This also changes between Normal - 2 and Locked Keep Out - 3, which is separate to what the 36 register offst is doing |
| 59 | Provisioned tag count. I think this is a total of active tags provisioned |
| 60 | Next free tag slot, when provisioning a tag from the cloud this selects the slot to use |
| 61-63 | Custom Mode in big endian byte order |
| 91-309 | Provisioned tags - 7 bytes for each provisioned tag with the first byte being the tag type |
| 519 | Curfew mode lock / unlock time |
| 525-618 | Pet movement state in or out |
| 621 | Unknown pet went outside |

## Pet Door Custom Modes
Custom Modes are a bitwise operator where each custom mode can be toggled on or off independently and allow additional features to be set on the Pet Door.

As per https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/enums.py

And:

| Custom Mode | Name | Description | YouTube Link |
| -- | -- | -- | -- |
| 1 | Nonselective | Unlocks the door inbound so any animal can come in | https://www.youtube.com/watch?v=riq5eRhDRGs |
| 2 | Rechargeables | Work with lower voltage from 1.2v Rechargeables vs 1.5v Alkaline | https://www.youtube.com/watch?v=QYtpQPlTFQM |
| 3 | ThreeSeconds | Timid Pets - 3 Seconds delay before closing door | https://www.youtube.com/watch?v=H3Xyqys5M88 |
| 4 | TenSeconds | Slower Locking - 10 Seconds delay before closing door | https://www.youtube.com/watch?v=7iRo38jNwLc |
| 5 | Intruder | Intruder Mode - Lock outside locks when non-provisioned animal detected by sensor to prevent door being pulled open | https://www.youtube.com/watch?v=PWz8JbAxCMo |
| 6 | OppositeCurfew | Opposite Curfew mode - Lock KeepOut rather than KeepIn | https://www.youtube.com/watch?v=wILX2J4dwKU |
| 7 | LockedCurfew | Fully Locking Curfew Mode - Locks both in and out locks when in curfew mode | https://www.youtube.com/watch?v=0L6lmuvt8U0 |
| 8 | MetalMode1 :metal: | Metal :metal: Interference - This mode will help with severe metal interference in an installation | https://www.youtube.com/watch?v=KIKXKdgp578 |
| 9 | MetalMode2 :metal::metal: | Metal :metal: Interference - This mode will help with severe metal interference in an installation | https://www.youtube.com/watch?v=yqlIVgcG-ns |
| 10 | ExtendedRange | Extended Mode - Extend frequency of scanning the tags | https://www.youtube.com/watch?v=jKmGEf2jxJE |
| 11 | ExtendedIntruder | Extended Intruder Mode - Extended Intruder Mode - Registers presence of intruder animal trying to enter the house and closes outside lock to prevent door being pulled open for longer period | https://www.youtube.com/watch?v=9Twi5wgHWsU |
| 13 | DoubleChip1 | Double Chip Operating Mode 1 - Allow animal with two tags interfering with each other to enter | https://www.youtube.com/watch?v=xpnFTLppBlM |
| 14 | DoubleChip2 | Double Chip Operating Mode 2 - Allow animal with two tags interfering with each other to enter | https://www.youtube.com/watch?v=CY7NdNzVB1M |
| 15 | DoubleChip3 | Double Chip Operating Mode 3 - Allow animal with two tags interfering with each other to enter | https://www.youtube.com/watch?v=vfhzancw_5A |
| 16 | ProximityTest | Custom Mode 16 - Proximity Sensor Test - Test the proximity function of the door | https://www.youtube.com/watch?v=tCIHRrZyzlI |

```
class PetDoorCustomMode(SureFlag):
    """ Custom Modes on the Pet Door """
    Disabled = 0              # All custom modes disabled
    Nonselective = 0x1        # Custom Mode 1 - Non-selective Entry - Unlocks the door inbound so any animal can come in
    Rechargeables = 0x2       # Custom Mode 2 - Rechargeable Batteries so work with lower voltage from 1.2v Rechargeables vs 1.5v Alkaline
    ThreeSeconds = 0x4        # Custom Mode 3 - Timid Pets - 3 Seconds delay before closing door
    TenSeconds = 0x8          # Custom Mode 4 - Slower Locking - 10 Seconds delay before closing door
    Intruder = 0x10           # Custom Mode 5 - Intruder Mode - Lock outside locks when non-provisioned animal detected by sensor to prevent door being pulled open
    OppositeCurfew = 0x20     # Custom Mode 6 - Opposite Curfew mode - Lock KeepOut rather than KeepIn
    LockedCurfew = 0x40       # Custom Mode 7 - Fully Locking Curfew Mode - Locks both in and out locks when in curfew mode
    MetalMode1 = 0x80         # Custom Mode 8 - Metal Interference - This mode will help with severe metal interference in an installation
    MetalMode2 = 0x100        # Custom Mode 9 - Metal Interference - This mode will help with severe metal interference in an installation
    ExtendedRange = 0x200     # Custom Mode 10 - Extended Mode - Extend frequency of scanning the tags
    ExtendedIntruder = 0x400  # Custom Mode 11 - Extended Intruder Mode - Extended Intruder Mode - Registers presence of intruder animal trying to enter the house and closes outside lock to prevent door being pulled open for longer period
    BIT12 = 0x800             # Bit12 - ?
    DoubleChip1 = 0x1000      # Custom Mode 13 - Double Chip Operating Mode 1 - Allow animal with two tags interfering with each other to enter
    DoubleChip2 = 0x2000      # Custom Mode 14 - Double Chip Operating Mode 2 - Allow animal with two tags interfering with each other to enter
    DoubleChip3 = 0x4000      # Custom Mode 15 - Double Chip Operating Mode 3 - Allow animal with two tags interfering with each other to enter
    ProximityTest = 0x8000    # Custom Mode 16 - Proximity Sensor Test - Test the proximity function of the door
```

## Pet Door Tags

There are two tag types, `01` for FDX-B tags and `04` for HDX tags, I have not yet figured out the calculation for the HDX tags.

The door tags is a big endian value for the FDX-B tags. The remaining 6 bytes is the tag value split into a 10 bits for the Country Code and 38 bits for the National Code

Example:

| Hex Tag | Decimal |
| -- | -- |
| `01 52 54 61 12 eb 9f` | `999.100001000010` |


`999.100001000010` to binary zero padding to 10 bits and 38 bits respectively : `1111100111` and `01011101001000100001100010101001001010`

`111110011101011101001000100001100010101001001010` reversed bits `010100100101010001100001000100101110101110011111`

`010100100101010001100001000100101110101110011111` as integer `90522359360415`

`90522359360415` as hex `52546112EB9F`

Then prefix `01` for the tag type of FDX-B and you get `01 52 54 61 12 eb 9f`

# Non Pet Door Devices Common Message

All the non Pet Door devices have a consistent messaging format with the message types `126` and `127`.

| Type | Feeder | Cat Flap | Poseidon | Description |
| -- | -- | -- | -- | -- |
| `00` | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | Acknowledgement Command Message to Status Message |
| `01` | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | Get Command Message to retrieve message state |
| `07` | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | Set Time Command Message to set device time |
| `09` | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | Device settings parameters |
| `0b` | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | 0b Boot Status Message |
| `0c` | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | Battery Status Message |
| `0d` | :heavy_check_mark: | :heavy_check_mark: |  | Zero Feeder Scales and Cat Flap Curfew Status messages |
| `10` | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | 10 Boot Status Message |
| `11` | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | Tag Provision Command Message to Status Message |
| `12` | | :heavy_check_mark: |  | Set Cat Flap Curfew Command message |
| `13` | | :heavy_check_mark: |  | Cat Movement through cat flap status message |
| `17` | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | 17 Unknown message |
| `18` | :heavy_check_mark: |  |  | Feeding status message |
| `1b` | | | :heavy_check_mark: | Drinking status message |

## 126 Messages

These are the status message where multiple messages can all be sent in a single payload from the device. Directly after the `126` the message length in hex, then the message type, and the payload. At the end of the payload an additional message can be sent. Again the length then message type until the end of the payload. The first byte after the timestamp is the same hub counter message as already talked about `0xx0` above. That is consistent for all messages.

This means a message can look like:

```
tttttttt 0010 126 0d 09 00 01 00 9f cc 42 59 0c 02 00 00 00
   |      |    |  |  |     --+-- -----+-----  | +----------
   |      |    |  |  |       |        |       | +- Bowl Count = 2
   |      |    |  |  |       |        |       +--- SubType = C
   |      |    |  |  |       |        +----------- Timestamp
   |      |    |  |  |       +-------------------- Counter
   |      |    |  |  +---------------------------- Message Type
   |      |    |  +------------------------------- Length
   |      |    +---------------------------------- Message Type 126
   |      +--------------------------------------- Hub Counter Status Message
   +---------------------------------------------- Hub UTC Hex Timestamp
```

The above secnario the message is a single message length is `0d` bytes long. The message type is a `09` and the payload is `00 01 00 9f cc 42 59 0c 02 00 00 00`.

```
tttttttt 0xx0 126 0d 09 00 01 00 9f cc 42 59 0c 02 00 00 00 0d 09 00 02 00 9f cc 42 59 0a 88 13 00 00
```
Similar to the above, the first message is the same message above, the second message is also `0d` bytes long and a message type `09` with the payload `00 02 00 9f cc 42 59 0a 88 13 00 00`.

## 127 Messages

The 127 messages are command messages either sent from the cloud service to change something, such as locking the cat flap or provisioning a new tag to the device. Because they are command messages they will always be `1000`, but they are a single message so do not include the length so you need to *just know* how long the message you want to send is and just start with the message type immedateily after `127` ie

```
tttttttt 1000 127 09 00 01 00 9f cc 42 59 0c 01 00 00 00
   |      |    |  |     --+-- -----+-----  | +----------
   |      |    |  |       |        |       | +- Bowl Count = 1
   |      |    |  |       |        |       +--- SubType = C
   |      |    |  |       |        +----------- Timestamp
   |      |    |  |       +-------------------- Counter
   |      |    |  +---------------------------- Message Type
   |      |    +------------------------------- Message Type 126
   |      +------------------------------------ Command Message
   +------------------------------------------- Hub UTC Hex Timestamp
```

The above is setting the Bowl Count to 1.

## Message Counter

Directly after the message type is a `00`, then a two byte little endian byte counter which counts from 0 to 65534 before cycling back to 0. This is so the cloud knows the current state of the devices and if messages are lost as if the hub is offline or disconnected from the internet none of the messsages are retained. Typical messages:
There is also a send a retrive counter, which are generated and increment independently. As each end have their own *sent counter*

```
tttttttt 0xx0 126 09 01 00 01 00 9e cc 42 59 11 11 - Device Counter = 1
tttttttt 1000 127 01 00 02 f2 9e cc 42 59 11 11 - Cloud Counter = F202 = 61954
```

## Message Timestamp

After the counter is a 4 byte timestamp of the event. This is used by the device to set the time of it, and report back the timestamp of the device if there is any skew. The time is stored in UTC, but instead of using the UTC Timestamp they have their own custom rolled timestamp using bitwise operators as shown below which bits are converted into numbers. This is calculated in function [devicetimestamp in message.py](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/message.py)

```
def devicetimestamp(hexts, time_zone, tz_format):
    """ Convert device hex timestamp into something human readable """
    timestamp_int = int.from_bytes(hexts, byteorder='little')
    year = int(f'20{timestamp_int >> 26:02}')  # Year - 6 Bits, prepending 20 for 4 digit year
    month = timestamp_int >> 22 & 0xf          # Month - 4 Bits
    day = timestamp_int >> 17 & 0x1f           # Day - 5 Bits
    hour = timestamp_int >> 12 & 0x1f          # Hour - 5 Bits
    minute = timestamp_int >> 6 & 0x3f         # Minute - 6 Bits
    second = timestamp_int & 0x3f              # Second - 6 Bits
```

**Example Conversion**

```
Using the example timestamp `9e cc 42 59` 
9e cc 42 59 -> Little Endian -> 5942CC9E -> Integer = 1497549982
1497549982 -> Binary 01011001010000101100110010011110
First 6 Bits 010110 -> 22 Year, so prepent 20 to 2022.
Next 4 Bits 0101 -> 5 Month
Next 5 Bits 00001 -> 1 Day
Next 5 Bits 01100 -> 12 Hour
Next 6 Bits 110010 -> 50 Minute
Next 6 Bits 011110 -> 30 Second
UTC Time = 2022-05-01 12:50:30
```

And since this is UTC time, it needs to be converted into local time.

## Message 00 - Acknowledgement
Message Length = `07`

Every status message needs to be acknowledged back using a `127` and a message type of `00` once it has been processed by the other end. This is sent with the same send counter, but the timestamp of the sending host, and include the message type and two bytes of `00` ie `01 00 00` being acknowledged.

```
Hub to Cloud -> tttttttt 0xx0 126 07 01 00 02 11 9e cc 42 59 11 22 33 44 55
Cloud to Hub -> tttttttt 1000 127 00 00 02 11 9f cc 42 59 01 00 00
```

## Message 01 - Get Values
Message Length = N/A as Command only, and varaible length.

To retrieve the current status of a message type a `01` command message is sent to the device.

```
Cloud to Hub -> tttttttt 1000 127 01 00 01 00 9f cc 42 59 09 00 ff  # Boot message 09
Cloud to Hub -> tttttttt 1000 127 01 00 02 00 9f cc 42 59 0b 00     # Unknown 0b
Cloud to Hub -> tttttttt 1000 127 01 00 03 00 9f cc 42 59 0c 00     # Battery state
Cloud to Hub -> tttttttt 1000 127 01 00 04 00 9f cc 42 59 0d 00     # Lock state
Cloud to Hub -> tttttttt 1000 127 01 00 05 00 9f cc 42 59 10 00     # Boot message 10
Cloud to Hub -> tttttttt 1000 127 01 00 06 00 9f cc 42 59 11 00 ff  # Tag provisioned
Cloud to Hub -> tttttttt 1000 127 01 00 07 00 9f cc 42 59 12 00     # Curfew state
Cloud to Hub -> tttttttt 1000 127 01 00 08 00 9f cc 42 59 17 00 00  # Boot message 17
Cloud to Hub -> tttttttt 1000 127 01 00 09 00 9f cc 42 59 1b 00     # Water state
```

## Message 07 - Set Time

Command message to set the time of the device in UTC. As all the time is in UTC the device timestamp value needs to be set to UTC time.

```
Cloud to Hub -> tttttttt 1000 127 01 00 01 00 9f cc 42 59 00 00 00 00 07  # Sets the device time to 2022-05-01 12:50:30
```

## Message 09 - Device settings messages
Messages Length = `0d`

This is used for setting device settings. Such as custom modes and bowl counts for the feeder. The sub-type messages vary depending on the device and have a byte after the timestamp for the sub-type, then little endian 4 byte word for the value of the sub-type.

Status 09 Example:
```
0d 09 00 01 00 9f cc 42 59 0c 02 00 00 00
|  |     --+-- -----+-----  | -----+-----
|  |                        |    Bowl Count = 2
|  |                        +--- SubType = C
|  +---------------------------- Message Type
+------------------------------- Length
```

## Message 0b - Boot Device Information

The boot device message length is `43` and the figured fields defined [message](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/message.py) 

```
    elif value[0] == 0x0b:  # Status - Boot Device Information
        frame_response.Operation = "DeviceInfo"
        frame_response.Hardware = b2is(value[8:12])
        frame_response.Firmware = b2is(value[12:16])
        frame_response.EntityType = EntityType(value[64]).name
        frame_response.Val1 = b2is(value[16:20])        # **TODO Some value
        frame_response.HexTS = value[20:28].hex()
        frame_response.SerialHex = value[36:45].hex()  # **TODO Serial Number calc somehow
```

## Message 0c - Battery Information
Message length = `18`

The battery information in the [message](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/message.py) 

The 4 byte word after the timestamp is the battery value with 3 decimal places of accuracy as reported in the app.

```
126 18 0c 00 01 00 9f cc 42 59 b8 14 00 00 dd 0c 00 00 25 01 00 00 00 00 00 00

b8 14 00 00 -> Hex 14B8 -> Decimal 5304 / 1000 -> Battery voltage 5.304
```

Code:
```
    elif value[0] == 0x0c:  # Status - Battery state for four bytes
        frame_response.Operation = 'Battery'
        battery = str(round(int(b2is(value[8:12])) / 1000, 4))
        frame_response.Battery = battery
        frame_response.Value2 = str(int(b2is(value[12:16])))  # **TODO Not sure what this value is.
        frame_response.Value3 = str(int(b2is(value[16:20])))  # **TODO Or this one
        frame_response.BatteryTime = devicetimestamp(value[20:24], time_zone, '')  # **TODO Last time the time was set?
```        

## Message 10 - Some boot message
Message length = `18`

The Boot in the [message](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/message.py) 
No idea what is in the message.

## Message 11 - Tag Provisioning
Message Length = `12`

This is used to and and remove tags to the device. The Tag Calculation is completely different from the Pet Door as it is little endian. This is also used to set lock state on the Cat Flap on Tag `00`.

Example Message:
```
126 12 11 00 01 00 9f cc 42 59 4a 2a 86 48 d7 f9 01 02 01 00
       |     --+-- -----+----- --------+-------- |  |  |  +- Always 0
       |                               |         |  |  +---- Tag Offset starting with 1
       |                               |         |  +------- Tag State, 02 = Normal
       |                               |         +---------- Tag Type `01` for FDX-B
       |                               +-------------------- Tag Value
       +---------------------------------------------------- Message Type 11
```

There are 2 types of tags:
- Tag `01` is a FDX-B tag with the numbering `xxx.xxxxxxxxxx` decimal value 3 + 10 digits
- Tag `03` is a HDX tag which is presented as 5 bytes of hex `hhhhhhhhhh`

FDX-B Tag Conversion:
```
4a 2a 86 48 d7 f9 -> Little endian hex F9D748862A4A -> Int 274703030037066
274703030037066 -> Binary 111110011101011101001000100001100010101001001010
First 10 Bytes are Country Code = 11111001110 = 999
38 Bytes are National Code = 1011101001000100001100010101001001010 = 100001000010
FDX-B Tag = 999.100001000010
```

HDX Tag:
```
11 22 33 44 55 00 -> Direct translation of the hex to the tag and the last byte is always 00
```

Tag State is either `02` for active/normal, `06` for disabled, or `03` if it is a CatFlap and the Tag is set to Keep In

Tag Offset starts at 1, and supports up to 32 aka `1f` tags on the device. Tag Offset `0` is used on the Cat Flap to set door locking state.

# Feeder
Product ID = 4

## Message 09 - Feeder Messages

```
Status            -> 09 00 01 00 9f cc 42 59 05 00 00 00 00  # Training Mode 
Status or Command -> 09 00 02 00 9f cc 42 59 0a 88 13 00 00  # Get/Set Left Weight if two bowls or if 1 bowl target weight to 50.00 grams
Status or Command -> 09 00 03 00 9f cc 42 59 0b 70 17 00 00  # Get/Set Right Weight if two bowls to 60.00 grams
Status or Command -> 09 00 04 00 9f cc 42 59 0c 02 00 00 00  # Get/Set Bowl Count either 1 or 2
Status or Command -> 09 00 05 00 9f cc 42 59 0d a0 0f 00 00  # Get/Set Lid Close Delay 4 Seconds - 0(fast) or 4000(normal) or 20000(slow) decimal values
Status or Command -> 09 00 06 00 9f cc 42 59 14 80 00 00 00  # Get/Set Custom Mode, refer to enum for values
Status            -> 09 00 07 00 9f cc 42 59 17 f9 10 00 00  # Zero left scales absolute weight result = 4345
Status            -> 09 00 08 00 9f cc 42 59 18 66 0e 00 00  # Zero right scales absolute weight result = 3686
```

### Target Weights

Target weights are `0a` and `0b` sub-types. If bowls = 1 then only `0a` is set. If bowls = 2 then `0a` is the left target weight and `0b` is the right target weight.
Like all weights it is grams and 2 decimal places of accuracy. So to set 50 grams as the target weight:
```
50 * 100 = 5000 = Hex 1388 = Little endian word 88 13 00 00
```

### Lid Close Delay

The values that can be set are specified is in [enums](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/enums.py) 

```
class FeederCloseDelay(SureEnum):
    """ Feeder Close Delay Speed """
    FAST         = 0               # Fast delay   - 0 Seconds
    NORMAL       = 4000            # Normal delay - 4 Seconds
    SLOW         = 20000           # Slow delay   - 20 Seconds
```

### Cat Flap Custom Modes

The values that can be set are specified is in [enums](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/enums.py) 

Non-Connect Feeder Custom Modes... May be aligned to the Connect version with a bitwise operator.?

 -- Order of flashing light when selecting the custom mode.
- 0 - Static Red
- 1 - Static Green
- 2 - Static Orange
- 3 - Flashing Red 
- 4 - Flashing Green
- 5 - Flashing Orange

| Custom Mode | Name | Description | YouTube Link |
| -- | -- | -- | -- |
| Mode I | 0-Static Red | Extended Mode | https://www.youtube.com/watch?v=pYyNOvLfu6A |
| Mode I | 4-Flashing Green | German Funk antenna custom mode | https://www.youtube.com/watch?v=PIe0OwpC8QY |
| Mode I | 5-Flash Orange | Metal Mode :metal: | https://www.youtube.com/watch?v=hQ7aQ1S61w8 |
| Mode II | 0-Static Red | Non Selective Feeding Mode | https://www.youtube.com/watch?v=Ifu7SQ0hWz4 |
| Mode II | 3-Flash Red | Multi-Scan Custom Mode | https://www.youtube.com/watch?v=k2YZewDOG6A |
| Mode II | 4-Flashing Green | Intruder Training Mode | https://www.youtube.com/watch?v=a_IgU4q5E1A or https://www.youtube.com/watch?v=BewTCQ0lbso |

```
class FeederCustomMode(SureFlag):
    """ Custom Modes on the Feeder """
    Disabled = 0         # All custom modes disabled
    NonSelective = 0x40  # Bit7 - Non-selective Entry - Allow any animal who breaks the infrared link to open feeder
    GeniusCat = 0x80     # Bit8 - Genius Cat Mode - Disable open/close button as Genius Cat has figured out how to open the feeder by pressing button.
    Intruder = 0x100     # Bit9 - Intruder Mode - Close lid when another non-provisioned tag turns up
```

## Message 0d - Feeder Zero Scales Command

This command is sent to the Feeder when you Zero Scales command from the app.

- FeederZeroScales [FeederState enums](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/enums.py) 01 - Zero Left, 02 - Zero Right, 03 - Zero Both

```
127 0d 00 26 00 9f cc 42 59 00 19 00 00 00 03 00 00 00 00 01 03
                                                             +- Zero Both Feeders
```

The response is a number of different acknowledgements including sending the scales state.

```
126 0d 09 00 15 0e 9f cc 42 59 12 f4 01 00 00 0b 00 00 23 00 9f cc 42 59 09 00 00 0b 00 00 24 00 9f cc 42 59 0d 00 00 0d 09 00 16 0e 9f cc 42 59 17 3c 0c 00 00 0d 09 00 17 0e 9f cc 42 59 18 ad 0d 00 00

Mult-message broken into:

0d 09 00 15 0e 9f cc 42 59 12 f4 01 00 00  - 09 Sub-Value 12, always 500
0b 00 00 23 00 9f cc 42 59 09 00 00        - Ack of Message 09
0b 00 00 24 00 9f cc 42 59 0d 00 00        - Act of Message 0d
0d 09 00 16 0e 9f cc 42 59 17 3c 0c 00 00  - 09 Sub-Value 17 - Left absolute weight 3132 = 31.32g
0d 09 00 17 0e 9f cc 42 59 18 ad 0d 00 00  - 09 Sub-Value 18 - Right absolute weight 3501 = 35.01g

Feeder Status

126 29 18 00 18 0e 9f cc 42 59 01 02 03 04 05 06 07 06 00 00 02 00 00 00 00 05 00 00 00 00 00 00 00 f8 ff ff ff f9 00 22 01 00 00
       +     --+-- -----+----- +------------------- |                       +----------             +----------
       + Feeder Message        + Tag 1234567        + Action 6 Zero Scales  + Left Weight = 0.05g   + Right Weight = -0.08g
```

## Message 18 - Feeder Messages
Message length = `29`

The format is: 
- 7 Bytes Animal Tag or all 0's if manually opened
- 1 Byte Bowl Action [FeederState enums](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/enums.py) Animal Open 0, Animal Closed = 1...
- 2 Bytes Time open in seconds
- 4 Bytes Left Bowl Start weight to 2 decimal places, or all bowls if single bowl
- 4 Bytes Left Bowl End weight to 2 decimal places or all zeros if opening
- 4 Bytes Right Bowl Start weight to 2 decimal places
- 4 Bytes Right Bowl End weight to 2 decimal places or all zeros if opening

It's a signed integer so you can get negative values if the zeroing of the scales had something in the scales so when you remove it a negative value is returned.

Example Message
```
126 29 18 00 01 00 9f cc 42 59 4a 2a 86 48 d7 f9 01 01 44 01 02 5c 05 00 00 6d 03 00 00 8e 0d 00 00 a1 06 00 00 ea 00 24 01 00 00
       +     --+-- -----+----- ---------+---------- |  --+-- |  -----+----- -----+----- -----+----- -----+----- --+-- +----------
                                        |           |    |   |       |           |           |           |        |   + Unsure
                                        |           |    |   |       |           |           |           |        +---- Counter
                                        |           |    |   |       |           |           |           +------------- Closing Right Weight To
                                        |           |    |   |       |           |           +------------------------- Opening Right Weight From
                                        |           |    |   |       |           +------------------------------------- Closing Left Weight To, or Total if 1 Bowl
                                        |           |    |   |       +------------------------------------------------- Opening Left Weight From, or Total if 1 Bowl
                                        |           |    |   +--------------------------------------------------------- Bowl Count(?) = 2
                                        |           |    +------------------------------------------------------------- Open Seconds, or 0 if Opening not closing
                                        |           +------------------------------------------------------------------ Feeder Bowl Action, as per above
                                        +------------------------------------------------------------------------------ Tag = 999.100001000010
```

Other observations is if the Custom Mode is set to Intruder then the opening tag is the correct animal able to open the feeder, and the closing message is the intruder.

# Cat Flap
Product ID = 6

Cat Flap advantages:
- Runs on AA batteries which are typically cheaper, but does not support re-chargables unlike the Pet Door.
- Lock modes are separate from curfew mode, so curfew can be set along with lock in/out which isn't supported on Pet Door as either you have curfews or locked state.
- Support for four curfew times, pet door only supports one.
- Support for dual scan, so scans to let animal out as well as in. Pet door does not have the ability to scan pet on the inside as there is no antenna coil on the interior side.
- Because of dual scan you can set lock state of a particular cat to indoor only when others can exit.

Disadvantages:
- Cannot change curfews without the app curfews
- Custom modes can be changed via the buttons but not documented
- No LCD to see state

## Message 09 - Cat Flap settings messages
Messages Length = `0d`

### Setup Custom Modes

Custom modes can be setup on the device, or sent remotely. To do it locally press and hold *BOTH* the Settings Button on the left *AND* the Add Pet Button for 3 seconds until the indicator light comes on Solid Red. Then select the mode you want based on the LED then to activate the mode press and hold Settings Button for 3 seconds. If you don't do anything in 60 seconds it exits settings mode.

All custom modes are in settings registers on `09`

| Custom Mode | Colour | Name | Description | Youtube |
| -- | -- | -- | -- | -- | 
| `04` | Solid Red | Extended Frequency | To extend the frequency for detuned microchips | https://www.youtube.com/watch?v=2BoMq6g0XMc or https://www.youtube.com/watch?v=BW_TWN5KAt4 or https://www.youtube.com/watch?v=Tc17W_zxowE |
| `05` | Solid Green | Non Selective Exit | Any cat can exit, only tagged cats can enter | https://www.youtube.com/watch?v=G4MFkrmyDto |
| `07` | Solid Orange | Metal :metal: | :metal: Mode Resolve metal inteference issues either 0 or 2 | https://www.youtube.com/watch?v=M4r-q2IVOSo or https://www.youtube.com/watch?v=VpDqvVTKjA0 |
| `08` | Flashing Red | Fast Locking | Locking faster than usual | https://www.youtube.com/watch?v=56xQspY5t5I |
| `01` | Flashing Green | Double Chip Operating | To extend the frequency for detuned microchips either `01` or `02` | https://www.youtube.com/watch?v=P1prjghrsz4 |
| `??` | Flashing Orance | Fail Safe | Unlock if the batteries fail | https://www.youtube.com/watch?v=zP9KO98PiHw or https://www.youtube.com/watch?v=z36MukKUzDQ |
| N/A | Flash Red & Green | Erase All Custom Modes | Clear all custom modes | https://www.youtube.com/watch?v=Wohm9dXP8G0 

Command 09 Settings Messages Example:
```
0d 09 00 01 00 9f cc 42 59 04 00 00 00 00 - Disable Extended Frequency
0d 09 00 01 00 9f cc 42 59 04 01 00 00 00 - Enable Extended Frequency
0d 09 00 01 00 9f cc 42 59 05 00 00 00 00 - Disable Non Selective Exit
0d 09 00 01 00 9f cc 42 59 05 01 00 00 00 - Enable Non Selective Exit
0d 09 00 01 00 9f cc 42 59 07 00 00 00 00 - Disable Metal Mode
0d 09 00 01 00 9f cc 42 59 07 02 00 00 00 - Enable Metal Mode
0d 09 00 01 00 9f cc 42 59 08 00 00 00 00 - Disable Fast Lock
0d 09 00 01 00 9f cc 42 59 08 01 00 00 00 - Enable Fast Lock
0d 09 00 01 00 9f cc 42 59 01 00 00 00 00 - Disable Double Chip
0d 09 00 01 00 9f cc 42 59 01 01 00 00 00 - Disable Double Chip
```

## Message 0d - Cat Flap Curfew Enabled 
Message Length = `1e`

These happen when the cat flap reports back on it's current status and the curfew is enabled. It doesn't send a message when the curfew lock state changes.
- CatFlapCurfewState [FeederState enums](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/enums.py) 

The lock state will either be 03 for on, or 06 for off so that means it is locked at this point in time or not.

```
126 1e 0d 00 a5 09 9f cc 42 59 40 4f fa 75 00 00 00 00 00 00 00 00 00 00 00 00 ff 00 02 00 04 06
       +     --+-- -----+----- -----+-----                                                    +- Lock State
                                    +--------- No idea, could be a counter or something
```

## Message 11 - Set Lock State and Cat Inside Only
Message Length = `12`

- CatFlapLockState [FeederState enums](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/enums.py) 

This is a command message that sets the lock state of the whole cat flap and overrides all pets

Example Message:
```
127 11 00 20 00 9f cc 42 59 00 00 00 00 00 00 07 05 00 02
    |     --+-- -----+----- --------+-------- |  |  |  +- Always 2 for Cat Flap State changes
    |                               |         |  |  +---- Tag Offset 0 - Cat Flap
    |                               |         |  +------- Lock State - 05 = Keepout
    |                               |         +---------- Tag Type always 07
    |                               +-------------------- Tag Value always zero's
    +---------------------------------------------------- Message Type 11
```

The other specific setting along with Tag Provisioning is where the feature is you can set on a per-tag basis if the cat is to be kept inside only. This is set witht he Lock State byte on the tag.

Example Message:
```
127 11 00 01 00 9f cc 42 59 4a 2a 86 48 d7 f9 01 03 01 00
    |     --+-- -----+----- --------+-------- |  |  |  +- Always 0 for Cat Tag State
                                                 |  +---- Tag Offset 1
                                                 +------- Lock State - 03 = Keepin, or 02 for normal.
```

## Message 12 - Set Curfew of Cat Flap
Message length = `33` or N/A as it's a command only.

```
Disable all curfews
1000 127 12 00 01 00 tt tt tt tt 00 00 00 00 00 00 07 00 00 00 42 00 00 00 42 00 06 00 00 42 00 00 00 42 00 06 00 00 42 00 00 00 42 00 06 00 00 42 00 00 00 42 00 06
         |     --+-- -----+-----                         ------------+------------- ------------+------------- ------------+------------- ------------+-------------
         |      CC   Timestamp                                       |                          |                          |                          +------------- Curfew 4
         |                                                           |                          |                          +---------------------------------------- Curfew 3
         |                                                           |                          +------------------------------------------------------------------- Curfew 2
         |                                                           +---------------------------------------------------------------------------------------------- Curfew 1
         +---------------------------------------------------------------------------------------------------------------------------------------------------------- Message 12
```

As you can set 4 curfews this is all sent in one message. Using the same [Message Timestamp](#message-timestamp) calculation and as always in UTC time as all times on the device are in UTC. But the message is sent in todays Date rather than some other artibrary date. And the HH:MM based on the UTC time to open / close and seconds set to 00. The last byte of the curfew is either `03` for enabled or `06` for disabled. If you change curfews on the app and have curfew 1,2,3 and remove curfew 2, then curfew 3 is replaced on curfew 2 and curfew 3 is blank.
The message length always needs to be the same where all curfews are populated even if they are disabled.

## Message 13 - Pet Movement through door
Message length = `1e`

Similar to the Pet Door when a cat sticks their haed in, or goes through the cat flap this message is generated. It also generates status messages for whatever reason.

- CatFlapDirection [FeederState enums](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/enums.py) 

The Came in/out/looked in/out are fairly straightforward in the enum. The Status messages of `01 02` and `02 02` seem to happen very frequently, and no idea what they mean.

Example Cat Message 
```
126 1e 13 00 22 00 9f cc 42 59 00 00 00 00 ba 11 00 00 02 00 4a 2a 86 48 d7 f9 01 05 25 01 00 00
       |     --+-- -----+-----             -----+----- --+-- --------+-------- |  |  +---------- 25 01, 28 01.. Unsure
       |                                        |        |           |         |  +------------- Unknown Counter
       |                                        |        |           |         +---------------- Tag Type
       |                                        |        |           +-------------------------- Tag that was detected
       |                                        |        +-------------------------------------- Pet Movement - Looked out
       |                                        +----------------------------------------------- Unknown Counter
       +---------------------------------------------------------------------------------------- Message 13
```

Example Status Message
```
126 1e 13 00 54 00 9f cc 42 59 00 00 00 00 42 16 01 00 01 02 00 00 00 00 00 00 00 39 28 01 00 00 1e 13 00 55 00 6e 90 a0 59 00 00 00 00 ca 01 00 00 02 02 00 00 00 00 00 00 00 39 28 01 00 00

Multi-message

126 1e 13 00 54 00 9f cc 42 59 00 00 00 00 42 16 01 00 01 02 00 00 00 00 00 00 00 39 28 01 00 00
    1e 13 00 55 00 9f cc 42 59 00 00 00 00 ca 01 00 00 02 02 00 00 00 00 00 00 00 39 28 01 00 00
       |     --+-- -----+-----             -----+----- --+-- --------+-------- |  |  +---------- 25 01, 28 01.. Unsure
       |                                        |        +-------------------------------------- Pet Movement Status, either 01 02 or 02 02
       |                                        +----------------------------------------------- Unknown Counter
       +---------------------------------------------------------------------------------------- Message 13
```

# Poseidon
Product ID = 8

The Poseidon / Felaqua is a fairly dumb device. The 4 feet have scales to detect weight changes and report values in a similar way to the feeder. Really there is only one message type which is the `1b` 

## Message 09 - Poseidon Command

The only command you can send is for the Posedion to start looking for tags.

```
1000 127 09 00 01 00 9f cc 42 59 0f 01 00 00 00  # Look for tags
1000 127 09 00 01 00 9f cc 42 59 0f 00 00 00 00  # Stop looking for tags
```

## Message 1b - Poseidon drinking
Message Length - Variable

The message length is typically `1b` when no tags have been detected, when a tag is detected then the last byte changes from `00` to the number of tags. So thus the message length also changes. Similar that there is a Drinking action, time spent drinking, from and to weights.

For the drinking action refer to the enum
- PoseidonState [FeederState enums](https://github.com/PetHubLocal/pethublocal/blob/main/pethublocal/enums.py) 

Example Messages
```
Water Bowl Status update Action = 00 
126 1b 1b 00 ea 04 9f cc 42 59 00 00 00 01 0c d2 ff ff 8f 6f fe ff cb 00 21 01 00 00 00
       |     --+-- -----+----- |  --+-- |  -----+----- -----+----- |     --+--       +- Number of Tags  
       |                       |    |   |       |           |      |       +----------- Always 21-28 01
       |                       |    |   |       |           |      +------------------- Event counter
       |                       |    |   |       |           +-------------------------- Scale To
       |                       |    |   |       +-------------------------------------- Scale From
       |                       |    |   |       +-------------------------------------- Scale From
       |                       |    |   +---------------------------------------------- Always 01
       |                       |    +-------------------------------------------------- Time Drinking in Seconds
       |                       +------------------------------------------------------- Drinking Action, water removed no tag
       +------------------------------------------------------------------------------- Message 1b

Single Animal Drink - Action = 01 and Tag Count = 1
126 22 1b 00 bb 04 9f cc 42 59 01 0f 00 01 7e 36 00 00 7e 36 00 00 b6 00 23 01 00 00 01 4a 2a 86 48 d7 f9 01
       |     --+-- -----+----- |  --+-- |  -----+----- -----+----- |     --+--       |  --------+-------- +- Tag Type 01 = FDX-B 
       |                       |    |           |           |                        |          +----------- Tag FDX-B = 999.100001000010
       |                       |    |           |           |                        +---------------------- Number of Tags = 1
       |                       |    |           |           +----------------------------------------------- Scale To
       |                       |    |           +----------------------------------------------------------- Scale From
       |                       |    +----------------------------------------------------------------------- Time Drinking in Seconds
       |                       +---------------------------------------------------------------------------- Drinking Action, water removed with 1 tag

Two Animals Drink - Action = 01 and Tag Count = 2
126 29 1b 00 bb 04 9f cc 42 59 01 3c 00 01 87 61 01 00 6c 5d 01 00 0b 00 24 01 00 00 02 4a 2a 86 48 d7 f9 01 4a 2a 86 48 d7 f9 01
       |     --+-- -----+----- |  --+-- |  -----+----- -----+----- |     --+--       |  --------+-------- +- --------+-------- +- Tag 2 Type 01 = FDX-B 
                                                                                     |          |         |          +----------- Tag 2 FDX-B = 999.100001000010
                                                                                     |          |         +---------------------- Tag 1 Type 01 = FDX-B 
                                                                                     |          +-------------------------------- Tag 1 FDX-B = 999.100001000010
                                                                                     +------------------------------------------- Number of Tags = 2

Water Bowl removed, to weight goes to negative value.
126 1b 1b 00 e8 04 9f cc 42 59 02 00 00 01 b7 87 00 00 0c d2 ff ff ca 00 21 01 00 00 00
       |     --+-- -----+----- |  --+-- |  -----+----- -----+----- |     --+--       +- Number of Tags  
       |                       |    |   |       |           |      |       +----------- Always 21-28 01
       |                       |    |   |       |           |      +------------------- Event counter
       |                       |    |   |       |           +-------------------------- Scale To
       |                       |    |   |       +-------------------------------------- Scale From
       |                       |    |   |       +-------------------------------------- Scale From
       |                       |    |   +---------------------------------------------- Always 01
       |                       |    +-------------------------------------------------- Time Drinking in Seconds
       |                       +------------------------------------------------------- Drinking Action, water removed no tag
       +------------------------------------------------------------------------------- Message 1b

Water Bowl Refilled, notice how from weight is 0.
126 1b 1b 00 3a 05 9f cc 42 59 03 00 00 01 00 00 00 00 fa 64 01 00 0b 00 24 01 00 00 00
       |     --+-- -----+----- |  --+-- |  -----+----- -----+----- |     --+--       +- Number of Tags  
       |                       |                |           +-------------------------- Scale To
       |                       |                +-------------------------------------- Scale From
       |                       +------------------------------------------------------- Drinking Action, Refilled
       +------------------------------------------------------------------------------- Message 1b
```
