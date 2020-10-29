---
layout: posts
title: "Chapter Four: "
date: 2020-10-29
---  

So far, we've got software running on the system that allows us to use a console to input and output data. While this is all fine and dandy, our underlying uart device driver used to read from the console does so by polling. This involves the driver code checking if the recieve buffer is not empty every time it wants to read. This scheme is not a very efficient use of system resources - particularly when the system in question supports interrupts.  

### The whys and hows  

An interrupt is a alert -from the hardware - that needs attention from the software. For instance, when we type anything on the console, the processor is alerted of this event so that it can be handled in a timely manner. This may require the current flow of control/execution to be interrupted - hence the name. Since they can occur at any time, interrupts (at least hardware interrupts) are asynchronous.  How the interrupts are handled depends on interrupt handling/exception handling model of the processor.  

Let's go back to chapter 2 where we identified that the first 240 bytes of code represent the **interrupt vector table** with each entry pointing to an interrupt handler or interrupt service routine - the software fucntion that handles a particular interrupt. 
