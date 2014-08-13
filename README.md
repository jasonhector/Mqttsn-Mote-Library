#Quick Start Guide for Architecture 0
TODO easy steps to get up and going quickly.

1. pre- requisites: arduino, rsmb and TwistedRouter. An understanding of MQTTSN wouldnt hurt either.
2. Download the sample sketches

#Intended Application
The intended application behind this is a cost effective wireless/radio home automation systems consistent of distributed sensors and control elements. Energy monitoring is currently being done using mqtt, but light control, presence detection, distributed temperature, are a few of the end elements.

##OpenHAB
[http://www.openhab.org/](http://www.openhab.org/) is the DIY home automation system that is used. It is extensive in interfaces/ bindings of which mqtt is one of them. This is the main mqtt client that will interface to the rsmb broker.

# Architecture Diagrams
TODO

# RSMB
Really Small Message Broker can be found at <insertlink> and instruction for compiling at <insertlink>. Once compiled create a **broker.cfg** file with content as below:

```
\# will show you packets being sent and received
trace_output protocol                                   

\# normal MQTT listener
listener 1884 INADDR_ANY 

\# MQTT-S listener 
listener 1884 INADDR_ANY mqtts     
\# optional advertise packets parameters: address, interval
\#       advertise 127.0.0.1:1885 20 1
```

#MQTTSN Python Client
The MQTTSN clients within the RSMB folders have been modified and are available in this repo under the folder xxx. Specifically a publish python script that accepts user inputs of topic and paylod before publishing.


#Twisted Router
## Why socat doesnt work
Initially I used `sudo socat -x -v -t 0.01 /dev/ttyACM0,raw,echo=0,b115200 UDP4:127.0.0.1:1884` for routing between serial and UDP but soon found out that socat was not encapsulating complete mqttsn messages into single UDP packets. RSMB doesnt like this one bit and rejected connections with DISCONNECT messages. I found that increasing the serial baudrate did help slightly but one in every 5 messages was split and the connection terminated.
So I went to my favourite [python twisted networking library](https://twistedmatrix.com/trac/) and re-used a router function I developed earlier that routed between udp and tcp. This time I added a MqttsnSerial protocol to the router framework for routing between udp and serial and viola...it worked with no mqttsn message splitting. Inside the MqttsnSerial protocol I add incoming data to a deque [bi directional queue] and then decode the head of the queue to determine the Mqttsn length. With the length I then pop off the right amount of data or a complete mqttsn message. I also decode and console display the Mqttsn packets using the sample client python lib in RSMB 
TODO Sometimes the serial buffers to the router contains too much data and lock up. this needs fixing, but for now I use Ctrl-z and then `fuser -n udp -k 1884` to free up the udp port. I then restart the router.
NOTE:: The TwistedRouter has been configured and tested on Linux. I think there are windows drivers for the serial part of TwistedMatrix but I have not tested this.

# Mote Library
TODO provides ....debounce, delta, pulse and latch controls, 
the mote library provides the application layer for Mqttsn. Its is the interface between Mqttsn and the hardware io found in arduino. It comprises of a few features typically found in Remote Terminal Units [RTUs] which are used for SCADA applications [Can u guess my background?]. 

There is also a Mote library written in Python specifically for the Raspberry PI.
The features are as follows:
###Pulse Controls ###
Depending on the control interface linked to the arduino, you might only need to send a pulse signal to effect a control signal. For example to open my garage door needs a pulse of a few hundrede milliseconds only. The Mote library allows this by using the `$P` object type [explained later] with the Mote/ Mqttsn payload being the pulse duration. This single message will have the two stage effect used in pulse signal [rising and falling edge] 
###Latch controls###
Easier to implement, this control `$L` simply latches the binary output to the desired state. 
###Analog Delta###
Together with pulse controls, this is probably one of the more valuable features. an analog delta sets a windows for analog values that only when exceeded will report the analog value via an mqttsn message. For example if the temperature is being sensed on arduino, we can set the analog delta to 2. If the current value of temperature is 21, then only when the temperature rises above 23 or drops below 19, will the Mote library callback publish an mqttsn message with the value. This is a very useful technique to control/ limit the traffic on the network as larger deltas/deadbands have fewer messages being sent on the network. The deltas are configurable when instantiating io within the Mote lib.

TODO insert graphic of delta

###Binary Input Debounce###
As binary inputs change state, say from 0 to 1 there could be many noisy transitional states inbetween before the binary settles at 1 ie transition from 1-0 and 0-1 maany times before settling. This is called debounce and until the binary settles we dont want to send mqttsn messages of the transitional unsettled states. So the debounce in Mote library waits a debounce time before re-reading the binary input and if the settled state changed from the pre-state, then only is a mqqtsn message sent.  
# Pairing Mqttsn and Mote Libraries
Mqttsn is great but doesnt provide an application layer definition. Thats where the Mote lib comes in. It is the application interface between Mqttsn and hardware io.  So the combination looks something like:

` (network/radio)----[Mqttsn lib][Mote lib]----(Hardware IO)`

The arduino sketch is the glue linking the two libraries with the callback of the one calling methods in the other. The loop of the sketch also calls Motes `scanIo()` and Mqttsn's `poll()` recursively.

# Mote Protocol Definition
Initially designed independantly of Mqttsn. The Mote lib is a simple stateless protocol aimed to cater for binary outputs, analog inputs, binary inputs and time. Outputs/ Control use the `$` prefix while the `@` prefix indicated inputs. 
The following are the object types:

* `$P = Pulse Control with value/payload being the pulse time in ms`
* `$L = Latch Control with value/payload being 1(ON) or 0(OFF)`
* `$A = Analog Control with value/payload being the analog`
* `$T = Time Control with value/payload being [year]-[month]-[day]-[hour]-[min]-[sec]`
* `@I = Indication (Important Binary)  [think QOS 1 or 2] with value/payload being being 1(ON) or 0(OFF)`
* `@S = Status (Ordinary Binary) [think QOS0] with value/payload being being 1(ON) or 0(OFF)`
* `@A = Analog Input with value/payload being the analog`

The types of all the above are integers except for `$T` which is a char array/ string type. TODO implement `$T` in the Mote library. 

Initially the Mote definition looks like this:

`[startByte] [srcNode] [destNode] [objType] [index] [value]`

* **startByte** - not applicable when using Mqttsn
* **srcNode** - this is the node address of the source of the mqttsn message. This should correspond to the RFM12B node address.
* **destNode** - this is the node address of the destination of the mqttsn message. This should correspond to the RFM12B node address
* **objType** - Defines the type of message. Types explained above
*  **index** - a unique index within the io of the arduino. This corresponds to the id supplied to the Mote for a particular io and not the pinid. The ranges used for the different io types are given later.
*  **value** - the value or payload of the message
## Mote-Mqttsn mapping
Mapping the Mote definition onto Mqttsn was relatively painless with the topic being:

`TOPIC: [srcNode] / [destNode] / [objType] / [index]`

`PAYLOAD: [value]`

The startbyte was dropped in the mapping.

The topic mapping from Mote to Mqttsn lends itself to a favourable hierarchy filter within Mqttsn.

##Mote ID recommended index ranges
The recommended index ranges to use for the different **ojectTypes** are:

* @S and @I 	->> 1-50
* @A			->> 51-100
* $L			->> 101-150
* $P			->> 151-200
* $T			->> 201-250
* $A			->> 251-300 

##Mote api on init
TODO



# Mqttsn Reference Sheet
TODO

# Screendumps
TODO of twistedRouter, debugs, 
