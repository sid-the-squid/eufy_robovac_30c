Hi all, welcome to my latest project.  Making a Eufy RoboVac 30C integrate with Homeassistant!

TLDR;
- open robot
- solder wires
- compile a binary using the yaml with ESPHome
- flash binary to 'wifi module'
- see it in HA
- enjoy

__Long version.__

As mentioned previous foray into Eufy RoboVac modding/hacking for the 11S I found some clues online to suggest the 15C and 30C had ESP chips which handled the WiFi control, this basically sealed the deal for me to purchase and investigate one of those RoboVacs, so I did, in the form of a 30C, and I was not disappointed :)

Now there are already projects to integrate Eufy Robovacs into HA, https://github.com/mitchellrj/eufy_robovac being the most popular I could find, however these still rely on the Tuya component being present and piggybacking off that, I wanted a more basic solution which could rip out any chance of the Tuya cloud messing things about in the future, so I got my screwdriver kit out and my soldering iron and got to disassembling the robot!

![open RoboVac](/pictures/RoboVac/robo1.jpg)

Opening up the Robot it is very clear where the WiFi module is.
and thankfully we have all the pins we need ladled on the board, very nice of Eufy to do that for us ;)

![wifi module](/pictures/RoboVac/wificard.jpg)

Even with my shonkey soldering skills I managed to get the 5 wires we need connected.
3.3V, GND, TX, RX, GPIO0.

3 are on one side, 2 on the other side
yes I did check for shorts and cleaned up those loose wires!!

![wire one](/pictures/RoboVac/wires1.jpg)

I also decided I would keep these wires connected just in case I needed to flash this back to stock, or something went horribly wrong.

![wires with glue](/pictures/RoboVac/wires2.jpg)

During the making of this project I stumbled or should that be googled my way onto the blog of Daniel Humpf (www.dhumpf.de), who's blog I think deserves more attention, in his blog he describes using Tasmota to control a RoboVac 30C, I thought why re-invent the wheel and thus that is the path I started down with this project.  

Things worked exactly as Daniel described in his blog, however I wanted more of the 'commands' mapped to switches and sensors, as I'm not very familiar with Tasmota I soon got frustrated with the command mapping, so decided to switch to my favourite ESP manipulation/programming solution, the wonderful ESPHome, which I think ultimately has led to me be able to create a much more complete Homeassistant integration, being as its ESPHome you don't HAVE to use HA, you can use its MQTT functionality to integrate with any smart home manager, I won't be covering using the MQTT here but the documentation is out there to very easily change this project yaml to MQTT.

Therefore the start of this guide will deal with installing Tasmota, and the commands which are known in Tasmota..

if you too decide to start on the Tasmota route (not recommended) but then decide to go to ESPHome it is then very easy to flash an ESPHome binary, using tasmota's web interface

Alternatively just compile the ESPHome binary first (from the yaml provided in this guide) and use in the intial flash instead of tasmota.bin

First off we need to setup a flashing station for the ESP chip, unlike off the shelf chips we don't get the luxury of build in USB, so for this task I would recommend using a raspberry pi.

Follow the guide to setup your rpi for tasmota flashing via uart here
https://tasmota.github.io/docs/Flash-Sonoff-using-Raspberry-Pi/

Make sure you have the following packages installed
~~~~
sudo apt-get install python3 python3-pip python-serial esptool
~~~~
now that everything is setup you can connect the pins into the pi or in our case breadboard
note the TX and RX pins need to cross, so TX from the ESP should go to RX on the pi and vice versa

before connecting the power to the esp, attach GPIO 0 on the ESP to GND, this puts it in flash mode

pic showing me setting up the connection to my flashing Pi

![flashing station](/pictures/RoboVac/flash1.jpg)

first thing is to determine our flash size, some say its 1mb, some 2mb, so lets see if we can find out

run this command
~~~~
esptool.py --port /dev/ttyS0 flash_id
~~~~
it should output something like this
~~~~
Detecting chip type... ESP8266
Chip is ESP8266EX
Features: WiFi
Crystal is 26MHz
MAC: e8:68:e7:52:06:0c
Uploading stub...
Running stub...
Stub running...
Manufacturer: c8
Device: 4015
Detected flash size: 2MB
Hard resetting via RTS pin...
~~~~
mine is a 2mb flash type, so I need this command to dump the existing flash content in case I want it for later
~~~~
esptool.py --port /dev/ttyS0 read_flash 0x00000 0x200000 backup.img
~~~~
verify our dump, I was a tad unclear on the verify command, so tried these two
~~~~
esptool.py --port /dev/ttyS0 verify_flash --diff yes 0x00000 backup.img

esptool.py --port /dev/ttyS0 verify_flash --diff yes 0x200000 backup.img
~~~~
both commands told me
~~~~
-- verify OK (digest matched)
~~~~
so now we go on to flashing tasmota (or ESPHome)

now erase the flash
~~~~
esptool.py --port /dev/ttyS0 erase_flash
~~~~
now flash it with tasmota
~~~~
esptool.py --port /dev/ttyS0 write_flash -fm dout 0x0 tasmota.bin
~~~~
now disconnect the 3.3v and GPIO0, TX and RX pins.
reconnect 3.3v and the esp should boot

__If you're using the ESPHome yaml you're done! log in to HA and enjoy__

__If you're going down the Tasmota route.__
keep reading!

log into the new tasmota wifi hotspot and configure your wifi, once completed the device should appear on your lan
open its ip address in a web browser, now the tasmota config can begin.

Configure Tasmota in this fashion.

![config](/pictures/tasmota/tasmota_config.png)

now we need to do some console configure

set this flag in the console to enable fast UART 0 = 9600 or 1 = 115200
~~~~
SetOption97 1
~~~~
now we need to 'map' the tuya commands to tasmota things
detals here https://tasmota.github.io/docs/TuyaMCU/

in essence there are tuya id's called (dpId), these can be mapped to tasmota Id's (fnId)
Command TuyaSend is used to send commands to dpId's

The commands captured by Heir Dumfp are as follows
~~~~
55 AA 00 06 00 05 05 04 00 01 00 14
-> Auto -> TuyaSend4 5,0

55 AA 00 06 00 05 05 04 00 01 01 15
-> 30 min -> TuyaSend4 5,1

55 AA 00 06 00 05 05 04 00 01 02 16
-> Spot -> TuyaSend4 5,2

55 AA 00 06 00 05 05 04 00 01 03 17
-> Edges-> TuyaSend4 5,3

55 AA 00 06 00 05 02 01 00 01 00 0E
-> Stop -> TuyaSend1 2,0
-> Start -> TuyaSend1 2,1

55 AA 00 06 00 05 65 01 00 01 01 72
-> Home -> TuyaSend1 101,1

55 AA 00 06 00 05 67 01 00 01 01 74
-> Find Start -> TuyaSend1 103,1

55 AA 00 06 00 05 67 01 00 01 00 73
-> Find Stop -> TuyaSend1 103,0

55 AA 00 06 00 05 03 04 00 01 00 12
-> go Forward-> TuyaSend4 3,0

55 AA 00 06 00 05 03 04 00 01 01 13
-> go Backward-> TuyaSend4 3,1

55 AA 00 06 00 05 03 04 00 01 02 14
-> go left -> TuyaSend4 3,2

55 AA 00 06 00 05 03 04 00 01 03 15
-> go right -> TuyaSend4 3,3
~~~~

__Creating some Tasmota mapping__
~~~~
TuyaMCU 11,2
~~~~
this maps 'Relay1' (fnId 11) to dpId 2 (Start/Stop)
~~~~
TuyaMCU 12,101
~~~~
this maps 'Relay2' (fnId 12) to dpId 101 (Go Home)
~~~~
TuyaMCU 13,103
~~~~
this maps 'Relay3' (fnId 13) to dpId 103 (Locate Mode)
~~~~
TuyaMCU 31,104
~~~~
this maps 'Power' (fnId 31) to dpId 104 (Battery Percentage)
this shows up in HA as power in watts, when actually its battery % divided by 10.

if you encounter any issues check the tasmota console, enable verbose log
weblog 4

as I'm going to integrate into homeassistant configure the MQTT as well.
if you use anonymous MQTT logins set the following in the console
MqttUser 0
MqttPassword 0

Sadly that’s as far as I got with Tasmota, I couldn't figure out how to map the ENUM's to switches or sensors.

So I switched to ESPHome, I've included the yaml for that here, and as you can see I've got all commands mapped to switches, and all status's returning raw and a text_sensor for a human readable status.

__Final thoughts/future TODO__

I haven't mapped all the error codes, as I didn't want to deliberately hurt the robot, and could only easily simulate 1 type of error.

Sadly the Homeassistant lovelace card isn't that impressive, maybe one day I'll figure out how to do something pretty, or maybe some other more talented individual will find this and build something out of it :)

Well I hope you have fun, I know I did :)
