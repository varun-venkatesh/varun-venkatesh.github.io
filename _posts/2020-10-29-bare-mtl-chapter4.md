---
layout: posts
title: "Chapter Four: "
date: 2020-10-29
---  

So far, we've got software running on the system that allows us to use a console to input and output data. While this is all fine and dandy, our underlying uart device driver used to read from the console does so by polling. This involves the driver code checking if the recieve buffer is not empty every time it wants to read. This scheme is not a very efficient use of system resources - particularly when the system in question supports interrupts.  

### The whys and hows  

