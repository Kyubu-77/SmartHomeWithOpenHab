
Basic information 
=================

* Raspberry pi name: RASPI-SH
* Ipaddress(static): 192.168.0.201
  * Exclude static ip vom Router DHCP

Install Remote Desctop
=======================

Manual: https://goneuland.de/raspberry-pi-remote-desktop-aktivieren/


```console
sudo raspi-config

# Select "System Options"
# Select "Boot / Auto Login
# Select "Desktop"
# Finish and reboot


# Select "Advanced Options" 
# Select "Switch between X and Wayland * Backends" 
# Chose "X11"
# Finish and reboot

### prerequisite

sudo gpasswd -d $USER video
sudo gpasswd -d $USER render
sudo apt-get -y purge realvnc-vnc-server
# verify with, which shows all groups with $USER
getent group | grep $USER

### Install

sudo apt-get -y purge realvnc-vnc-server
sudo apt-get update
sudo apt-get install xrdp
# Add xrdp user to ssl-cert group
usermod -aG ssl-cert xrdp


# check if xrdp is running
sudo systemctl status xrdp
>Sep 23 22:44:36 raspi-sh systemd[1]: Starting xrdp.service - xrdp daemon...
>Sep 23 22:44:36 raspi-sh systemd[1]: Started xrdp.service - xrdp daemon.
>Sep 23 22:44:36 raspi-sh xrdp[1557]: [INFO ] starting xrdp with pid 1557
>Sep 23 22:44:36 raspi-sh xrdp[1557]: [INFO ] address [0.0.0.0] port [3389]mode 1
>Sep 23 22:44:36 raspi-sh xrdp[1557]: [INFO ] listening to port 3389 on 0.0.0.0
>Sep 23 22:44:36 raspi-sh xrdp[1557]: [INFO ] xrdp_listen_pp done
```

### Connect

In windows run "Remotedesctopverbindung" or "mstsc.exe"
In the upcomming xrpg dialog

Enter the ip of the Raspberry pi -> "Connect"
Chose  Session "Xorg"
Enter User and Password
Then OK.

### Disconnect

Disconnect with logging out 


Install Putty
=============

Download and install Putty or PuttyPortable
Open Putty and create a new Session...
Use the IP of the raspberry PI and Port 22 and open the connection to logon to the Raspi

For easy connect:
Save the connection settings to a Session in puttys settings dialog and use a proper name e.g. "Smarthome".
This session-settings can then be reused.

### For easy access
Create an Shortcut for the excutable "PuTTYPortable.exe"
and add the following paramters to the Target: "-load "Raspi5_Keller" -ssh 192.168.0.201 -l skadmin -pw <your pw>"

e.g. 'E:\Portable\PuTTYPortable\PuTTYPortable.exe -load "Raspi5_Keller" -ssh 192.168.0.201 -l skadmin -pw SAMPLE_PW'

Then the shortlink should open the connection and also login automatically


Install Docker 
==============

Manual: https://pimylifeup.com/raspberry-pi-docker/

```console
sudo apt update
sudo apt upgrade -y

cd /home/skadmin
mkdir tmp
cd tmp
curl -sSL https://get.docker.com | sh


# add own user to the docker user group
sudo usermod -aG docker $USER
sudo reboot
# check user<->group assignment, docker should be shown
groups
# test docker should show sample output
docker run hello-world
```

Directory and Environment
===============

I put all smarthome stuff to /home/skadmin/Smarthome
```
mkdir Smarthome
cd Smarthome
```

Now inside */home/skadmin/Smarthome* create a docker compose.yml file keep it empty.
Also create a docker *.env* file to store environment data which shoud not be in the compose.yml

Sample .env file:
```
SAMBA_EXT_NAME=smarthome
SAMBA_EXT_USER=<user>
SAMBA_EXT_PW=<password>
```

Install Openhab
===============

Manual: https://www.openhab.org/docs/installation/docker.html#installation-through-docker

bash:
```shell
cd /home/skadmin/Smarthome

# create user 'openhub'
sudo useradd -r -s /sbin/nologin openhab

# add user 'openhub' to group 'openhab'
sudo usermod -a -G openhab openhab

# test should show someting like 
# > "uid=999(openhab) gid=984(openhab) groups=984(openhab)" 
id openhab   

# create openhub config directories which are mounted into docker
mkdir openhub
sudo mkdir -p ./openhab/{conf,userdata,addons}

# change ownership to user 'openhab'
sudo chown -R openhab:openhab /home/skadmin/openhab
```

Now inside /home/skadmin/Smarthome create a docker compose.yml for docker and
add the openhab service to it:

compose.yml
```yml
services:
  openhab:
    image: openhab/openhab:latest
    container_name: openhab
    restart: always
    network_mode: host
    volumes:
      - "/home/skadmin/Smarthome/openhab/conf:/openhab/conf"
      - "/home/skadmin/Smarthome/openhab/userdata:/openhab/userdata"
      - "/home/skadmin/Smarthome/openhab/addons:/openhab/addons"
    environment:
      TZ: "Europe/Berlin"
      USER_ID: 999
      GROUP_ID: 984
      CRYPTO_POLICY: "unlimited"
      OPENHAB_HTTP_PORT: 8080
      OPENHAB_HTTPS_PORT: 8443
      JAVA_MIN_MEM: 4g
      JAVA_MAX_MEM: 4g
```

After creating the compose.yml file run the container

bash:
```shell
# Start container
docker compose up

# Show container
docker compose ps

# after every change compose.yml restart the container
docker compose up -d
```
If the container is running Openhab should be avialable on the Raspberrys IP-Adress 
http://192.168.0.201:8080/overview/ oder
http://RASPI-SH:8080/overview/
and if called start with the setup. 
For setup refer to https://www.openhab.org/docs/installation/


Install Samba
=============

Manual: https://github.com/ServerContainers/samba

Add samba to compose.yml services section
```yml
  samba:
    # sample config here: https://github.com/ServerContainers/samba/blob/master/docker-compose.yml
    # access in windows via \\RASPI-SH\smarthome in file explorer
    # access in windows via \\<ip>\smarthome in file explorer
    image: ghcr.io/servercontainers/samba:latest
    restart: always
    network_mode: host
    cap_add:
      - CAP_NET_ADMIN
    environment:
      MODEL: 'TimeCapsule'
      AVAHI_NAME: "samba_smarthome"

      SAMBA_CONF_LOG_LEVEL: 3

      GROUP_smarthomegroup: 444

      ACCOUNT_smarthome: ${SAMBA_EXT_PW}
      UID_smarthome: 1000
      GROUPS_smarthome: smarthomegroup

      SAMBA_VOLUME_CONFIG_smarthome: "[${SAMBA_EXT_NAME}]; path=/shares/Smarthome;: valid users = ${SAMBA_EXT_USER}; guest ok = no; browseable = yes; read only = no"
    volumes:
      - /etc/avahi/services/:/external/avahi
      - /home/skadmin/Smarthome:/shares/Smarthome
```

Start samba container

```shell
#Start container
docker compose up -d
```

Access in windows via \\RASPI-SH\Smarthome or \\<ip>\smarthome

Install Mosquitto
=================

Manual: https://www.schaerens.ch/raspi-setting-up-mosquitto-mqtt-broker-on-raspberry-pi-docker/

```shell
# Create OS User anlegen
groupadd -g 1883 mosquitto
useradd mosquitto -u 1883 -g 1883

# create directories
create 
/home/skadmin/Smarthome/mosquitto
/home/skadmin/Smarthome/mosquitto/config
/home/skadmin/Smarthome/mosquitto/data
/home/skadmin/Smarthome/mosquitto/log

# assingn the directories to the msoquitto user 
cd /home/skadmin/Smarthome/mosquitto
chown -R 1883:1883 config/ data/ log/
```

Create the mosquitto.conf file in "/home/skadmin/Smarthome/mosquitto"
```
# Config file for mosquito
listener 1883
#protocol websockets
persistence true
persistence_location /mosquitto/data/
password_file /mosquitto/data/pwfile
log_dest file /mosquitto/log/mosquitto.log
allow_anonymous false
```

### Update compose.yml
Add to the service section
```
  mosquitto:
    container_name: mosquitto
    #profiles:
    #  - donotstart
    restart: always
    image: eclipse-mosquitto
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - /home/skadmin/Smarthome/mosquitto/config/mosquitto.conf:/mosquitto/config/mosquitto.conf
      - /home/skadmin/Smarthome/mosquitto/data:/mosquitto/data
      - /home/skadmin/Smarthome/mosquitto/log:/mosquitto/log
    networks:
      - default
```

```shell
#Start container
docker compose up -d
```

Create mosquitto user inside the docker container

```shell
# step into mosquitto container
docker exec -it mosquitto sh
#mosquitto user anlegen
mosquitto_passwd -c /mosquitto/data/pwfile mqtt-user
# leave container
exit
```

### Test the mosquitto server

# Test with 2 shells
# Shell 1
```shell
docker exec -it mosquitto sh
mosquitto_sub -u mqtt-user -P <userpw> -d -t home/kitchen/temperature
```

```shell
# Shell 2
docker exec -it mosquitto sh
mosquitto_pub -u mqtt-user -P <userpw> -d -t home/kitchen/temperature -m "21.3"
```

Install chiptool
================

Manual: https://github.com/canonical/chip-tool-snap 

### Install
```shell
# install snapd
sudo apt install snapd
sudo reboot
sudo snap install snapd

# install chip-tool
snap install chip-tool
```

For easy call of chiptool add the path to **.bashrc**
```shell
echo 'export PATH=$PATH:/snap/bin' >> ~/.bashrc
source ~/.bashrc
```

or add to own configuation file **.myconfig**
```
export PATH=$PATH:/snap/bin
alias sh='cd /home/skadmin/Smarthome'
```

and source .myconfig from .bashrc

bash
```shell
echo 'source ./.myconfig' >> ~/.bashrc
source ~/.bashrc
```

### BLE-Interface verbinden
```shell
# allow  that 
snap connect chip-tool:bluez
snap connect chip-tool:avahi-observe
snap connect chip-tool:process-control
```
If avahi and bluez are missing
```shell
sudo snap install avahi bluez
snap connect chip-tool:bluez bluez:service
snap connect chip-tool:avahi-observe avahi:avahi-observe
```

Openthread border router to docker
==================================

Manual: https://openthread.io/guides/border-router/build-docker

curl -sSL https://raw.githubusercontent.com/openthread/ot-br-posix/refs/heads/main/etc/docker/border-router/setup-host | bash


```
# enable ip forwarding
curl -sSL https://raw.githubusercontent.com/openthread/ot-br-posix/refs/heads/main/etc/docker/border-router/setup-host | INFRA_IF_NAME=eth0 bash

# get Docker image
docker pull openthread/border-router:latest

# check docker image 
docker images

# create folder fro thread border router
cd /home/skadmin/Smarthome
mkdir openthread_border_router
```

There is no need to create and otbr-env.list file since everthing can go into the compose.yml file

Add openthread-border-router service to compose.yml
```
openthread-border-router:
    container_name: openthread-border-router
    image: openthread/border-router
    restart: unless-stopped
    network_mode: host
    cap_add:
      - NET_ADMIN
    environment:
      # works with sonoff dongle max https://sonoff.tech/de-de/products/sonoff-dongle-max-zigbee-thread-poe-dongle-dongle-m?srsltid=AU7gw4ULsXuwEU5Di3sY1ONrSDIKuOt4AM0Wpu48nxVrgVF68H1w-xxl 
      # which i have connected directly via LAN to the router and via USB to the Raspsberry
      OT_RCP_DEVICE: "spinel+hdlc+uart:///dev/ttyUSB_THREAD?uart-baudrate=115200"
      OT_INFRA_IF: "eth0"
      OT_THREAD_IF: "wpan0"
      OT_LOG_LEVEL: 7
      OT_WEB_LISTEN_ADDR: "0.0.0.0"
      # this must be different to port 8080 which is used for openhab
      OT_WEB_LISTEN_PORT: 8084
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB_THREAD
      # This is not working but would be better since it addresses the physical USB connector 
      #- "/dev/bus/usb/003/002:/dev/ttyUSB_THREAD"
      - /dev/net/tun:/dev/net/tun
    volumes:
      - "/home/skadmin/Smarthome/openthread_border_router:/data"
```

Shell
```
#Start container
docker compose up -d
```

Browser:
http://192.168.0.201:8084/
http://RASPI-SH:8084/


Forming a thread network: 
=========================


Manual: https://openthread.io/guides/border-router/form-network
This can be done via shell or via OBTR UI "Form" on http://RASPI-SH:8084/
Shel
```
docker exec -it openthread-border-router ot-ctl
# create new dataset
dataset init new

# test via dataset
dataset
>Active Timestamp: 1
>Channel: 16
>Wake-up Channel: 23
>Channel Mask: 0x07fff800
>Ext PAN ID: c07c26f53994fe16
>Mesh Local Prefix: fd42:efba:afcf:72f1::/64
>Network Key: 06fce7cfbe3d47b9fcd5793d2f0b7945
>Network Name: OpenThread-7369
>PAN ID: 0x7369
>PSKc: 79f4c78a18d09e2a0820263c643c2a52
>Security Policy: 672 onrc 0
>Done


#Commit new dataset to the Active Operational Dataset in non-volatile storage.
dataset commit active

#Show the dataset
dataset active

#Enable Thread interface.
ifconfig up
thread start

exit
```

Show network:
Shell
```
ifconfig wpan0
>wpan0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1280
>        inet6 fd76:90ac:73ae:c18a:ed75:69fd:2e31:7260  prefixlen 64  scopeid 0x0<global>
>        inet6 fd76:90ac:73ae:c18a:0:ff:fe00:d400  prefixlen 64  scopeid 0x0<global>
>        inet6 fd76:90ac:73ae:c18a:0:ff:fe00:fc38  prefixlen 64  scopeid 0x0<global>
>        inet6 fd76:90ac:73ae:c18a:0:ff:fe00:fc11  prefixlen 64  scopeid 0x0<global>
>        inet6 fe80::ac15:8e3d:771a:fe47  prefixlen 64  scopeid 0x20<link>
>        inet6 fd76:90ac:73ae:c18a:0:ff:fe00:fc10  prefixlen 64  scopeid 0x0<global>
>        inet6 fd01:82dc:7bcf:1:ae43:8819:5f07:5226  prefixlen 64  scopeid 0x0<global>
>        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 500  (UNSPEC)
>        RX packets 305  bytes 34993 (34.1 KiB)
>        RX errors 0  dropped 2  overruns 0  frame 0
>        TX packets 81  bytes 15427 (15.0 KiB)
>        TX errors 0  dropped 2 overruns 0  carrier 0  collisions 0
```


## Paaring

Manual: https://community.simon42.com/t/ikea-matter-thread-geraete-auf-raspberrypi-docker-commissionen-via-chip-tool-ohne-smartphone/81720#google_vignette

### Prereq:

sysctl net.ipv6.conf.all.forwarding    # MUSS 1 sein!
sysctl net.ipv6.conf.eth0.accept_ra    # MUSS 2 sein!

If thats not the case 

Shell 
```
sudo nano /etc/systemd/system/ipv6-forwarding.service
```

Add to file:
```
[Unit]
Description=Enable IPv6 Forwarding (after Docker)
After=docker.service
Wants=docker.service

[Service]
Type=oneshot
ExecStart=/sbin/sysctl -w net.ipv6.conf.all.forwarding=1
ExecStart=/sbin/sysctl -w net.ipv6.conf.eth0.accept_ra=2
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Shell 
```console
systemctl daemon-reload
systemctl enable --now ipv6-forwarding.service
```

### check env

```console
#Bluetooth muss UP RUNNING sein
hciconfig hci0
>hci0:   Type: Primary  Bus: UART
>        BD Address: 2C:CF:67:93:D2:63  ACL MTU: 1021:8  SCO MTU: 64:1
>        UP RUNNING
>        RX bytes:3771 acl:0 sco:0 events:398 errors:0
>        TX bytes:68389 acl:0 sco:0 commands:398 errors:0

# chip-tool muss antworten
chip-tool pairing
>[1789859596.779] [5950:5950] [TOO] Missing command name
>Usage:
>  /snap/chip-tool/316/bin/chip-tool pairing command_name [param1 param2 ...]


# OTBR muss "leader" sein
curl -s http://localhost:8081/node/state

# Thread-Dataset muss abrufbar sein
curl -s -H 'Accept: text/plain' http://localhost:8081/node/dataset/active
> 0E08000000000...
```

Install Matter Plugings for openhab
=====================================
Prerequisite:
* Install the Matter binding from the openHAB add-on store.
  * Open openhub  http://192.168.0.201:8080/addons/
  * Goto Bindings or http://192.168.0.201:8080/addons/binding/
  * Add the "Matter Binding"
 
* Add a Matter "Controller" Thing to the inbox using the default settings.
  * Open Thing settings http://192.168.0.201:8080/settings/things/
  * Press add
  * Select "Matter Binding"
  * Select "Matter Controller"
  * Presse Create Thing


Install a Matter/Treads device
=====================================


### Overview:
Schritt 1: BLE-Pairing
* Dein RPi verbindet sich per Bluetooth Low Energy mit dem neuen Gerät.
* Das ist der Grund warum ein Smartphone/BLE nötig ist.

Schritt 2: Thread-Credentials übergeben
* Der RPi schickt dem Gerät die Thread-Netzwerk-Daten (Network Key, Channel, PAN ID).
* Das Gerät weiß jetzt wie es ins Thread-Netz kommt.

Schritt 3: Gerät tritt Thread-Netzwerk bei
* Das Gerät verbindet sich mit deinem Thread Border Router.
* Ab jetzt ist es über IPv6 erreichbar, BLE wird nicht mehr gebraucht.

Schritt 4: Multi-Admin (Fabric-Übergabe)
* chip-tool öffnet ein Commissioning-Fenster auf dem Gerät.
* Home Assistant kann das Gerät jetzt als zweite "Fabric" übernehmen.

Schritt 5: In HA hinzufügen
* HA's Matter-Server kommuniziert über IPv6/Thread mit dem Gerät.
* Kein BLE mehr nötig, alles geht über Thread.

### Execution

Sample: Ikea Stecker 3248-791-8936

# Schritt 1:
Check manual of Ikea Stecker to do activate pairing mode

# Schritt 2:
Get pairing code 
from print on Ikea Stecker:  "3248-791-8936"
form qr code(some sample): MT:Y3.13OTB00KA0648G00
or
chip-tool payload parse-setup-payload 3248-791-8936
[1789860564.195] [7347:7347] [SPL] Version:             0
[1789860564.195] [7347:7347] [SPL] VendorID:            0
[1789860564.195] [7347:7347] [SPL] ProductID:           0
[1789860564.195] [7347:7347] [SPL] Custom flow:         0    (STANDARD)
[1789860564.195] [7347:7347] [SPL] Discovery Bitmask:   UNKNOWN
[1789860564.195] [7347:7347] [SPL] Short discriminator: 13   (0xd)
[1789860564.195] [7347:7347] [SPL] Passcode:            31023407

# Schritt 3:
Get dataset vom OTBR and store in env variable dataset

```
DATASET=$(curl -s -H 'Accept: text/plain' http://localhost:8081/node/dataset/active)
echo $DATASET
```

# Schritt 4 Pairing 


* Put the descice in paring mode: e.g for plug TOFSMYGGA
  * Hold the button until the LED flashes red and then stops (~10 seconds) to perform a factory reset.
  * The plug is now in Thread pairing mode
  * (skip is using thread) Press the button 4 times
  * (skip if using thread) Press the button 8 times.
* The plug should be no in Pairing mode

Start the comissining process via chip-tool

```
# sample call with parameter
chip-tool pairing code-thread <NODE_ID> hex:$DATASET <PAIRING_CODE> \
  --bypass-attestation-verifier true \
  --timeout 120
#Concrete call with node id 100 (the node id is just a consecutive number)
chip-tool pairing code-thread 100 \
  hex:$(curl -s -H 'Accept: text/plain' http://localhost:8081/node/dataset/active) \
  32487918936 \
  --bypass-attestation-verifier true \
  --timeout 120
  or 
chip-tool pairing code-thread 100  hex:$DATASET  32487918936  --bypass-attestation-verifier true  --timeout 120   
```

# Step 5:
```
# sample call with parameter
chip-tool basicinformation read product-name <NODE_ID> 0
#concrete for node 100
chip-tool basicinformation read product-name 100 0
# There should be shown someting like:
>[1789937143.543] [4674:4707] [DMG] ReportDataMessage =
>[1789937143.543] [4674:4707] [DMG] {
>[1789937143.543] [4674:4707] [DMG]      AttributeReportIBs =
>[1789937143.543] [4674:4707] [DMG]      [
>[1789937143.543] [4674:4707] [DMG]              AttributeReportIB =
>[1789937143.543] [4674:4707] [DMG]              {
>[1789937143.543] [4674:4707] [DMG]                      AttributeDataIB =
>[1789937143.543] [4674:4707] [DMG]                      {
>[1789937143.543] [4674:4707] [DMG]                              DataVersion = 0x7da52a3a,
>[1789937143.543] [4674:4707] [DMG]                              AttributePathIB =
>[1789937143.543] [4674:4707] [DMG]                              {
>[1789937143.543] [4674:4707] [DMG]                                      Endpoint = 0x0,
>[1789937143.543] [4674:4707] [DMG]                                      Cluster = 0x28,
>[1789937143.543] [4674:4707] [DMG]                                      Attribute = 0x0000_0003,
>[1789937143.543] [4674:4707] [DMG]                              }
>[1789937143.543] [4674:4707] [DMG]
>[1789937143.543] [4674:4707] [DMG]                              Data = "TOFSMYGGA plug outdoor" (22 chars),
>[1789937143.543] [4674:4707] [DMG]                      },
>[1789937143.543] [4674:4707] [DMG]
>[1789937143.543] [4674:4707] [DMG]              },
>[1789937143.543] [4674:4707] [DMG]
>[1789937143.543] [4674:4707] [DMG]      ],
>[1789937143.543] [4674:4707] [DMG]
>[1789937143.543] [4674:4707] [DMG]      SuppressResponse = true,
>[1789937143.543] [4674:4707] [DMG]      InteractionModelRevision = 11
>[1789937143.543] [4674:4707] [DMG] }
>
```
# Step 6: Start Commissioning

Commissioning-Fenster für HA öffnen

```
# sample call with parameter and timeout 5 minutes timout
chip-tool administratorcommissioning open-basic-commissioning-window 600 <NODE_ID> 0  --timedInteractionTimeoutMs 5000
# concrete
chip-tool administratorcommissioning open-basic-commissioning-window 600 100 0  --timedInteractionTimeoutMs 5000
```

# Step 7:

* Enter Paring mode
* Goto Things settings http://192.168.0.201:8080/settings/things/
* Select Matter Controller 
* Select Pair a Matter device
* Enter the Pairing Code from your device e.g. "32487918936"
* Make sure the chip-tool command from step 6 is still running
* Press Execute Action
* There should be the answer "Device added to inbox"
* Close the dialog and go the the thing inbox(url http://192.168.0.201:8080/settings/things/ -> orange button)
* Chose Add as New Thing and enter a label
* See and configure the thing TOFSMYGGA in the Things menu
* Check if the channels and configure as any other thing
