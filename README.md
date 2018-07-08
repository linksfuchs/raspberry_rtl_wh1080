
---
# raspberry_rtl_wh1080/3080
---
---
[RaspberryPi](https://www.raspberrypi.org/) / [BananaPi](https://en.wikipedia.org/wiki/Banana_Pi) all-in-one rtl-sdr solution to decode Weather and Datetime RF data signal from  [Fine Offset WH1080 weather stations](http://www.foshk.com/Weather_Professional/WH1080.html), Weather, Datetime and UV/solar radiation RF data from [Fine Offset WH3080 weather stations](http://www.foshk.com/Weather_Professional/WH3080.html), and to read pressure data from  [BMP085](https://www.google.com/search?q=BMP085)/[BMP180](https://www.google.com/search?q=BMP180) barometric sensor.
------------------------------
------------------------------
**2018.07.07: Updated rtl_433 sources.**
This lead to these **Changes:**
- Add support to  **Fine OffsetWH3080** (with UV and solar radiation data). To try to mantain compatibility with your scripts I changed the rtl_433 original "**Fine Offset Electronics WH1080/WH3080 Weather Station**" string back to the old "**Fine Offset WH1080 Weather Station**". Both models uses the same modulation/data scheme so this is not relevant. The UV/lux data from WH3080 creates a new msg_type (2), so you can discriminate them easily.
- Add support to some different model of WH1080/3080 that was not decoded with previous version.
- Add support to battery low detection (**new field "battery" (OK/LOW). Beware, you'll probably need to adjust your scripts!**)
- Add support to UTC timestamp (via -U parameter).
- All of the improvements made to [rtl_433](https://github.com/merbanan/rtl_433) since last source version.  

**NOTE about the WH3080 UV/solar sensor:**  the current rtl_433 'driver' for this station uses a formula to calculate solar data to reflect the same values that you can read on your LCD console. 
That's OK if you want to see the same data on both console and datalogger, but in some case it seems that these original console's values are underestimated (the sensor looks 'blind', solar measured data are too low). In other words they are really wrong. This is a bug of the console itself, not of rtl_433. If you want correct data you can try to use a much better formula: just open in an editor **src/devices/fineoffset_wh1080.c** and go to lines 619-663 for better explanations.

---
------------------------------

**Based on [rtl_433](https://github.com/merbanan/rtl_433), a fantastic tool to turn your [Realtek RTL2832 based DVB dongle](https://www.google.com/search?q=Realtek+RTL2832+based+DVB+dongle&source=lnms&tbm=isch&sa=X&ved=0) into a generic data receiver.**


- The [Raspberry Pi](https://www.raspberrypi.org/) is a tiny and affordable computer with a nice [GPIO](https://en.wikipedia.org/wiki/General-purpose_input/output) interface.
- The [BananaPi](https://en.wikipedia.org/wiki/Banana_Pi) is similar to RaspberryPi. This program has been tested with version M1 (Allwinner A20 - ARM Cortex-A7 dual-core, 1GHz, Mali400MP2 GPU, 1GB DDR3 DRAM).
- The [BMP085](https://www.google.com/search?q=BMP085) is a cheap barometric sensor. It's common but obsolete, and is replaced by the also cheap and pin-to-pin compatible [BMP180](https://www.google.com/search?q=BMP180).
- The [Fine Offset WH1080](http://www.foshk.com/Weather_Professional/WH1080.html) is a relatively low-cost weather station and is also sold rebranded with many names: Watson W-8681, Digitech XC0348 Weather Station, PCE-FWS 20, Elecsa AstroTouch 6975, Froggit WH1080 and many others.
- The [Fine Offset WH3080](http://www.foshk.com/Weather_Professional/WH3080.html) is similar to the WH1080, plus an UV and light (solar radiation) sensor.
 
--
The WH1080/WH3080 weather station is composed by an indoor touchscreen control panel and an outdoor wireless sensors group. The latter sends periodically radio data packets to the indoor console, containing weather measurements about wind speed, wind direction, temperature, humidity and rainfall (and UV/solar for the WH3080). It periodically sends also time signals ([DCF77](https://en.wikipedia.org/wiki/DCF77) and other time signals standards) to keep the console clock perfectly synced with an atomic clock. 
The indoor console itself contains a barometric sensor and another hygro and temperature sensor. 
All of these data are available thru the console's USB port: by connecting a PC and by using some opportune software it's possible to keep track of the weather conditions on your area.


Unfortunately the USB connection is not much reliable in the longtime and it tends to stall every often, giving the need for a console's reset. Furthermore, having a pc connected to the console with a (short) USB cable makes impossible to keep the console handy, so you have to choose if you want to see your console around or if you want to record its data.

A solution is to grab the radio signals sent by the outdoor sensors with some kind of receiver, opportunely decoding and saving them. But there is a disadvantage: because the barometric sensor is enclosed into the indoor console unit and not in the outdoor sensors group, pressure data is NOT available into these radio signals. So we should connect a barometric sensor to this receiver to integrate the missing pressure data.

The RaspberryPi mini computer fits perfectly to this purpose: it's cheap, it has GPIO ports to connect sensors, its low power is more than enough for this task and it will not weigh down your electric bill even with 24/7 service (it's less than 5 watts). The pressure sensor (barometer) it's a cheap BMP085 or BMP180 and its wiring requires only four wires.

To receive my WH1080's data with a RaspberryPi I have first tested by using an [RFM01](https://www.google.com/search?q=RFM01) module: it's a tiny radio data receiver that works fine in normal conditions. Some line of code found on the web, some little wiring, and the ensemble 'Rasp + RFM01 + BMP085' has worked for a season... Or so. 

Unfortunately, with season changes I've found that my WH1080 tends to drift in frequency, probably because of the lack of frequency/temperature compensation on the external sensors' TX: the more the temperature was lowering in winter, the more the signal was drifting and my 'ensemble' was unable to cope with it. So during the most interesting part of winter I was unable to track down data!

Trying to recalibrate frequency on RFM01 was an option, but as soon as summer approaching, the same problem obviously rised again with the same result: no data, need to recalibrate back again...

Furthermore the C code solution used to read data from the RFM01 module was built to write the received data into a temporary file. The file was then read by my Python datalogger script to get the data contained within. Then again the file was  overwritten by the C program with new incoming data, and so on and on (and on....).
This process was happening every 48 seconds (the WH1080/3080 sends its weather data every 48 seconds), so this means that in a year that file was overwritten more than 500,000 times!  
It's way too much for the poor SDcard which is the 'hard disk' of the Rasp (as you know there is a finite number of write-cycles in such a media). 

So after some try I've hacked the C code, so the program was now passing its values 'on-the-fly' to the Python datalogger script instead of writing data to a file. But this approach, summed up to the frequency drift problems, somehow introduced new instability behaviours...

There was the need for another way to receive data by using a different approach: **[rtl-sdr](http://sdr.osmocom.org/trac/wiki/rtl-sdr)**.

There is much documentation on the web about it, and the **[rtl_433](https://github.com/merbanan/rtl_433)** project, which relies on rtl-sdr libraries, is a **perfect** solution for this task. 

By testing I've found that:

- **the rtl-sdr dongle is far more sensible than the RFM01** (even by using its mini-antenna);
- **it's a MUCH more stable solution**: if you build a good datalogger script with a good error management (database errors, malformed json/csv output caused by temporary bad signal or interferences...) it can work flawlessly for weeks and weeks;
- **it's more able to cope with frequency drifting.** No more datalogger down, no more data loss;
- **there's no need anymore to pass data to a datalogger script by using such a 'data file transfer' method.** You can grab **json/csv formatted data** 'on the fly' from the program by piping its output into your (Python?) datalogger script. As a result, the Rasp SDcard is much less stressed and its theoretical life is -of course- much longer.  
- **the reading process takes much less CPU power than by using RFM01** (my rtl_433 process is around 15% on an 'old' Rasp model B);
- **The receiver dongle it's almost plug & play!** No wiring, just insert the dongle in the Rasp USB port (and of course the mini antenna to the dongle :smile: ) 

...But we still have the same problem about lack of pressure data so we still need a BMP085 or compatible sensor, with relative data reading code, and being [rtl_433](https://github.com/merbanan/rtl_433) an hardware-agnostic project, it does not contains itself specific code for the Raspberry (or BananaPi) and its sensors.

**So that's my try to integrate some borrowed-from-the-web code to read BMP085 data to an rtl_433 snapshot to build this all-in-one solution for the WH1080/3080.**

Looking at the code you'll may find that it's not such elegant, but it's a kind of test and it's working fine to me. It's tested with a RaspberryPi model B (also tested with a B+ model), Raspbian Jessie (2015-11-21), a nameless USB DVB-T RTL2832U dongle and a BMP180 sensor. It works OK also on a BananaPi M1 (with a little editing of 1 character, see below), and it compiles and works happily on the Raspberry Pi 2 and 3 too.

I have stripped all of the devices modules from rtl_433 source, leaving only active the **'Fine Offset WH1080/3080 weather station'** one to keep the resources use at minimum, but if you need to re-add modules to support some other device, I think that it should be not so difficult.


So this software can:
----------------

- **Get your WH1080/3080 outdoor weather data:** wind direction and speed, temperature, humidity, rain, UV/solar radiation (WH3080 only) and pressure + internal temperature (from the wired BMP085/BMP180 sensor);
- **Get the exact time and date (DCF77 time system and maybe more) coming from the station**: the WH1080/3080 sends datetime data packets at the start of the even hours. Datetime is broadcasted by a hi-power transmitter located in Germany (DCF77 system) grabbing its time from an atomic clock. By using some scripting you could easily keep the Rasp internal clock to the **exact** time and date without the need of [NTP](https://en.wikipedia.org/wiki/Network_Time_Protocol) or [RTC](https://www.google.com/search?q=raspberry+rtc). No data connection required!
- **Give you a valid json/csv data output for your Python's (or other programming languages) needs;**
- **Give you the flexibility of rtl_433 thanks to its options:** you can optimize data mode, signal, frequency etc. ...


--


Installation instructions (tested on Raspbian Jessie (2015-11-21) for the RaspberryPi, and on Bananian Linux (3.4.104-bananian) for the BananaPi):
--------------------------

Plug the USB dongle into the RaspPi/BananaPi and connect the pressure sensor to the GPIO port (search Google for how to do that on your RaspberryPi / BananaPi model. There are just 4 wires to connect. Just **BEWARE** to power the barometric module by using the **3.3VDC** pin, **NOT** the 5VDC pin).
 
As this work is derived from [rtl_433](https://github.com/merbanan/rtl_433), the same compilation and installation procedures applies, but because of the barometric sensor you need some extra operation:

First of all SPI and I2C on the Rasp must be enabled. Use *sudo raspi-config* and go to the 'Advanced Options' and enable both. Answer 'Yes' to the question about kernel module to be loaded by default, but do not reboot at the moment.

If you have a **BananaPi** you should install its GPIO_BP libraries. Look here:
https://github.com/LeMaker/RPi.GPIO_BP


You should now be ready for the following steps. We assume /home/pi as your home; you can use the folder that you want, but obviously it should be writable by your user (and you should adapt following commands, of course). BananaPi users: I don't remember exactly if some other package is needed. If you find some trouble let me know!

--

```bash
sudo apt-get update

sudo apt-get install build-essential libusb-1.0-0-dev i2c-tools libi2c-dev cmake git

cd /home/pi

git clone git://git.osmocom.org/rtl-sdr.git

cd rtl-sdr

mkdir build

cd build

cmake ../ -DINSTALL_UDEV_RULES=ON -DDETACH_KERNEL_DRIVER=ON

make

sudo make install

sudo ldconfig

sudo reboot
```

--
Now that we have the rtl-sdr base installed we can proceed with **raspberry_rtl_wh1080**:


--
```bash
cd /home/pi

git clone https://github.com/ovrheat/raspberry_rtl_wh1080.git

cd raspberry_rtl_wh1080
```

*Now an important part: you MUST edit the file:*
--------
```bash
~/raspberry_wh_1080/src/devices/fineoffset_wh1080.c
```
find the line (should be #~~108~~   #126) containing:

```c
const unsigned char station_altitude = 10;  // <----- Edit this value entering YOUR station altitude!
```

'10' is my station altitude in meters. You must change this to YOUR station altitude (in meters), otherwise your pressure reading could be incorrect.


Another thing to look for is this line, **especially you BananaPi users**:

```c
char *fileName = "/dev/i2c-1"; //<------- If your Raspberry is an older model and pressure doesn't work, 
	// try changing '1' to '0'. Also change it to '2' if you are using a BananaPi! ("/dev/i2c-2";)**
```

It's self-explaining, I hope. If something doesn't work with pressure and you are sure of your correct BMP085 wiring, then try changing that '/dev/i2c-1' in '/dev/i2c-0' . **BananaPi users MUST change this** from 

**"/dev/i2c-1";**

to 

**"/dev/i2c-2";**


or it will not work!


--

After that, save the file and go back to the root of the source directory:

--

```bash
cd /home/pi/raspberry_rtl_wh1080

mkdir build

cd build

cmake ../

make

sudo make install

```
--
Do not pay too much attention to the various 'Warning' that you'll see, they are not importants.


You are now done and ready to test. Remember to connect the mini antenna to the dongle (..doh!). 

---
Usage:
---------------- 
For the help and to see all of the program's options:
 ```bash
rtl_433 -h
 ```

Let's now go to see some example of practical use. Remember: the sensor group sends its data every 48 seconds, so don't pretend to see data immediately, probably it will take some tens of seconds. Keep this in mind especially at the start of the even hours, because of the 3-4 minutes of silence before the time signal, as explained later.


At first we need to know on what frequency is your WH1080/3080 transmitting. This station comes in (at least) three different TX frequencies models: 868 Mhz, 433 Mhz and 915 Mhz. You should find yours on a label on the back of the indoor console. 
My station sends its data on 868.3 Mhz, so my command line should be:

```bash
rtl_433 -f 868300000
```
***NOTE: the old ' -l 0 ' parameter can now be omitted because rtl_433 now default on this.*

Note that sometime you could have reception problems if you tune to the exact frequency of your station (that's the way rtl-sdr works), and you'll better move off a little from the frequency center. So for my station which is 868.3 Mhz, I'll better tune (for example) to 868.25 Mhz:

```bash
rtl_433 -f 868250000
```

or to 868.35 Mhz:

```bash
rtl_433 -f 868350000
```

Keep this in mind especially in case of WH1080/3080's drifting frequencies: you should find the right value for your station by seeking a little, and it should be fine on all seasons :). A good start point could be to use your USB dongle with some rtl-sdr program (SDRSharp, CubicSDR...) to find your WH1080/3080 frequency as 'seen' by your dongle, then, as said, move off just a little in raspberry_rtl_wh1080.

 
If your station transmits on 433 Mhz you can omit the '-f' part, as rtl_433 defaults to that frequency.


If you need json formatted data output, use -F json parameter:

```bash
rtl_433 -f 868300000 -F json
```


If you need csv formatted data output, use -F csv parameter:

```bash
rtl_433 -f 868300000 -F csv
```




--
The WH1080/WH3080 sends time packets on the start of (most) every even hour: at the minute 59 of the odd hour the station stops sending weather data. After some 3-4 minutes of silence, probably used to sync purpose, the station starts to send time data for around three minutes or so (with the usual 48-seconds cycle between data). Then it returns again to send weather data as usual.


To recognize message type (weather, time or UV/solar) and adapt your data acquisition scripts, you can look at the 'msg_type' field on json/csv output:

**msg_type 0 = weather data**

**msg_type 1 = time data**

**msg_type 2 =UV/solar data (WH3080 only)**


  
 
Just another thing to know: the station often sends double or more data packet in the same second (or so): 

```

2016-02-24 00:26:02 :   Fine Offset WH1080 weather station
        Msg type:        0
        StationID:       0005
        Temperature:     7.3 C
        Humidity:        93 %
        Pressure:        1009.47 hPa
        Wind string:     N
        Wind degrees:    0
        Wind avg speed:  0.00
        Wind gust:       0.00
        Total rainfall:  255.0
        Internal temp.:  18.1 C
2016-02-24 00:26:02 :   Fine Offset WH1080 weather station
        Msg type:        0
        StationID:       0005
        Temperature:     7.3 C
        Humidity:        93 %
        Pressure:        1009.50 hPa
        Wind string:     N
        Wind degrees:    0
        Wind avg speed:  0.00
        Wind gust:       0.00
        Total rainfall:  255.0
        Internal temp.:  18.1 C 
        
```

Notice the slightly different pressure, but all the rest is the same. 

**This is the way the station works.** You could enable some mechanism on your script to ignore the second data packet if having the same datetime string (and eventually the same station id), should this be an unwanted behaviour.

--
For specific usage of rtl_433 (and other relative options) you can look at the [project page](https://github.com/merbanan/rtl_433). Just **don't** bother them with questions related to Raspberries/Bananas and pressure sensors...  :smile:


------------------------------------------------------------------

Notes:
- this is essentially a 'hacked' version of [rtl_433](https://github.com/merbanan/rtl_433); kudos should go to this fantastic tool authors.
- BMP085 code comes from https://www.john.geek.nz/2013/02/update-bosch-bmp085-source-raspberry-pi/ . Kudos!







