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

This post shows how to implement a simple tiled VGA (640x480, 60Hz) framebuffer 
in VHDL which supports 8x16 pixels tiles.

You can see it in action below:

![Photo of Nexys3](/assets/tiled-vga-framebuffer-in-action.jpg) 

I implemented this project on a [Digilent Nexys 3](http://www.digilentinc.com/Products/Detail.cfm?NavPath=2,400,897&Prod=NEXYS3) board. This board is equipped with (among others):
    
- A Xilinx Spartan-6 FPGA (XC6LX16-CS324)
- A *100Mhz* Clock
- A FT232 USB-UART Controller
- An *8-bit* VGA port

You should be able to adapt this project to other boards or FPGA quite easily, 
provided that the same I/O devices are avaible. You will obviously have to change 
FPGA pin assignement, but also adjust clock divisors if yours has a different
frequency.

## Project Overview ##

## Principles ## 

## VGA Synchronisation Circuit ##

## Tile Framebuffer Component ##

### Memory components ###

### State machine ###

{% highlight vhdl %}
entity adder is
    port(
        signal input: in std_logic_vector(1 downto 0);
        signal output: in std_logic_vector(1 downto 0);
end entity;
{% endhighlight %}
