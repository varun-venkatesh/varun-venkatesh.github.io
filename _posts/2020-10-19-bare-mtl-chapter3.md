---
layout: posts
title: "Chapter Three: For it is in giving that we receive"
date: 2020-10-19
---  

Previously, we got our C runtime environment which allowed us to start programming in C on a bare-metal system. We went on to get **Hello, World** printed on the console, which was great. It would be nice to have the ability to write what we want to the console or even read from say some hardware registers and display that info on the console. All in all, communication with our system is pretty useful. Enter, the Universal Asynchcronous Receiver Transmitter UART. This peripheral can be read from/written to - and as we commented earlier, the serial port/UART is redirected to the terminal/console (just one of the many uses - you could use the UART to communicate between two systems, for instance).  

### UART basics  

Although it is the simplest of the serial communication protocols, let's get familiar with how it works:  

UART communication involves a pair of devices sending data on one wire and recieving data on another. The communication doesn't require the use of a clock or synchronization signal, instead the devices are configured to communicate at the same baudrate (baud referes to a unit of transmission speed equal to the number of times a signal changes state per second - which translates to bits per second in the digital communication universe. The term is originally from the world of telegraphy). The data transmitted is interpreted as packets by either side of the communication - with each packet composed of a start bit, 5-9 data bits, a parity bit and 1-2 stop bits.  


| Start bit | Data bit 1 | Data bit 2 | Data bit 3 | Data bit 4 | Data bit 5 | Data bit 6 | Data bit 7 | Parity bit 6 | Stop bit 7 |  
| :-------: | :--------: | :--------: | :--------: | :---------:| :--------: |:---------: |:---------: |:-----------: |:---------: |  

The two UARTs in communication should use the same packet framing format (along with the same baudrate) to communicate successfully. 

### What are we dealing with?  

The hardware we're using, Stellaris LM3S6965, comes with three fully programmable 16C550-type UARTsperipherals on board. The 16C550 UART refers to an integrated circuit used for serial communication on which the said UART interfaces are modelled. These UARTS come with FIFOs for transmission/reception of data, support a baudrate of upto 3.125Mbps, support IR (infra-red) transmission/reception and so on. Section 12 in the data sheet provides details about this and more.  

Controlling the UART devices is doen through what are termed special function registers that are unique to each of the UART peripherals supported. The aspects of teh UART and it's usage contorlled by these registers include - baud rate, packet format, holding data for transmission/reception, status of the UART, error status, controls etc.  
