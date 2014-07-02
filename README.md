Mqttsn Mote Library
==========

This repo contains 3 parts:
1)- MQTT-SN library for Arduino that has been "forked" from http://bitbucket.org/MerseyViking/mqtt-sn and https://github.com/boriz/MQTT-SN-Arduino. Sample sketches for publishing and subscribing are provided
2)- Mote library for Arduino - this provides Remote Terminal Unit [RTU] functionality to arduino with features like analog deadbands, binary debounce, pulse and latch controls, etc. The Mote protocol is an application layer protocol to the MQTTSN libraries and has been defined in the this repos wiki
3)- Mqttsn Router using Twisted Networking [python] - Data is routed between serial and UDP. The router buffers incoming data, checks for complete mqttsn packets and send complete packets on. Also decoded Mqttsn messages for debug console printout

Architecture:
-------------
The Really Small Message Broker [RSMB] is used for Mqttsn. The config is in the wiki.

Essentially split into 2 architectures:
-> Architecture 0  - direct connectivity between router and arduino over serial
-> Architecture 1 - extension to replace direct with RFM12B radio comms that is supported in MerseyViking's Mqttsn library

Architecture 0:

[RSMB]----(UDP)----[TwistedRouter]----(SERIAL)----[Arduino with Mqttsn and Mote libs]

Architecture 1:

[RSMB]----(UDP)----[TwistedRouter]----(SERIAL)----[RFM12PI]----(RADIO)----[Arduino with Mqttsn and Mote and Jeelib libs]

The wiki contains:
------------------
- Architecture diagrams
- RSMB config
- Mote protocol definition and mapping to Mqttsn
- Mqttsn reference sheet : I decided to teach myself MQTTSN and creating a reference sheet helped



Todo:
------
- Code littered with debug print statements - clean this up
- additional sample sketches
- extend mqttsn functionality - only core implemented currently
