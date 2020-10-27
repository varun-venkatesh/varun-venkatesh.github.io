---
layout: posts
title: "Chapter Three: For it is in giving that we receive"
date: 2020-10-19
---  

Previously, we got our C runtime environment which allowed us to start programming in C on a bare-metal system. We went on to get **Hello, World** printed on the console, which was great. It would be nice to have the ability to write what we want to the console or even read from say some hardware registers and display that info on the console. All in all, communication with our system is pretty useful. Enter, the Universal Asynchcronous Receiver Transmitter UART. This peripheral can be read from/written to - and as we commented earlier, the serial port/UART is redirected to the terminal/console (just one of the many uses - you could use the UART to communicate between two systems, for instance).  

### UART basics  

Although it is the simplest of the serial communication protocols, let's get familiar with how it works:  

UART communication involves a pair of devices sending data on one wire and recieving data on another. The communication doesn't require the use of a clock or synchronization signal, instead the devices are configured to communicate at the same baudrate (baud referes to a unit of transmission speed equal to the number of times a signal changes state per second - which translates to bits per second in the digital communication universe. The term is originally from the world of telegraphy). The data transmitted is interpreted as packets by either side of the communication - with each packet composed of a start bit, 5-9 data bits, a parity bit and 1-2 stop bits.  

![uart_frame]({{ site.baseurl }}/images/uart_frame.png "uart frame")  

The two UARTs in communication should use the same packet framing format (along with the same baudrate) to communicate successfully. 

### What are we dealing with?  

The hardware we're using, Stellaris LM3S6965, comes with three fully programmable 16C550-type UARTsperipherals on board. The 16C550 UART refers to an integrated circuit used for serial communication on which the said UART interfaces are modelled. These UARTS come with FIFOs for transmission/reception of data, support a baudrate of upto 3.125Mbps, support IR (infra-red) transmission/reception and so on. Section 12 in the data sheet provides details about this and more.  

Controlling the UART devices is doen through what are termed special function registers (SFR) that are unique to each of the UART peripherals supported. The aspects of the UART and it's usage contorlled by these registers include - baud rate, packet format, holding data for transmission/reception, status of the UART, error status, controls etc.  

Of these, the ones that we'll be using here are:

**UARTDATA**: The register that holds data to be sent/received. Although there is are 16 FIFOs each for sending and recieving data, we'll start by using this register exclusively lfor all our communication.  
**UARTRSR/UARTECR**: The receive status register/error clear register - records any error in communication. The registers contents can be cleared.  
**UARTFR**: This is the flag register that records the status of the UART FIFOs and indicates wheteher UART is busy.  
**UARTIBRD and UARTFBRD**: These registers are used to set the UART baud-rate.  
**UARTLCRH**: This register, called the line control register is used to control parameters such as data length, parity, and stop bit selection.  
**UARTCTL**: This is the UART control register used to enable/disable the various capabilities of the UART device.  

### What do we want out of the UART?  

As discussed in the beginning, we want to be able to send and recieved data via the UART device. The device in communication with us will be the console represented by the standard input and standard output in the current example. Although, we'd want to write software that can operate the UART to communicate with any UART capable device - in other words we will program a device driver. The software that we write to communicate with the console using the driver APIs will be our application. So, at the end of this we'll have harware -> hardware startup and initialization -> device driver -> application software. See how the various software layers are being formed as we move along.  

### Devil in the details:  

In it's most basic form, a device driver will have the following capabilities:  

1. Initialization: starts up the UART device.  
2. Configuration: configures the UART device in accordance with the agreed upon transmission rate and packet format.  
3. Runtime functionality: the send and recieve functionality during UART operation as well as error detection and handling during device operation.  
4. Termination: stop/disables/turns-off the device. This bit is usually omitted.  

Now, let's start with the initialization and configuration capabilities - and translate them into functions. The data sheet has a great follow along example if you refer to section 12.4. Notice how there are references to setting bits in the SFRs referenced earlier. How do we do this programatically?  





