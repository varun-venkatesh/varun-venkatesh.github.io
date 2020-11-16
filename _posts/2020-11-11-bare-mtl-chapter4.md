---
layout: posts
title: "Chapter Four: Time and tide..."
date: 2020-11-11
---  

Our system is now running software - every bit of which has been written by us - using which we can interact with it (in terms of accepting our keyboard inputs to display the same on the console). We did this by programming the uart device and the interrupt controller.  

The actual hardware comes with a variety of peripherals and interfaces to potentail peripherals that can be controlled by our processor onboard. Things like GPIO, I2C, SSI, SPI, PWM, ADC, ethernet etc. In order to use these interfaces and peripherals or even run useful applications, one of the prerequisites happens to be the ability to keep/measure time (not to mention the fact that this is something that is achievable with the use of an emulator).  

Time keeping on any hardware is achieved with the use of clock and timer hardware available on boad. Programming these bits of hardware and being able to keep time accurately will be the subject of this chapter. If you recall the time when we started programming the uart driver, we had assumed that the system clock frequency was about 12.5 MHz by default and we had enabled the clock signal to drive the uart peripheral. It's now time to throw the assumptions out the window and actually set up a system clock and timer driver. We'll then use these to build a simple scheduler.  

### The clock is ticking  

The hardware lm3s6965 comes with a set of clock sources on board that can be controlled via SFRs (what else?!). These clocks are similar in operation to your mum and dad's quartz wristwatches and are part of the system control. More details on the various clock sources and their properties can be found in section 5.2.4 of the data sheet. Utlimately, the internal system clock (SysClk) which drives the various peripherals and aides in keeping time is derived from one of these clocks on board. Figure 5-5 on the data sheet gives us a good look at how the various clock sources are arranged and their relation with the system clock.  

Controlling the clock and getting a stable accurate system clock frequency is key to operating the peripherals as well as for time keeping. As with other peripherals discussed earlier, we'll need to program the SFRs corresponding to the system control (specifically, clock control). The data sheet has a great follow along for programming the clock control.  

The source code for this section is available [here](https://github.com/varun-venkatesh/bare-metal-arm/tree/master/src/chapter4).

We start off by mirroring the System Control Register Map using a structure like we did with ```UART``` and ```NVIC```.
