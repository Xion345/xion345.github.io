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
- A VGA Synchronisation circuit, which generate synchronisation circuits
    required to drive the screen.
- A VGA Tiled Framebuffer which stores video data and transmits to the
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

The red, green and blue signals are analog signals (0-0.7V), which respectively represents
the intensity of red, green and blue. The Digilent Nexys3 board
includes a simple resistor-based DAC (Digital to Analog Converter), so the FPGA
only has to output digital signals for the three colors. The DAC supports 3 bits
for red intensity, 3 bits for green intensity and 2 bits for blue intensity, so
8 bits pixel-depth or 256 representable colors. Red, green and blue signals
evolve over time so that different colors are be displayed on different pixels.

The Hsync (Horizontal Synchronisation) and VSync (Vertical Synchronisation)
tell the screen that the FPGA has finished transmitting a line of
pixels, or a full screen, respectively.

Indeed,

### Obtaining VGA Timing Information ###

## Tiled Framebuffer ##