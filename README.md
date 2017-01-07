# This document: Replacing QTS with xubuntu 16.04 on a TS 453A

This repository is basically a README only (no code). It documents my findings while replacing QTS (QNAP's own OS) with XUbuntu 16.04 on a QNAP TS 453A.

If you find it useful, my pleasure. Your mileage may vary, of course.

If you manage to get parts to work that I didn't, I'd love to hear how you did it. Feel free to PM me or supply a pull request!

### Working parts

* I got Xubuntu running
* The LCD works (on/off & set text)
* Reading the CPU temp works

### Not working parts

* I'm not able to change the fan speed.
* Beeping short/long at will
* Controlling the various LEDs


# How to install Xubuntu

Create a USB device for an OS (I used xubuntu 16.04 64 bit - Xubuntu is a personal preference that seemed to fit well with the idea of using a NAS (not very powerful) as a full PC)

At boot:

* Connect a HDMI device
* On the beep, press F2/Del to get into the BIOS (I never actually found out which of those it was; I just kept pressing both of those keys until the BIOS showed up)

* Some searching for the proper USB device was required (my USB disk showed up as 2 devices, of which only one worked; also, saving the boot order and restarting did not work, but selecting the boot-device straight from the BIOS in the rightmost tab did work)

After selecting the USB, I was able to do a live boot and "proceed as usual".

## QNAP Flash

The QNAP flash is a 500MB piece of "firmware" that contained the QNAP bootstrapping system. I identified it by size (I had no other 500MB devices connected to the system).

It showed up as something available for reformatting while installing ubuntu. (This is actually something I selected, to ensure that QTS would not accidentally resurface)

* I was prompted with the following message and chose "continue in UEFI mode":
    "This machine's firmware has started the installer in UEFI mode but it looks like there may be existing operating systems already installed using 'BIOS compatability mode [..] 

## LCD notes

I managed to get the LCD to work like so:

Set the baud:

    stty -F /dev/ttyS1 1200

And then execute these commands to control the LCD (Switch off, Switch on)

    echo -e "M^\x0" > /dev/ttyS1
    echo -e "M^\x1" > /dev/ttyS1

Set text in line 1, Set text in line 2):

    echo -e "M\f\x0 HELLO WORLD 1234" > /dev/ttyS1
    echo -e "M\f\x1 HELLO WORLD 1234" > /dev/ttyS1

My sources are here:

* https://github.com/bkram/qnapdisplay/blob/master/qnapdisplay/__init__.py
* https://forum.qnap.com/viewtopic.php?t=19180
* https://github.com/jdupl/QnapFreeLCD
* https://github.com/bkram/qnapdisplay

I am not sure what the encoding is, some extended version of ASCII (in any case: one byte, one symbol)

## Fan & Sensors

Next, I tried to get fan control to work. In the process, I also examined further options for controlling the HW, such as the LEDs and Beeps. Until now, all without any success.

First, I tried the "more or less standard" path:

The packages `sensors` and `hddtemp` were already installed. Their outputs are as such:

    # sensors
    acpitz-virtual-0
    Adapter: Virtual device
    temp1:        +26.8 C  (crit = +95.0 C)
    
    coretemp-isa-0000
    Adapter: ISA adapter
    Core 0:       +39.0 C  (high = +90.0 C, crit = +90.0 C)
    Core 1:       +45.0 C  (high = +90.0 C, crit = +90.0 C)
    Core 2:       +41.0 C  (high = +90.0 C, crit = +90.0 C)
    Core 3:       +38.0 C  (high = +90.0 C, crit = +90.0 C)

    # hddtemp /dev/sda /dev/sdb
    /dev/sda: WDC XXXXXXXXXXXXXXXX: 35 C
    /dev/sdb: WDC XXXXXXXXXXXXXXXX: 33 C

Next, I installed fancontrol

    # aptitude install fancontrol

    # pwmconfig 
    # pwmconfig revision 6243 (2014-03-20)
    This program will search your sensors for pulse width modulation (pwm)
    controls, and test each one to see if it controls a fan on
    your motherboard. Note that many motherboards do not have pwm
    circuitry installed, even if your sensor chip supports pwm.

    We will attempt to briefly stop each fan using the pwm controls.
    The program will attempt to restore each fan to full speed
    after testing. However, it is ** very important ** that you
    physically verify that the fans have been to full speed
    after the program has completed.

    /usr/sbin/pwmconfig: There are no pwm-capable sensor modules installed

I ran `sensors-detect` to attempt to find them, but to no avail (I answered YES on all questions)
I agreed to the automatic adding of a line to this to /etc/modules, like so:

    # Chip drivers
    coretemp

I also ran `/etc/init.d/kmod start`, and did a full reboot. Still, no fan control was available.

Source on the net:

* https://wiki.archlinux.org/index.php/Fan_speed_control
* http://askubuntu.com/questions/22108/how-to-control-fan-speed

## Further (unsuccessful) attempts at directly controlling the Fan speed

I found a number of sources on the internet (quoted below); They seem to be mostly for the ARM side of the equation, but I figured by reading and understanding them I might be able to achieve the same for the 453A

Given the success of writing to /dev/tty1 for the LCD, I figured I might be able to achieve similar successes, but didn't.

Some relevant opcodes (at least for the ARM stuff) are:

    command["fan_stop"]       = 0x30
    command["fan_silence"]    = 0x31
    command["fan_low"]        = 0x32
    command["fan_medium"]     = 0x33
    command["fan_high"]       = 0x34
    command["fan_full"]       = 0x35

    command["powerled_off"]   = 0x4b
    command["powerled_blink"] = 0x4c
    command["powerled_on"]    = 0x4d

    command["beep_short"]     = 0x50
    command["beep_long"]      = 0x51


### Examining the QTS source:

I went as far as downloading the entire kernel source, and diffing it with 3.10.20... but nothing jumped out to me, and I'm not enough of a hardware person to proceed.

The QTS kernel can be found here:

https://sourceforge.net/projects/qosgpl/

The file `GPL_TS/src/linux-3.10.20-tas/include/qnap/pic.h` indeed contains the opcodes mentioned above, but they are surrounded by a `ARM`-specific `ifdef`.

### Attempts by experimentation:

_N.B. sending arbitrary opcode to poorly understood interfaces is potentially dangerous_ (since you don't know how the other end will interpret it)

I experimented a bit using the opcodes mentioned above.

In particular, I did stuff like so:

    # stty -F /dev/ttyS1 1200
    # echo -e "\x50" > /dev/ttyS1
    # echo -e "\x51" > /dev/ttyS1

And also on /ttyS0 in an attempt to get the short/long beeps, but this did not work.

## Fan speed: references:

Various refernces I used while attempting to control the fan speed:

* https://github.com/edddeduck/QNAP_Fan_Control/blob/master/FanControl.sh
* https://www.cyrius.com/debian/orion/qnap/ts-409/install/
* https://www.cyrius.com/debian/kirkwood/qnap/ts-219/tips/
* https://packages.debian.org/wheezy/qcontrol
* https://launchpad.net/ubuntu/+source/qcontrol/0.5.4-5
* https://github.com/rubeon/qnappy/blob/master/qcontrol.py
* https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=712283
* https://github.com/groeck/it87/issues/1
* https://lists.debian.org/debian-arm/2014/02/msg00063.html
* http://thread.gmane.org/gmane.linux.kernel/1508763 
* https://github.com/tomtastic/qnap-gpio/ 
* http://www.gossamer-threads.com/lists/linux/kernel/1799465
* https://packages.debian.org/search?keywords=qcontrol
* https://www.hellion.org.uk/blog/posts/qcontrol-support-for-x86-devices/
* https://github.com/groeck/it87/issues/1
* https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=712283
