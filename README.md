# wolf-pi

## What is it?
A carrier board for the NanoPi Duo 2 supporting Direwolf and other amateur radio applications. 
See http://wiki.friendlyarm.com/wiki/index.php/NanoPi_Duo2 for details on the NanoPi Duo 2. Schematics of the NanoPi Duo 2 and it's 
factory carrier board schematic are in the [docs](./docs) folder.

The NanoPi Duo 2 has a built-in sound card (ADC/DAC), GPIO, WiFi, and Ethernet, making it an attractive product for integration.

This device allows you to use a NanoPi Duo 2 with the Direwolf Project. See https://packet-radio.net/direwolf/

## Implementation
In this application, we wire out the Ethernet (following the design used on the NanoPi IoT carrier board), as that is often preferred 
in a stand-alone application, for connection to a LAN.

The GPIO is used with the built-in functions in Direwolf for /PTT and software indication of Carrier Detect.

A 555-based (selectable) time-out-timer is employed, to prevent a locked-up transmitter or software from holding up the radio channel
with dead carrier. This circuit is based on the one recommended in the Direwolf documentation.

For telemetry applications, support for the ADS1115 ADC converter is included. This is accomplished by using a commonly available 
breakout board from Sparkfun/Adafruit/Universal Solder. The header pins for the breakout solder directly on to the carrier.

Non-populated resistor positions are provided for build-your-own voltage dividers for making single-ended ADC measurements (battery voltages, door switches, etc.).

JP2 is provided to facilitate easy measurement of the battery voltage on ADC AN0.

Support for Dallas 1-wire sensors is provided, with pads available to install a DS18B20 on-board, if desired. Note that the onboard DS18B20 will be measuring the temperature inside the enclosure...

ADC and 1-Wire are wired out to a header connector that is external to the case, for external connections.

The DB9 connector is wired to be compatible with Kantronics KPC3+ cables. Additional solder jumpers are provided to modify some of those pin functions for custom applications.

Additional GPIO (buffered) outputs are also added for future use.

Support for hardware COS (DCD) is provided for future applications.

Ferrite beads are used to reduce RFI/EMI.

The board is designed to be installed in a Hammond 1455K1201 case. Actually, it will fit the 1455J1201 (which is thinner). The -1202 version is less expensive (plastic end panels, vs aluminum ones), if you are going to be replacing the end panels with laser cut ones. End plates are still to be designed.

Alternatively, an enclosure has been designed in FreeCad, with the .stl [files available](./enclosure). Confirmed working for the v1.2 board, printed in PLA. Uses #4-40 hardware.

## Usage
The easiest way to get going is to install an [Armbian](https://armbian.com) image, using their imager.

As of 2026/09/03, it was installing:

*v26.11 rolling for NanoPi Duo2 running Armbian Linux 6.18.48-current-sunxi*

### Boot Configuration Changes
In order to utilize all the features of the implementation, additional configuration of the Armbian environment is required.

The following changes are needed to `/boot/armbianEnv.txt` in order to enable the sound card (codec), 1-wire, and i2c (for the ADS1115).

```
overlays=analog-codec i2c0 usbhost0 usbhost2 usbhost3 w1-gpio
param_w1_pin=PA13
param_w1_pin_int_pullup=0
```

### 1-Wire
With the changes to `/boot/armbianEnv.txt`, the 1-wire subsystem will be available in `/sys/bus/w1/devices`.

If the onboard sensor is installed, it should enumerate with a `28-` prefix.

Follow any of the many sources on the Internet for how to utilize it from there.


### ADC via i2c
With the changes to `/boot/armbianEnv.txt`, the ADS1115 daughter board should be detected. 

Install the i2c tools:

```
sudo apt install i2c-tools -y
```

Make sure the bus is there:

```
ve7fet@nanopiduo2:~$ ls /dev/i2c*
/dev/i2c-0
```

Make sure we can see the ADC at address 0x48:

```
ve7fet@nanopiduo2:~$ sudo i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- 48 -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

Unfortunately, the Armbian kernel doesn't have the driver compiled into it in order to use userspace tools. So, that means one will have to access it directly via i2c.

With a little massaging, the library from https://github.com/sanjuruk/ADS1X15_TLA2024_Linux is capable of accessing the ADC.

Note that the value returned by the "single ended" utility is the ADC count. In order to convert it back to a voltage, you'd need to do something like:

```
#include <cstdio>
#include <unistd.h>
#include "ADS1X15_TLA2024.h"

ADS1115 ads;  /* Use this for the 16-bit version */
// ADS1015 ads;     /* Use this for the 12-bit version */
// TLA2024 ads;

int DEBUG = 1;
float AIN0vout, AIN1vout, AIN2vout, AIN3vout;
float AIN0cal, AIN1cal, AIN2cal, AIN3cal;
float PGA = 4.096; /* PGA is defined by GAIN setting */

// AIN0 Resistors
float R5 = 10000; // 10k ohms
float R6 = 3000;  // 3k ohms
// AIN1 Resistors
float R7 = 1; // dummy, so we don't divide by 0
float R8 = 1; // dummy, so we don't divide by 0
// AIN2 Resistors
float R9 = 1000; // 1kohm
float R10 = 3740; // 3.74kohm
// AIN3 Resistors
float R12 = 1;
float R13 = 1;

// AIN0 Max Input Voltage
//
// VINmax = (VDD + 0.3) x ((R5 + R6)/R6)
// VINmax = (3.6) x ((13000)/3000)
// VINmax = 15.6VDC
//
// AIN2 Max Input Voltage
//
// VINmax = (VDD + 0.3) x ((R9 + R10)/R10)
// VINmax = (3.6) x ((4740)/3740)
// VINmax = 4.56VDC

int main()
{
    int16_t adc[4];
    int8_t i;

    if (DEBUG) {
        printf("Getting single-ended readings from AIN0..3\n");
        printf("ADC Range: +/- 4.096V (1 bit = 2mV/ADS1015, 0.125mV/ADS1115)\n");
        printf("ADC Max Input: VDD + 0.3 = 3.6VDC\n\n");
    }

  // The ADC input range (or gain) can be changed via the following
  // functions, but be careful never to exceed VDD +0.3V max, or to
  // exceed the upper and lower limits if you adjust the input range!
  // Setting these values incorrectly may destroy your ADC!
  //                                                                ADS1015  ADS1115
  //                                                                -------  -------
  // ads.setGain(GAIN_TWOTHIRDS);  // 2/3x gain +/- 6.144V  1 bit = 3mV      0.1875mV (default)
  //
  // We're using VDD = 3.3V, so we will use GAIN_ONE to get the most resolution, being
  // careful to choose our voltage divider resistors!
    ads.setGain(GAIN_ONE);        // 1x gain   +/- 4.096V  1 bit = 2mV      0.125mV
  // ads.setGain(GAIN_TWO);        // 2x gain   +/- 2.048V  1 bit = 1mV      0.0625mV
  // ads.setGain(GAIN_FOUR);       // 4x gain   +/- 1.024V  1 bit = 0.5mV    0.03125mV
  // ads.setGain(GAIN_EIGHT);      // 8x gain   +/- 0.512V  1 bit = 0.25mV   0.015625mV
  // ads.setGain(GAIN_SIXTEEN);    // 16x gain  +/- 0.256V  1 bit = 0.125mV  0.0078125mV

    for (i = 0; i < 4; i++) {
        adc[i] = ads.readADC_SingleEnded(i);
        if (DEBUG) {
            printf("AIN%i: %d\n", i, adc[i]);
        }
    }

    /* Calculate output voltage for AIN0. */
    AIN0vout = (adc[0]*(PGA/32767))*((R5+R6)/R6);
    printf("AIN0vout = %.2f\n", AIN0vout);

    /* Calculate output voltage for AIN1. */
    AIN1vout = (adc[1]*(PGA/32767))*((R7+R8)/R8);
    printf("AIN1vout = %.2f\n", AIN1vout);

    /* Calculate output voltage for AIN2. */
    AIN2vout = (adc[2]*(PGA/32767))*((R9+R10)/R10);
    printf("AIN2vout = %.2f\n", AIN2vout);

    /* Calculate output voltage for AIN3. */
    AIN3vout = (adc[3]*(PGA/32767))*((R12+R13)/R13);
    printf("AIN3vout = %.2f\n", AIN3vout);
```

### Sound Card
You can use `alsamixer` to set the audio levels. Use `sudo alsactl store` to save settings in `/var/lib/alsa/asound.state` (*remember to backup this file*).

If you run Direwolf out of the box, it will likely complain:

```
Audio input device 0 error code -5: Input/output error
```

That is because there is no capture device selected. You can either run this:

```
#mic1 CAPTURE switch (on/off) - this will let you use the microphone
/usr/bin/amixer -c 0 cset numid=18 on
```

Or, load `alsamixer`, and select MIC1 as the capture device.

### GPIO
We use GPIO for the /PTT (signal and LED), and the DCD LED.

Put the following in the `direwolf.conf` for Channel 0 to enable them:

```
MODEM 1200

# Our PTT is on GPIOL11 (Pin 9)
PTT GPIO 363
# Our DCD LED is on GPIOG11 (Pin 11)
DCD GPIO 203
```

Note that this is the deprecated sysfs way of using the GPIOs... but it works, for now.
