---
layout: post
title:  "Unlocking a Linksys Velop Router"
subtitle: "Debranding the MX4200"
date: 2025-07-11 20:00:00
background: '/assets/images/2025/07/11/linksys_velop.jpg'
---

# Background

A couple of years back I changed my Fibre provider and received a new [Linksys Velop MX4200 router](https://support.linksys.com/kb/article/952-en/) as part of my contract. This worked *just fine* unless I wanted to do some administration on my network, in which case I could either use the "fairly terrible" web interface or the mania-inducing, eyeball-gougingly awful Linksys Android app (I'm not sure if the iOS app is any better). Every action seems to take an age to complete (if it completes at all) and it just defies logic as to how a company which was at one time valued at $500M can release such crap. I lived with it for a while, but once I started adding nodes to my mesh and wanted to troubleshoot some wireless issues, it was clear I couldn't achieve this with the Linksys administration tools.

{% include image-link.html image_path="/assets/images/2025/07/11/velop.jpg" alt_text="The Linksys Velop MX4200" link="https://support.linksys.com/kb/article/952-en/" caption="The Linksys Velop MX4200" width="35%" %}

I'd used [OpenWRT](https://openwrt.org/) for other routers in the past and I was pleased to see that it had been ported to the MX4200. OpenWRT is a Linux-based embedded operating system which is highly customisable and provides full package management and baked in remote administration for more technically-minded users. Not only that, but it is excellent for diagnostics. I decided to flash all my nodes with the latest stable version (24.10.1) and all my troubles would go away...

# Upgrades

Now the first MX4200 I tried was simple. You can download official firmware files from the [product page](https://support.linksys.com/kb/article/952-en/) and then use the existing web administration tool (at https://192.168.1.1) to upgrade the firmware. You can do this by logging in, clicking `Connectivity` under `Router Settings` and then simply use the `Choose File` button to select the firmware file you wish to flash. Great. This was going to be a breeze.


{% include image-link.html image_path="/assets/images/2025/07/11/linksys_firmware_update.png" alt_text="The Linksys firmware update screen" link="#" caption="Simples?" width="75%" %}

Flushed with success, I then tried the OpenWRT firmware which is available from the [MX4200 product page on the OpenWRT site](https://openwrt.org/toh/linksys/mx4200_v1_and_v2). This also worked just fine and I was excited to find some nifty graphs regarding wireless signal strength.

{% include image-link.html image_path="/assets/images/2025/07/11/wireless_graphs.png" alt_text="OpenWRT's Wireless status page" link="#" caption="Who doesn't love a graph?" width="75%" %}

Then I tried to apply the same steps to the second router, after all, it was the exact same hardware and software as the first, right? Well, as it turns out that was wrong. The other two routers were "ISP branded" which means you can't flash regular or "retail" Linksys firmware and certainly not custom firmware (like OpenWRT).

As a side note, this is just plain wrong. Off-the-shelf consumer routers don't have the best track record when it comes to security, but at the very least they should have a secure upgrade path, so when security issues are identified they can be patched. The best version of this is automatic upgrades, although even this [can go wrong on occasion](https://www.pcmag.com/news/hp-races-to-fix-faulty-firmware-update-that-bricked-printers). If that isn't available, then the manual route should be available to end users to benefit from any firmware updates.

I tried to update the device by flashing an official Linksys firmware (legit surely) and was greeted with this pop up:

{% include image-link.html image_path="/assets/images/2025/07/11/invalid_firmware_file.png" alt_text="Invalid firmware file popup" link="#" caption="Computer says no" width="75%" %}

Wtf? I have automatic updates turned on and I can tell you that the ISP provided firmware hadn't been updated since I first switched the thing on a couple of years back. That's not good.

So what does any experienced security professional do in the face of adversity? Search for quick and easy solutions that someone else has discovered of course!

# Searching it up

Now this router is a few years old so I found a bunch of posts on Reddit about firmware including the [original announcement](https://www.reddit.com/r/LinksysVelop/comments/1903t1j/linksys_mx4200_openwrt_official/) that support for the MX4200 v1 and v2 had been added to OpenWRT.

There were also a few posts commenting that you could upgrade via the `/fwupgrade.html` page, but when I tried that I got the following JSON response:

```
{
"result": "ErrorCannotInstallRetailImage"
} 
```

One chap did come along with [a solution to the locked ISP firmware](https://github.com/ishi0/Community-Fibre-WHW03CFv2/wiki) early on. If you went to the hidden 'CA settings' menu you could apply a retail firmware to de-brand the device, and then you'd be able to upgrade to OpenWRT. Unfortunately, for me and others, this didn't appear to work for my ISP firmware in 2025. One post in particular summed it up nicely:

{% include image-link.html image_path="/assets/images/2025/07/11/sadly.png" alt_text="Reddit post saying the old method doesn't work any more" link="#" caption="I'm in complete agreement" width="75%" %}

What we have here is a genuine requirement! I have a bit of a background in poking around with embedded devices, so I thought I could take a quick look.

Most people just use embedded devices without much thought to how they work and what to do when they go wrong (beyond contacting vendor support). That said, if you've bought a second-hand router off eBay and somewhere down the line it isn't working, you can either <s>bin it</s> recycle it or pop off the case and see if there are any hardware debugging interfaces available to play with.

After 20 minutes of trying to work out where the join was on this case so I could take it apart without breaking it, I discovered a [nice YouTube video](https://www.youtube.com/watch?v=JELVEofJwBg) where someone demonstrates the correct technique. A little while after this I had the Printed Circuit Board (PCB) exposed and could see a number of populated and unpopulated headers (points to connect to on the board).

On a new device I'd start looking for different types of pins and workout what voltage the board is at before probing various parts with a multimeter or mini oscilloscope to work out if any pins could have digital data across them. However, in this case I didn't have to, as the OpenWRT page for the device provided details on a usable [UART debug interface](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter). The page also has photos and diagrams illustrating which of the UART pins in the image below were ground, power, receive and transmit.

{% include image-link.html image_path="/assets/images/2025/07/11/mx4200v1-serial.jpeg" alt_text="UART header pins" link="#" caption="So convenient (Copyright: openwrt.org)" width="75%" %}

The simplest way to make use of these pins is to buy a cheap UART-to-serial USB adaptor [like this one](https://www.amazon.co.uk/dp/B07TXVRQ7V).


{% include image-link.html image_path="/assets/images/2025/07/11/uart-adaptor.jpg" alt_text="UART-to-serial USB adaptor" link="#" caption="A simple way to connect to UART interfaces" width="50%" %}

On the adaptor, note the 6 labelled pins opposite the USB male end. These can be easily connected to using female-to-female push fit connectors (known as [Dupont cables](https://www.amazon.co.uk/Dupont-Female-Solderless-Breadboard-Connectors/dp/B0BLZC3SQ2)). First you can use a cable to link the ground pins on the PCB and your adaptor. You then connect the TX (transmit) pin from the MX4200 to the RX (receive) pin on your adaptor. Finally you connect the RX pin from the MX4200 to the TX pin on your adaptor. In this case you can ignore the VCC (power) pin as the MX4200 PCB will provide the power itself. With these 3 pins connected you can plug the USB end of the adaptor into your computer.

On your PC (assuming you are using Linux - search around for doing this on Windows), you'll find that a new device `/dev/ttyUSB0` has been registered. You can use a terminal emulator program like screen or minicom to connect to the MX4200 via this adaptor. I prefer minicom so I installed this and ran it like so:

```
sudo apt install minicom
sudo minicom -D /dev/ttyUSB0
```

With UART there are various settings you can tweak for different hardware, but in this case the default settings should suffice. For those that are interested that's a baud rate of 115200 bps, 8 data bits, no parity and 1 stop bit (aka 8N1).

Once you've set this up, power on the MX4200 and you should see a load of log prints, first from the bootloader and then from Linux, including this rather snazzy, ASCII art banner:

```
[    3.259347] VFS: Mounted root (squashfs filesystem) readonly on device 253:0.
[    3.260217] Freeing unused kernel memory: 264K (808dc000 - 8091e000)
*********************************************************************************
              _        _  __    _    __ _____ __   __ _____
             | |      | ||  \  | |  / // ____]\ \ / // ____]TM
             | |      | ||   \ | | / /| (___   \ V /| (____
             | |      | || |\ \| |\ \  \____ \  \ /  \____ \
             | |_____ | || | \   | \ \  ____) | | |   ____) |
             |_______||_||_|  \__|  \_\[____ /  |_|  [_____/

 (c) 2013 Belkin International, Inc. and/or its affiliates. All rights reserved.
 Booting chiron (firmware version 1.0.11.215720)
*********************************************************************************
[utopia][init] System Initialization
[utopia][init] Creating /proc
[utopia][init] Creating /sys
```

Once the log output has slowed down somewhat, hit enter a couple of times and you should drop into a login shell. Use the username 'admin', the same password you set for the web admin interface and you'll be presented with a shell prompt. Type `su` and voila we have a root shell to poke around in.

```
$ su
# 
```

I spent some time trying to upgrade the firmware of the branded device via the web GUI (and being presented with the error popup) so that I could examine the logs. Sure enough I was able to spot where the problem was occurring. Here are the relevant log entries:

```
[up.sh] update /tmp/var/config/jcgi.tmp
[fw.sh] verify_linksys_header
[fw.sh] SKU verification Error device(MX42CF), image(MX42)
```

Ah. We have a mismatch between what the device reports itself to be (an MX42 series with a "Community Fibre" tag) and the image we are trying to flash (a plain 'ol MX42 image).

From this point it was a quick grep of the filesystem to find the offending script. The log entries are generated by `/usr/sbin/update` which is a bash script which contains this comment:

```
# ISP SKU device can not be updated to retail firmware, 
# but retail SKU device can be updated to whatever.
```

As we all know, only a Sith deals in absolutes, so I'm going to ignore the words "can not". Further examination of the script shows a simple string compare which, on failure (i.e. the strings don't match), emits the dreaded "SKU verification Error".

Fortunately, the script gives us some clues as to how we can overcome this. We have a root shell so we can adjust certain syscfg values. In this case, all we need to do is:

```
# syscfg set device::model_base MX42
# syscfg set fwup_softsku_name MX42
```

Ta-da. Now the check will pass and we can install retail images again. Using the instructions linked above we can flash stock Linksys firmware and then using the 'CA settings' (casupport) trick, we can manually upgrade the device to the latest OpenWRT image. Phew!

As is the point of this blog, I hope this either helps you with your own de-branding project or you found it somewhat interesting. If you've found an easier way to do this drop me an email (I don't 'do' Twitter/X/MechaHitler) or alternatively post something in the [Linksys Velop Reddit thread](https://www.reddit.com/r/LinksysVelop). I'm sure your effort will be much appreciated.
