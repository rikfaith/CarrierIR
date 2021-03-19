# CarrierIR

## Introduction

I have a Carrier ductless mini-split system that is controlled using the
RG52F3/BGEFU1 IR remote. This remote is also used on HVAC systems from other
vendors.

Using an oscilloscope attached to the remote's LED, the carrier was measured
at 38.1KHz.  In my code I used 38KHz even, which means the cycle time is about
26.3uS.

## Why I Hate LIRCD

Back in 2018, I tried to decode and replay the IR protocol using an IguanAir
USB adapter. While I made progress on decoding the protocol, I never could
play it back in a format to which the wall unit would respond. This was made
more complicated by lircd getting in the way of things, especially the
determination that a second header needed to be send between the two sets of
codes.  Linux now support /dev/lirc0 without the use of lircd and has tools
that have a more direct interface with the device, such as ir-ctl.

At that time, not being able to get the unit to respond to signals sent from
Linux, I obtained an additional remote from ebay, took it appart, hooked up
relays to the keys, and used that to control the Carrier wall unit. This
worked great, but was an embarassing kludge.

![Relay Board Solution](images/photo_2021-03-19_18-52-46.jpg?raw=true)

The remote was a RG52F3/BGEFU1 obtained via ebay ($23), which runs on 3V
supplied by the SMAKN DC-DC converter ($7 on amazon). The "8 Channel Ethernet
Relay" board is from Numato Lab ($90). 12V is supplied via a PoE adapter.

## Using a Raspberry Pi

Fast forward to 2021, and I hooked up a TSOP38238 Infrared Receiver and a
TSAL6400 Infrared Emitter to a Raspberry Pi, adding the following lines to
/boot/config.txt:
~~~
dtoverlay=gpio-ir,gpio_pin=23
dtoverlay=gpio-ir-tx,gpio_pin=18
~~~

This worked find with commands such as this:
~~~
ir-ctl -d /dev/lirc1 --receive=tmp.txt
~~~

## Protocol Description

The header was measured using ir-ctl and found to be a long 4.4mS pulse
followed by a 4.4mS space.  If we assume 168 cycles, then the time is 4418uS
for each.

A 1 is indicated by a .5mS pulse followed by a 1.6mS space.  If we assume 21
cycles for the pulse and 63 for the space, we have 552uS and 1657uS,
respectively.

A 0 is indicated by a 552uS pulse followed by a 552uS space.

The remote sends, in order:
* a header,
* 6 bytes, with the last space more then 5mS long (the sequence technically
ends with the pulse, as far as I can tell),
* another header,
* the same 6 bytes, but inverted (i.e., 1s are 0s and 0s are 1s), ending with
a timeout (i.e., a very long space).

### Byte 0: Command Type

The first byte defines the remaining fields:
* 0xa1: most normal commands
* 0xa2: a few specialized commands
* 0xa4: commands specific to the "follow me" function

### Byte 1

#### 0xa1

* The lower three bits define the mode (i.e., byte1 & 0x07)
  * 0: cool
  * 1: dry
  * 2: auto
  * 3: heat
  * 4: fan
* The next three bits define the fan speed (i.e., (byte1 & 0x38) >> 3)
  * 1: low
  * 2: med
  * 3: high
  * 4: auto
* The top two bits define the power state (i.e., (byte1 & 0xc0) >> 6)
  * 0: off
  * 2: on
  * 3: sleep

#### 0xa2

This field describes the command:
* 0x01: direct
* 0x02: swing
* 0x08: toggle the display on the wall unit
* 0x09: toggle turbo
* 0x0d: toggle self-clean
* 0x0f: toggle FP (this is a "silent" mode that only works for heat)

### 0xa4

This byte appears to be the same as for a 0xa1 command.

### Byte 2

#### 0xa1

The lower 5 bits define the temperature. I'm using a remote set to Fahrenheit
and an offset of 62 should be added to the value. I.e.,
~~~
(byte2 & 0x1f) + 62
~~~
The range of temperatures supported by the remote is from 62 to 86
degrees. When in "fan mode", this value is set to 92 degrees (i.e., 0x1f).

### 0xa2

This byte is set to 0xff by default.

### 0xa4

This field appears to be the same as for a 0xa1 command.

### Byte 3

#### 0xa1

This byte is set to 0xff by default. Otherwise, it contains the "timer off"
value.

### 0xa2

This byte is set to 0xff by default.

### 0xa4

* 0xff: followme on
* 0x3f: followme off

### Byte 4

### 0xa1

This byte is set to 0xff by default. Otherwise, it contains the "timer on"
value.

### 0xa2

This byte is set to 0xff by default.

### 0xa4

This byte contains the temperature reported by the remote, less 31.

### Byte 5: Checksum

The checksum for all commands, regardless of type, is computed in the same
way:
* The values of the first 5 bytes are reversed in LSB/MSB order and are
summed.
* One is subtracted from the sum.
* The result is inverted (i.e., 1s become 0s and 0s become ones).
* The result is reversed in LSB/MSB order.

This was difficult to figure out and there are several examples on the
internet of how this checksum is computed incorrectly.  For discussion, please
see:
~~~
https://www.eevblog.com/forum/projects/48-bit-(6-byte)-ir-remote-protocol-and-checksum/
https://github.com/kpishere/homie_heatPump/blob/master/src/SenvilleAURA.cpp
~~~
