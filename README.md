# lighttower
Schematics and source code for a light tower that can connect to a monitoring system 

# Background
At some jobs, such as ones that involve support or monitoring, it's important to respond to issues quickly. It may also be important not to let issues linger in case they weren't or couldn't be responded to immediately.

At my past job, I hooked up a light tower to our alerting system and found it very useful. I decided to make a similar setup for when I'm on support at my current job. However, due to computer security policies, some changes were necessary.

# Summary

A server component runs either in the cloud, in a datacenter, or on any computer that has access to the alerting or monitoring software. It will convert information about these issues into sounds of a specific frequency, which it streams via HTTP.

On the local computer connected to the light tower, VLC (or any other sound player, really) plays that audio stream to an analog 3.5mm audio output jack (possibly on a USB sound card for isolation). A 3.5mm analog audio cable ferries that sound to the light tower control board's 3.5mm audio input jack. The control board decodes the frequency and turns on the green, yellow, and/or red lights as appropriate.

# Getting started

## What you'll need
- A computer that can connect to your sound server
- USB sound card (optional, but recommended)
- USB-C power supply & cable (needs to provide at least 1A at 5V; a PC may not be sufficient, but most recent wall warts will be fine, especially the USB-C ones)
- Light tower & some way to mount it. 12v or 24v DC is simiplest
- PCB - I have a few; will need to make more with a different chipset as ATmega328PB is out of stock - switch to RP2040 maybe?
- 3D printed enclosure
- 3.5mm to 3.5mm male to male TRS audio cable. Dollar Tree has the cheapest I found so far, $1.25 + tax

## Hardware Setup
- Open up your light tower.
- Slide on a hex nut, then the L bracket facing down, then a hex nut. Tighten somewhere in the threaded area.
- Mount the light tower
- Use a small flathead screwdriver to connect the light tower wires to the screw terminal:
  - BLACK goes into + (YES, PLUS) - probably the one closest the middle is easiest
  - GREEN goes into 1
  - YELLOW goes into 2
  - RED goes into 3
  - WHITE goes into 4
- Close up the enclosure
- Connect the audio cable between the USB sound card or PC's Speaker or Headphone Out jack and the light tower control board
- Connect the USB-C power cable

## Software setup
### Server
Create a server that outputs the following via streaming audio:
- All off - 1300 Hz
- Green - 1400 Hz
- Yellow - 1500 Hz
- Green and yellow - 1600 Hz
- Red - 1700 Hz
- Red and green - 1800 Hz
- Red and yellow - 1900 Hz
- Red, yellow and green - 2000 Hz

### Client
- Open VLC
- Audio -> Audio Device -> choose the audio device e.g. USB sound card
- Media -> Open Network Stream…
  - Enter the URL for your server
  - Check "Show more options"
  - Set Caching to 100ms
  - Play
- Click the Loop button until Loop One is selected
- turn up the volume on both VLC and in Windows for the selected output device 


# Gory Details
## Board features

- Analog audio - standard 3.5mm TRS audio jack.
- USB-C - with pull down resistors to pull up to 3A, but it doesn't use even half of that. However an unpowered hub might not provide enough. Also has a Silicon Labs CP2102 USB serial converter, used for communication with and reprogramming the Arduino. While regular serial communication works fine, there's a bug on the reset circuit so reprogramming needs a manual step, or use ICSP header.
- Screw terminals - the eight pins are:
`-(GND) +(12/24V) -(GND) +(12/24V) 1(OUT1) 2(OUT2) 3(OUT3) 4(OUT4)`
OUT<N> are open-collector pull down, connected to a ULN2003
The power pins are duplicated so that, if you wanted to, you could attach an external power supply rather than using internal boost circuit, and still have connectors available for the light tower. Purely for convenience.
### Headers
- ICSP - Holding the board with the screw terminals on the bottom and audio & USB connectors on the top, the header is on the top-left and the pinout is:
```
GND - MOSI - +5V
RST - SCK - MISO
```
See e.g. https://docs.arduino.cc/built-in-examples/arduino-isp/ArduinoISP for instructions on how to connect your 5V Arduino to this header and flash bootloader, sketches, etc.
- Serial - Just below that is 5V TTL level serial. Pinout is, from "top" to "bottom" (or left to right if looking at it from the left edge):
GND
RX (data into Arduino) - connect to TX pin on your USB Serial cable
TX (data from Arduino) - connect to RX pin on your USB Serial cable
+5V
- Output header - 5V TTL level outputs for the four control outputs. This is in case you don't want to use internal ULN2003 and want to use an external relay board. This is useful if, for example, your particular light tower runs on AC, or draws more than 500 mA per channel; then you'll need to use an external relay board. Pinout is, again from "top" to "bottom" (or left to right if looking at it from the edge):
GND
Out_1
Out_2
Out_3
Out_4
+5V
These won't drive too much current; they drive the LEDs on the PCB but not much else (direct from the Atmega328PB)
### Jumpers
These adjust the functionality of the onboard TLV61046A boost converter. A jumper present is considered "closed"; no jumper present is considered "open". See datasheet for details.
The lone one, near the big capacitor, selects the output voltage.
Open - uses an internal resistor divider to indicate a request for approx. 24V on the output.
Closed - connects the FB pin to the VIN pin, which indicates a request for 12V on the output
Unfortunately the 24V boost doesn't work quite right, so keep either this one closed (creates 12V) or the enable one (next section) closed 
Two more are next to the screw terminals. The one closer to the edge controls the EN pin.
Open - EN pin is pulled high by an internal pull-up resistor
Closed - connects the EN pin to ground and disables the chip
For the current setup, keep this CLOSED!
The last one is right next to the enable pin, and either connects or disconnects the internal boost converter output to the + outputs of the screw terminals
Open - internal 24V is not connected to the + outputs of the screw terminals. This allows you to use your own power supply, etc.
Closed - internal 24V is connected to the + outputs of the screw terminals.
For the current setup, keep this OPEN!
The pin on the left, closest to the center, is the one actually wired to the + outputs of the screw terminals
### Bugs
This board has 3 known bugs, roughly in order of seriousness
24v boost converter doesn't work
Not sure why, I think I followed the datasheet correctly
Overheats like crazy, draws >1 Watt no load (this is in the spec; 200mA * 5V ~= 1 Watt)
TI designer says this chip will never do 24v at 100mA, not sure why
So either keep disabled, or run at 12v; that seems to work fine
Workaround: for <$1 each you can buy tunable boost converter boards on Amazon that can boost from 5V to 24V. There's enough headers and jumpers to connect it to all the right places:
5V and GND from either the serial or output headers
24V to the jumper connector near the screw terminals, the one closest to the center
24v and GND busses touch at screw terminal
Do not put 24v/GND into + and - and do not connect output connect jumper, without the workaround!
Workaround: cut the trace a little bit (which I did on all 10 PCBs, painstakingly carving out laminate, copper, and PCB material - don't breathe this!)
ATmega328PB Reset line pull-up capacitor
CP2102 runs at 3.3v, ATmega328PB runs at 5v, and that's fine - logic level wise, they understand each other
However, the CP2102's output RTS line maxes out at 3.3v, and capacitor from there to the ATmega328PB's RST line, is pull up by a resistor to 5V
So, this capacitor is always charged to 1.7v
When RTS pin is pulled low, the RST line goes down to 1.7v, not lower because of the charged capacitor
While the ATmega328P resets at 2.1V, the ATmega328PB version resets at 1.6V, so the built in USB serial converter can't trigger the ATmega328PB to go into the bootloader.
Workarounds:
More annoying: manually short C3 at the right moment. Use a small flathead screwdriver, and press hard; there might be laminate over it or something
Easier: Use an Arduino Uno (or any 5V Arduino such as Nano) and flash the ArduinoISP sketch onto it, connect it to the ICSP header, and use that to flash either bootloader and/or sketch: hold Shift to "upload using programmer" in Arduino IDE, or set the right flag on avrdude (not sure) or arduino-cli (-P arduinoasisp)

## Arduino
Computer generates tones of different frequency, Arduino listens for those and sets outputs accordingly

Arduino software uses Goertzel algorithm

ADC prescaler set to 16, ~76.8khz sample rate, good balance between accuracy and speed

Free running mode, runs continuously; interrupt triggers when sample is available, interrupt service routine (ISR) puts that sample into buffer

current buffer size is 256 samples; when buffer is full -> pause interrupt

main loop will see that buffer is full, check the power level on eight specific frequencies, if highest power one is over some threshold, assume that's a valid command and set outputs

reset buffer, resume interrupt

## Enclosure
3D printed, designed in OpenSCAD; see attachments


