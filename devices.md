---
layout: page
title: Devices
permalink: /devices/
---

There are five Connect devices that SurePet sell in New Zealand:

# List of devices
1. [Hub](#hub)
2. [Doors](#doors)
    - [Door Differences](#door-differences)
3. [Feeder](#doors)
4. [Felaqua](#doors)

# Hub

This is the device I have focused on and messages are primarily 132 style register messages where certain register offsets are used for certain functions. 

## Registers

| Register | Description |
| -- | -- |
| 3-6 | Firmware Version |
| 7 | Hardware Version |
| 15 | Adoption Mode to adopt new devices |
| 18 | Flash LEDs |
| 31-34 | Device uptime |
| 44 | Number of registered devices |
| 45-52 | Registered Device 1 Mac - 8 Bytes |
| 53-60 | Registered Device 1 State - 8 Bytes |
| 61-68 | Registered Device 2 Mac |
| 69-76 | Registered Device 2 State |
| .. | Device 3 - 9 |
| 189-196 | Registered Device 10 Mac |
| 197-204 | Registered Device 10 State |

## Hub Teardown
This documents up the known pins for the hub that has the following:

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

### Using PICKIT3

A site detailing using the PICKIT3 to connect up to a PIC32MX695 CPU to extract the firmware.

https://blog.rapid7.com/2019/04/30/extracting-firmware-from-microcontrollers-onboard-flash-memory-part-3-microchip-pic-microcontrollers/

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

The serial port speed is 57600 8/N/1 with flow control disabled.

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

| Connector Pin | aka Pin | CPU Pin | Description |
| -- | -- | -- | -- |
| 1 | 6 | 10, 26, 38 | Vdd - 3.3v Positive supply |
| 2 | - | NC |
| 3 | 7 | 9,25,41 | Vss - Ground |
| 4 | 1 | 17 | PGC2 used for ICSP debugger |
| 5 | 3 | 18 | PGD2 used for ICSP debugger |
| 6 | 2 | 7 | /MCLR via 100R |
| 7 | 4 | 31 | U2RX - UART RX for the Console output |
| 8 | 5 | 32 | U2TX - UART TX for the Console output |

# Console Boot Message:

The hub doesn't output too much during boot until you enable debug messages:

Standard boot message:

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

The above MAC address is the Zigbee Wireless MAC address not the ethernet MAC address

## Debug Menu
The Debug menu can be enabled if you send an upper case "A"

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

    Registers 27-30, uptime in seconds.

`h` is the same output as sending a `TS 1000 3 0 205` message over MQTT.
`z` is the same as pressing the button underneath to start paring.

`p` Pairing table you get:

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


If you enable debug when the hub first boots you get a lot more information:

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

# Doors

There is a Pet Door and Cat Flap, the pet door is obvious larger designed to take either a large cat, or a small dog and my 12.5kg dog which is a Lowchen happily uses the Pet Door.
A major disadvantage of all the Connect devices is they all run on batteries and only the Pet Door supports re-chargeable batteries.

## Door differences

Pet Door advantages:
- Supports a larger animals, but even a largeish cat should be fine with the cat flap.
- Uses 'C' batteries and also supports re-chargables if you select Custom Mode 2.
- You can select the Custom Mode via the buttons on the pet door, or send them via the cloud.
- Can unlock both directions with Non Selective Entry mode so door mechanism doesn't need to unlock for each entry improving battery life so you should get 6+ months life with Energizer Max Alkalines.
- Buttons on the front and LCD on top of the pet door to visually check the situation.

Cat Flap advantages:
- Runs on AA batteries which are typically cheaper, but does not support re-chargables.
- Lock modes are separate from curfew mode, so curfew can be set along with lock in/out which isn't supported on Pet Door as either you have curfews or locked state.
- Support for four curfew times, pet door only supports one.
- Support for dual scan, so scans to let animal out as well as in. Pet door does not have the ability to scan pet on the inside.
- Because of dual scan you can set lock state of a particular cat to indoor only when others can exit.

## Pet Door

The Pet Door sends a number of messages, the main ones we are interested in are the 132 and the 8.

## Message type 8
The 8 messages are generated by the Pet Door when an animal comes in or leaves through the door.

## Message type 132
The 132 messages are a register offset that all values are in big endian byte order

## 132 Door Registers

| Register | Description |
| -- | -- |
| 33 | Battery ADC Value  |
| 34-35 | Door Time in local HH:MM |
| 36 | Lock State 0-4 |
| 40 | Inbound lock state for keep pets out |
| 59 | Provisioned Chip Count |
| 60 | Next free chip slot |
| 61-63 | Custom Mode in big byte order |
| 91-309 | Provisioned tags |
| 519 | Curfew mode lock / unlock time |
| 525-618 | Pet movement state in or out |
| 621 | Unknown pet went outside |

## Cat Flap

The cat flap doesn't send registers it messages are the same as all

