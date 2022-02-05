---
layout: page
title: Hub Certificate
permalink: /certificate/
---

# Hub uses Client Certificate authentication to authenticate to AWS IoT MQTT

The hub uses a Client Certificate sent during the initial credentials response to authenticate to AWS and even though the Hub itself doesn't authenticate the server certificate it retrieves AWS **DOES** care about the Client certificate it is sent.

If you want to *man in the middle* or capture the traffic between the hub and AWS IoT MQTT server using PolarProxy or just know you can then you need a way to view the Client Certificate.

For that you will need the password for the PKCS12 certificate to then supply that to PolarProxy which is what this describes as the password for the client certificate is hard-coded into the hubs persistent flash so this talks about how to extract it

The password isn't sent over the console during a normal boot but when you hold down the `reset` button underneath the hub while powering on the hub it goes into `firmware update mode` and downloads the current firmware for your hub over HTTP.

What is **also** sent during a firmware update is the `long_serial` or certificate password over the serial console so if we solder on a TTL Serial adapter then do a Firmware Update we can capture the password.

# Understanding the hub boot process and the Client Certificate sent in the credentials response from the surehub api

When the hub boots it does a **POST** of the hubs serial number and mac address to `hub.api.surehub.io` to retrieve the hubs credentials.

```
curl -v -k -d "serial_number=H0xx-0xxxxxx&mac_address=0000xxxxxxxxxxxx&product_id=1&firmware_version=2.43" -H "curl/7.22.0 (x86_64-pc-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3" -H "Content-Type: application/x-www-form-urlencoded" -X POST -o credentials.bin https://hub.api.surehub.io/api/credentials
```

The credentials response contains a colon `:` delimited file which is used to configure the hub each time it boots, it also means that to run completely offline you need a backup of the credentials file.

```
v02:ssssss:uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu:::1:v2/production/uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu:a5kzy4c0c0226-ats.iot.us-east-1.amazonaws.com:MII...==
```

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
	Certificate:	MII...==
 	Length:		2611
	Cert Hash:	0x..
	Key Hash:	0x..
-------- End Credentials --------
```

The fields 

| Field | Example Value | Description |
| - | - | - |
| 0 | v02 | Version, which is always 'v02' |
| 1 | ssssss | ID which is serial number after the `-` with no leading zeros |
| 2 | uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu | Client ID which is just an AWS issued UUID for the MQTT Client ID |
| 3 |  | MQTT Username, which is blank but can be set if your local MQTT broker requires authentication |
| 4 |  | MQTT Password, which is also blank but can be set as required |
| 5 | 1 | Network Type, always set to 1 |
| 6 | v2/production/uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu | Base Topic, this is updated to `pethublocal/serialnumber` for pethublocal |
| 7 | xx | Certificate a base64 encoded PKCS12 AWS issued Client Certificate with a **password** that is hard-coded into the hub flash |

So we care about the last field which is the Client Certificate that we want to crack open.

## Connecting a TTL Serial Adapter to the Hub Serial Console

If you have a H010 revision hub or have already upgraded the firmware then you need to get a soldering iron out to get the password. This will of course void any warranty and don't blame me if you brick your hub, that being said, it isn't that hard. You will need a TTL 3.3v Serial Adapter, the CP2102 microusb ones are as cheap as chips on AliExpress and the advantage of the microusb is reducing the weight on the connector so search for "CP2102 Micro USB Aliexpress" and plenty of results will come up

### Hub pins
The pads on both sides of the board connect to the other side, so the below photos show the pin connections on both sides of the boards.
This may not be 100% correct so please check this before connecting anything.

<table align="center">
<tr><th>Hub </th><th>Hub Pins</th></tr>
<tr><td>

<p align="center">Hub ethernet side with pins labelled<br/>
<img src="/assets/Back-WithPins1.jpg" height="400">
<p>

</td><td>

| Pin Number | CPU Pin | Description |
| - | - | - |
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

</td></tr></table>


### Side Connector

<table align="center">
<tr><th>Side Connector </th><th> Soldering information</th></tr>
<tr><td>

<p align="center">Serial console pins<br/>
<img src="/assets/Back-Console.jpg" height="350">

</td><td>

Soldering the console I personally recommend as per [devices](/devices/) to connect to the Ground Pin 7, 4 & 5 on the side connector as I find them easier for long term connection otherwise the top pins on either side work too. Bottom two for RX/TX on the side with 5 pads and middle with the side with 3 pads.

</td></tr></table>

### Final Setup

<p align="center">
Serial console attached with ribbon cable long enough to allow the case to be put back on<br/>
<img src="/assets/ConsoleAttached.jpg" height="350">
<img src="/assets/ConsoleClosed.jpg" height="350">
</p>

#### Serial Connection on your computer

Then connect the serial console speed is: `57600 8/N/1`

On Linux I prefer minicom as it just works to capture all traffic from the command line and set the baud rate.

```
  minicom -D /dev/ttyUSB0 -b 57600 -C pethublocal.txt
```

Or if you want to use `stty` make sure to set your TTY to *raw* mode, otherwise the terminal driver might interfere by interpreting control characters.

```
  stty -F /dev/ttyUSB0 raw 57600
```

Or Windows use Putty, and make sure you log to a file, or change the **line of scrollback** to `20000`

# Downloading firmware locally

With `pethublocal` installed you can download the firmware and then start the webserver to listen on port 80:

```
  pethublocal downloadfirmware H0xx-xxxxxxx
  pethublocal http
```

This downloads the firmware based on the Hub Serial Number as each firmware is XORed using the password, or if you are using the older release of pethublocal then the first time the hub boots it downloads the firmware.

When the hub has connected to the docker stack for the first time it should automatically download the current firmware and put it into docker/output/web/H0xx-0xxxxxx-1.177-xx.bin where the last two xx are the 76 pages of the firmware, they are XOR encrypted and decrypted by the hub during the firmware update.

## BE AWARE YOU ARE JUST ABOUT TO FIRMWARE UPDATE YOUR HUB. DO NOT UNPLUG IT WHILE IT IS DOING THE UPDATE AS YOU COULD BRICK YOUR HUB JUST LEAVE IT TO COMPLETE!!!!

Then if you have it working on `57600/8/N/1` you should see the standard boot message when the hub normally. You should save the console log output to a file, as the firmware update generates about 20k lines, so if you are using Windows and Putty change the scroll back to 200000 or some large number.

The output from the firmware update is around 20,000 lines so when doing the firmware update process make sure you save everything to a file or have a large scrollback as you might miss it as it comes in the beginning. Also the firmware is XOR encoded with the key and split into around 76 4kb files. So pethublocal also downloads all the firmware locally to save for later if needed.

The interesting part of the firmware update is after the first record is sent and it extracts the password from flash:

```
serial_number=H0xx-0xxxxxx&page=0&bootloader_version=1.177
...
Read 6020 1d000000 005e 1d000000 1000 01 ....0 length=1024
0 36 6
1 30 0
..
22 31 1
23 20  
Read 6020 1d000000 5e 1d000000 1000 1
0 xx yy
1 xx yy
2 xx yy
3 xx yy
4 xx yy
5 xx yy
6 xx yy
7 xx yy
8 xx yy
9 xx yy
a xx yy
b xx yy
c xx yy
d xx yy
e xx yy
f xx yy
length=1024
```

That string of 16 bytes the first value is the offset and the `xx` values are the ones we are interested in as the `yy` is the ascii representative of it, which we don't care about.

In the python script `fwlogtopw.py` has 2 lines that are important:

```
  serial[int(linesplit[0],16)]=linesplit[1].zfill(2) #Pad zero to make a byte if it is a single character
and
  serialnumberorder = [10,7,8,11,0,5,12,13,15,1,2,14,4,6,3,9]
```

The first line creates a 16 byte array based on the first field being `0-f`, then the serialnumberorder reorders the above `0-f` fields into the correct order and upper cases the string. **This is your certificate password from the hub WOOP WOOP!!**.

Then you can check the password against the credentials file using the command:

```
awk -F":" '{print $9}' credentials.bin | base64 -d > certificate.p12
openssl pkcs12 -nodes -passin pass:**PASSWORD** -in certificate.p12 -out certificate.pem
```

If that successfully opens the PKCS12 certificate you know you have done it successfully!!

Lastly unplug the power to the hub and then hold down the "reset" button underneath the hub with a pen or something then plug the power in and you will see the the firmware update process start. This doesn't actually reset the hub, it just causes it to download the latest firmware so you won't lose your cloud configuration

# [PolarProxy](PolarProxy)

After you have the password you can use PolarProxy and view wireshark PCAPs of all traffic the hub talks to the cloud service.

# Older Hubs with original firmware

In the unlikely event The older model hubs before the H010-xx revsion they have firmware that send out the _long_serialnumber_ when they first boot up which is handy. If you setup the environment you should see the hub connect to the web server to talk to the __/api/credentials__ endpoint and send a message like:

```
 serial_number=H008-0xxxxxxx&mac_address=0000xxxxxxxxxxxx&product_id=1&long_serial=xxxxxxxxxxxxxxxxxxxxx
```

So... that last value is what you need, it's the password for the AWS Certificates so write it down, then follow the standard instructions to update the hubs firmware buy pressing and holding the reset button when you plug the power in to update the firmware.