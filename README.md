# meshcore_USB_control via node-red
====================

This is an embryonic project that I hope will develop into a complete node-red interface to a Meschcore USB companion.
The device used as a first approach is the\
Heltec V3 LoRa32.\
See the following address:\
[https://meshcore.co.uk/get.html]

## minimal reiquirements\
I installed Node-Red on a VMware Linux virtual machine (Ubuntu in my case)\
It doesn't require any special computing power, so in my opinion, this is an ideal solution for avoiding adding numerous applications to the host system and keeping the development environment isolated. Furthermore, everyone is well aware of the advantages of working in a virtual machine: _snapshots, copy-and-paste, move, etc._\
Naturally, you'll need a Heltec V3 and a USB cable to connect it.
In Linux, a standard user cannot access the USB port unless they're part of the "dialout" group.\
So, run this command to ensure there are no problems accessing the USB port.\
`usermod -a -G dialout $USER`\
When you insert the Heltec V3 device into the USB port, it automatically becomes visible to the host system.\
To transfer access to the virtual guest, you need to connect to it from the virtual machine menu.\
`VM - Removable Devices - Silicon CP2102 - Connect`
Now, if you haven't already done so, you need to install node-red by following the official guide at this address:
[https://nodered.org/docs/getting-started/local]
This is the message that node-red writes to my terminal when launching its instance:\

29 Jan 18:40:20 - [info] Node-RED version: v4.1.3\
29 Jan 18:40:20 - [info] Node.js version: v18.19.1\
29 Jan 18:40:20 - [info] Linux 6.14.0-37-generic x64 LE\
29 Jan 18:40:21 - [info] Loading palette nodes\
29 Jan 18:40:22 - [info] Dashboard version 3.6.6 started at /ui\
29 Jan 18:40:22 - [info] Settings file : ~/.node-red/settings.js\
29 Jan 18:40:22 - [info] Context store : 'default' [module=memory]\
29 Jan 18:40:22 - [info] User directory : ~/.node-red\
29 Jan 18:40:22 - [info] Projects directory: ^/.node-red/projects\
29 Jan 18:40:22 - [info] Server now running at http://127.0.0.1:1880/\
29 Jan 18:40:22 - [info] Active project : meshcore_USB_control\
29 Jan 18:40:22 - [info] Flows file: ~/.node-red/projects/meshcore_USB_control/flows.json\
29 Jan 18:40:22 - [info] Starting flows\
29 Jan 18:40:22 - [info] Started flows\
29 Jan 18:40:24 - [info] [serialconfig:ec7bf5fc2e81df2b] serial port /dev/ttyUSB0 opened at 115200 baud 8N1\

The last line refers to the opening of the /dev/ttyUSB0 port with baud rate 115200 and parameters N81\
This happens when a serial communication node is present in the default node-red flow.\
To use a serial communication node in node-red, you must install nodered-node-serialport.\
This operation is very easy, just access the menu\
`Manage palette - Install (tab) - search modules - nodered-node-serialport`\
then install the module.\
From now on, since I share the flows that I keep updated on Git, I don't think there will be any need to describe the communication modules settings anymore; just open them to see what's inside.

## flux description\
For now, there are only two flows.\
In the first (FLOW1), the working one, the messages sent to the Heltec module via USB will be assembled and tested. The responce messages the device retransmits to the USB port are also read in the same flow.\
The second flow contains the nodes communicating with the USB port.\
This separation is necessary to deploy only the modified flows, excluding those operating on the USB ports.\
Otherwise, the serial port would crash, and it would be necessary to close and run again node-red.\

## Implemented commands
Currently, the implemented messages are few and essential. They are only those useful for verifying the correct transmission and reception of responses.\
_query_\
_reboot_\
_start_\
_get\_time_\
_self\_advert_\
_set\_name_\
Pressing the inject button on the node corresponding to the command will generate the sequence for transmitting the command to the USB port. At the same time, if the device responds with a string, it will be captured and transmitted in the debug window.

## Future steps
As I've written several times, this is just a starting point for creating a node-red interface to a companion device.
One of the next steps will be to create a global memory to store important meshcore protocol variables, such as node data captured from issued advertisements and the meshcore nodes to use for ACC hops.\
Furthermore, the documentation of the _function_ nodes will be improved and maximized.

