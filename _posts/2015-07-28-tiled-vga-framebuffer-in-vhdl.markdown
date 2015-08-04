---
layout: post
title:  "Tiled VGA Framebuffer in VHDL"
date:   2015-07-28 16:35:55
tags: vhdl
---

<!--

## Abstract ##

- Tiled VGA Framebuffer
- What does it do ?
- Differences with other VGA Controllers
    - FPGA internal block RAM
    -

## VGA Synchronization ##

- VGA Video interface dates back from the time of cathodic screens
- red, green, blue: ADC Resistor based ADC
- vsync, hsync: New line, new screen

- Figure explanation
    - Pixel rate
    - Hsync, Vsync

- Timing information

- References
    - [Nexys3 Reference Manual](http://www.digilentinc.com/Data/Products/NEXYS3/Nexys3_rm_V2.pdf)

-->
* This will become a table of contents (this text will be scraped).
{:toc}

This post shows how to implement a simple tiled VGA framebuffer in VHDL.
It allows an FPGA to display a picture on a screen, provided that the picture
is composed of repeated patterns, named *tiles*. Tiled framebuffers have much
lower memory requirements than classic pixel framebuffers which makes it possible
to implement them using only the FPGA internal block RAM and without requiring
external memory elements. The tiled frambuffer presented in the post requires
35KiB of memory to display a picture on a 640x480 screen (8 bits per pixel)
where a classic pixel framebuffer would require 307KiB of memory.

You can see it in action below:

[Nyan cat picture]
[Pokemon picture]

I implemented this project on a [Digilent Nexys 3](http://www.digilentinc.com/Products/Detail.cfm?NavPath=2,400,897&Prod=NEXYS3) board.
This board is equipped with (among others):

- A Xilinx Spartan-6 FPGA (XC6LX16-CS324)
- A *100Mhz* Clock
- A FT232 USB-UART Controller
- An *8-bit* VGA port

You should be able to adapt this project to other boards or FPGA quite easily,
provided that the same I/O devices are avaible. You will obviously have to change
FPGA pin assignement, but also adjust clock divisors if yours has a different
frequency.

## Project Overview ##

The full project allows transferring a picture from a computer over USB-UART to
the FPGA, whichs displays it on an external screen.

[Project photo with arrows]

The project is made up of the following elements:

- An UART component to receive and transmit data over the serial line
- An UART "DMA" This component manages a simple protocol layer allowing the
    computer to read and write to the VGA Frambuffer memory by sending commands
    over UART.
- A VGA Synchronisation circuit, which generates synchronisation signals
    required to drive the screen.
- A VGA Tiled Framebuffer which stores video data and transmits it to the
    screen.

Description of the UART and UART "DMA" components is outside the scope of this post,
but I might describe in upcoming posts. By constrast, we will show how the VGA
Synchronisation circuit and the VGA Framebuffer work in combination to display
pictures on the screen.

## VGA Video ##

### VGA Signals and Timings ###

Video information transmitted to the screen via a VGA port is conveyed by five
separate signals:

- Red
- Green
- Blue
- Hsync (Horizontal Synchronisation)
- Vsync (Vertical Synchronisation)

Red, green and blue are analog signals (0-0.7V), which respectively
represent the intensity of red, green and blue. The Digilent Nexys3 board
includes a simple resistor-based Digital to Analog Converter (DAC), so the FPGA
only has to output digital signals for the three colors. The DAC supports 3 bits
for red intensity, 3 bits for green intensity and 2 bits for blue intensity, so
8 bits pixel depth or 256 representable colors. Red, green and blue signals
evolve over time so that different colors are be displayed on different pixels.

The Hsync (Horizontal Synchronisation) and VSync (Vertical Synchronisation)
tell the screen that the FPGA has finished transmitting a line of
pixels, or a full screen, respectively.

Indeed, when the VGA interface was designed, screen were based on Cathode Ray
Tubes (CRT) and video information was displayed by sending three electron beams
to the back of the front surface of the screen. Each pixel on the screen was
composed of three "cells" — one for red, one for green, one for blue. The
red, green and blue signals controlled the amount of electrons sent to each
corresponding cell.

[VGA CRT Picture]

CRT screens shifted right the three electron beams *pixel rate* (or *pixel clock*).

### Obtaining VGA Timing Information ###

This is the easy part, right? Well not quite. VGA timings are standardized and
published by the Video Electronics Standards Association (VESA — you probably have
heard this name before), but you have to pay to obtain these specifications. But you
know what might have happened? Some people might have bought the VESA specification, and
received an nice PDF. And they might have uploaded it to there FTP, because you know,
an FTP is convenient to share documents with your coworkers. And the FTP contents might
be exposed over HTTP to the whole Internet. And content might have been indexed by Google.
Just speculating here, but this kind of stuff could have happened.

Alternatively, there are a number of websites which list common timings and present the
information in an easy-to-use way. Among them, I would strongly recommend
[TinyVGA](http://tinyvga.com/vga-timing). You may also be able to find this information in
your monitor manual.

[Table with 640x480, 800x600, 1024x768]

<!-- GTF, CVT, CVT-RB, DMT -->
Some settings where predefined
Some are computed using formulas

    - DMT (Display Monitor Timings)
    - GTF (Generalized Timing Formula)
    - CVT (Coordinated Video Timings)
    - CVT-RB(Coordinated Video Timings - Reduced Blanking)

[Nvidia - Advanced Timings](http://www.nvidia.com/object/advanced_timings.html)
[OSDev Wiki - Video Signals and Timing](http://wiki.osdev.org/Video_Signals_And_Timing#GTF_Using_resolution_and_refresh_rate)

<!-- EDID -->
<!-- Dire que ça simplifie la configuration de l'ordianteur -->
Fairly recent monitors are able to transmit the list of VGA timings they support to
the host graphics adapter via a standard called EDID (Extended Display Identification Data).
We cannot read EDID from the FPGA because it is transmitting via pin 9 (DDC) of the
VGA connector, which is unconnected on the Digilent Nexys3 board. Besides, EDID packets
are somehow tedious to parse. However, it is possible to read your monitor EDID information
from your computer. This is probably the easiest way to retrieve the list of
supported VGA timings your monitor supports. Under Linux/Xorg, you can use `xrandr --verbose`
to do so :

    $ xrandr --verbose # Dell 22" Monitor
    [...]
    1920x1080 (0x67)  148.5MHz +HSync +VSync *current +preferred
            h: width  1920 start 2008 end 2052 total 2200 skew    0 clock   67.5KHz
            v: height 1080 start 1084 end 1089 total 1125           clock   60.0Hz
    1280x1024 (0x68)  135.0MHz +HSync +VSync
            h: width  1280 start 1296 end 1440 total 1688 skew    0 clock   80.0KHz
            v: height 1024 start 1025 end 1028 total 1066           clock   75.0Hz
    1280x1024 (0x69)  108.0MHz +HSync +VSync
            h: width  1280 start 1328 end 1440 total 1688 skew    0 clock   64.0KHz
            v: height 1024 start 1025 end 1028 total 1066           clock   60.0Hz
    1152x864 (0x6a)  108.0MHz +HSync +VSync
            h: width  1152 start 1216 end 1344 total 1600 skew    0 clock   67.5KHz
            v: height  864 start  865 end  868 total  900           clock   75.0Hz
    1024x768 (0x6b)   78.8MHz +HSync +VSync
            h: width  1024 start 1040 end 1136 total 1312 skew    0 clock   60.1KHz
            v: height  768 start  769 end  772 total  800           clock   75.1Hz
    1024x768 (0x6c)   65.0MHz -HSync -VSync
            h: width  1024 start 1048 end 1184 total 1344 skew    0 clock   48.4KHz
            v: height  768 start  771 end  777 total  806           clock   60.0Hz
    800x600 (0x6d)   49.5MHz +HSync +VSync
            h: width   800 start  816 end  896 total 1056 skew    0 clock   46.9KHz
            v: height  600 start  601 end  604 total  625           clock   75.0Hz
    800x600 (0x6e)   40.0MHz +HSync +VSync
            h: width   800 start  840 end  968 total 1056 skew    0 clock   37.9KHz
            v: height  600 start  601 end  605 total  628           clock   60.3Hz
    640x480 (0x6f)   31.5MHz -HSync -VSync
            h: width   640 start  656 end  720 total  840 skew    0 clock   37.5KHz
            v: height  480 start  481 end  484 total  500           clock   75.0Hz
    640x480 (0x70)   25.2MHz -HSync -VSync
            h: width   640 start  656 end  752 total  800 skew    0 clock   31.5KHz
            v: height  480 start  490 end  492 total  525           clock   60.0Hz
    720x400 (0x71)   28.3MHz -HSync +VSync
            h: width   720 start  738 end  846 total  900 skew    0 clock   31.5KHz
            v: height  400 start  412 end  414 total  449           clock   70.1Hz

### Graphics Controller Design ###

## Tiled Framebuffer ##

[Image divided into tiles]

### Principles ###

### Memory Elements ###

### Design Description and Use ###