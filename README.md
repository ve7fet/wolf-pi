# wolf-pi
A carrier board for the NanoPi Duo 2 supporting Direwolf and other amateur radio applications. 
See http://wiki.friendlyarm.com/wiki/index.php/NanoPi_Duo2 for details on the NanoPi Duo 2. Schematics of the NanoPu Duo 2 and it's 
factory carrier board schematic are in the docs folder.

This device allows you to use a NanoPi Duo 2 with the Direwolf Project. See https://packet-radio.net/direwolf/


The NanoPi Duo 2 has a built-in sound card (ADC/DAC), GPIO, WiFi, and Ethernet, making it an attractive product for integration.

In this application, we wire out the Ethernet (following the design used on the NanoPi IoT carrier board), as that is often preferred 
in a stand-alone application, for connection to a LAN.

The GPIO is used with the built-in functions in Direwolf for /PTT and software indication of Carrier Detect.

A 555-based (selectable) time-out-timer is employed, to prevent a locked-up transmitter or software from holding up the radio channel
with dead carrier. This circuit is based on the one recommended in the Direwolf documentation.

For telemetry applications, support for the ADS1115 ADC converter is included. This is accomplished by using a commonly available 
breakout board from Sparkfun/Adafruit/Universal Solder. The header pins for the breakout solder directly on to the carrier.

Non-populated resistor positions are provided for build-your-own voltage dividers for making single-ended ADC measurements (battery voltages, door switches, etc.).

JP2 is provided to facilitate easy measurement of the battery voltage on ADC A0.

Support for Dallas 1-wire sensors is provided, with pads available to install a DS18S20 on-board, if desired.

ADC and 1-Wire are wired out to a header connector that is external to the case, for external connections.

The DB9 connector is wired to be compatible with Kantronics KPC3+ cables. Additional solder jumpers are provided to modify some of those pin functions for custom applications.

Additional GPIO (buffered) outputs are also added for future use.

Support for hardware COS is provided for future applications.

Ferrite beads are used to reduce RFI/EMI.

The board is designed to be installed in a Hammond 1455K1201 case. End plates are still to be designed.
