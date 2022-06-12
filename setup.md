---
layout: page
title: Setup PetHubLocal
permalink: /setup
---
Below are the steps required to be followed to get Pet Hub Local working within your environment. It uses Home Assistant MQTT Discovery so there should be no configuration required within Home Assistant and the entities for the Hub and any attached devices should appear automagically.

## Step 1 - Update your local DNS

For pethublocal to work you **MUST** to update your local DNS server to respond with a different IP address for:
```
hub.api.surehub.io
```
Many routers support having local DNS entries or if you are running OpenWRT it is easy, however many ISP supplied routers do not support this feature so you may need a new router to poison the DNS entry. A [Mikrotik RB931-2nD](https://mikrotik.com/product/RB931-2nD) is a low-cost option and should work fine, but there are many cheap routers out there that can do it and I am not going to discuss how to set it up, so if you don't feel comfortable setting it all up then this project isn't for you.

## Step 2 - Install pethublocal

The hub requires port 80 and 443 on your host. Sorry, but that is how the hub works and unless you want to decompile and modify the firmware it is the way it is. Python 3.7 so either clone it and run locally, or deploy it in docker.

### Local install
```
git clone https://github.com/PetHubLocal/pethublocal
pip install .
```

### Docker
```
git clone https://github.com/PetHubLocal/pethublocal
docker build -t pethublocal .
```

## Step 3 - Build `pethubconfig.json` configuration from SurePetCare Cloud

It's just easier (TM) to use an existing configuration deployed in the cloud, I do plan to add support for automagically learning from the hub but if you have configured the cloud it pulls all the pet and device names and you just need to login to your cloud account.

### Local install
```
mkdir run
cd run
pethublocal setup
```

### Docker
```
docker run --rm -it --entrypoint /bin/ash --name="pethublocal" -p 80:80 -p 443:443 -v $PWD/run:/code/run pethublocal
pethublocal setup
```

You will be taken through the guided setup:

```
INFO:pethublocal:Missing Config file building from defaults
Use exist SurePetCare Cloud (C) or build empty Local config (L) C/L ?         <- Answer C for Cloud Setup
INFO:pethublocal:Building from Cloud Configuration, starting DNS check
INFO:pethublocal:This is *VITAL* as you need to update your internal DNS to point hub.api.surehub.io to this host running PetHubLocal so that the hub connects to this host not the internet
INFO:pethublocal:External DNS entry for hub.api.surehub.io: x.x.x.x
INFO:pethublocal:Internal DNS entry for hub.api.surehub.io: y.y.y.y
Is the Internal DNS updated to point to this host Y/N? Y                      <- Answer Y saying DNS is already setup
Cloud Config - Start initial setup Y/N? Y                                     <- Yes we need to do inital setup
SurePetCare Cloud EMail Address: email@domain.com                             <- EMail Address to login to SurePetCare
SurePetCare Cloud Password: password                                          <- Password, yes I don't mask it.. Meh
INFO:pethublocal:Cloud Config: Logging into Surepet to get JWT Bearer Token
INFO:pethublocal:Cloud Config: Authenticate to retrieve JWT Bearer Token: 
....
Logging into SurePetCare, and downloading the start json config
....
INFO:pethublocal:Start parsed and saved to config
Download Credentials and Firmware for Hub (highly recommended)? Y/N           <- Answer Y to download the credentials and Firmware locally, also highly recommended
INFO:pethublocal:Downloading Credentials for H0xx-0xxxxxx mac 0000xxxxxxxxxxxx
INFO:pethublocal:SureHub Host hub.api.surehub.io IP Address: x.x.x.x
....
INFO:pethublocal:Firmware successfully downloaded
MQTT Broker running on this host? Y/N                                         <- Y if MQTT Broker is the same host, or N and supply the IP address of the MQTT Broker
INFO:pethublocal:PetHubConfig config built: {"Config":...... long pethubconfig.json has been created
```

### Files created

Under the `run` folder all files are placed

- `pethubconfig.json` - Main configuration file created using the above command, all ongoing config and changes will go in here.
- `start-yyyymmdd-hhmmss.json` - The start JSON downloaded from the Cloud, which has all the profile information in it.
- `H0xx-0xxxxxx-0000xxxxxxxxxxxx-2.43.bin` - Credentials file downloaded by simulating the boot of the hub using Serial Number and MAC learnt from the cloud.
- `H0xx-0xxxxxx-1.177-xx.bin` - The 77 firmware files downloaded if you do a firmware update

## Step 4 - Setup Mosquitto MQTT Broker TLS listener 

Mosquitto MQTT on Debian/Ubuntu default configuration `/etc/mosquitto`, copy the `tls.conf` into `conf.d` to add the listener on port `8883` and the AWS certificate into `certs` and `hub.api.surehub.io` certificates the listener needs.

```
cd /etc/mosquitto
cp mqtt/tls.conf /etc/mosquitto/conf.d
cp mqtt/AmazonRootCA1.pem /etc/mosquitto/certs
cp pethublocal/static/hub.pem /etc/mosquitto/certs
cp pethublocal/static/hub.key /etc/mosquitto/certs
```

If the `/etc/mosquitto/conf.d` was empty you will need to also copy in `default.conf` so the broker will also listen on `1883` as if the directory is empty by default the broker listens on `1883` but if any config files exist the default listener isn't started, so you need to add it with the `default.conf`

Don't forget to restart:

```
/etc/init.d/mosquitto restart
```

## Step 4 - Start PetHubLocal

Start up PetHubLocal either interactively, or as a background Docker process.

### Local install
```
pethublocal start
```

### Docker
```
docker run -d --name="pethublocal" -p 80:80 -p 443:443 -v $PWD/run:/code/run pethublocal
docker update --restart unless-stopped pethublocal
```

## Step 6 - Check Home Assistant and the PetHubLocal frontend

All the entities should be created as MQTT Entites in Home Assistant.

Under Settings -> Devices & Services -> MQTT 

And all the devices and pets can be added as entites to dashboards.

The HTTP Port has a PetHubLocal frontend which displays the state of the devices and pets. http://pethublocal/index.html