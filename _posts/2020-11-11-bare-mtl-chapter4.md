---
layout: posts
title: "Chapter Four: Time and tide..."
date: 2020-11-11
---  

Our system is now running software - every bit of which has been written by us - using which we can interact with it (in terms of accepting our keyboard inputs to display the same on the console). We did this by programming the uart device and the interrupt controller.  

The actual hardware comes with a variety of peripherals and interfaces to potentail peripherals that can be controlled by our processor onboard. Things like GPIO, I2C, SSI, SPI, PWM, ADC, ethernet etc. In order to use these interfaces and peripherals or even run useful applications, one of the prerequisites happens to be the ability to keep/measure time (not to mention the fact that this is something that is achievable with the use of an emulator).  

Time keeping on any hardware is achieved with the use of clock and timer hardware available on boad. Programming these bits of hardware and being able to keep time accurately will be the subject of this chapter. If you recall the time when we started programming the uart driver, we had assumed that the system clock frequency was about 12.5 MHz by default and we had enabled the clock signal to drive the uart peripheral. It's now time to throw the assumptions out the window and actually set up a system clock and timer driver. We'll then use these to build a simple scheduler.  

### The clock is ticking  
