---
layout: posts
title: "Chapter Three: For it is in giving that we receive"
date: 2020-10-19
---  

Previously, we got our C runtime environment which allowed us to start programming in C on a bare-metal system. We went on to get **Hello, World** printed on the console, which was great. It would be nice to have the ability to write what we want to the console or even read from say some hardware registers and display that info on the console. All in all, communication with our system is pretty useful. Enter, the Universal Asynchcronous Receiver Transmitter UART. This peripheral can be read from/written to - and as we commented earlier, the serial port/UART is redirected to the terminal/console (just one of the many uses - you could use the UART to communicate between two systems, for instance).  

Although it is the simplest of the serial communication protocols, let's get familiar with how it works:  

