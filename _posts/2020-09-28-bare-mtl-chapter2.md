---
layout: posts
title: "Chapter Two: Hello... Is it me you're looking for?"
date: 2020-09-28
---

Now, that you have your setup ready, let's get coding. Remember, this is a bare-metal system - it has no software on-board and you have to do everything.

### Wading through assembly code  

We know that the hardware includes an ARM Cortex-M3 microcontroller, so we should be able to use ARM assembly instructions to get something done for us, say, find the sum of two numbers. That sounds like a good starting point.

```assembly
.syntax unified
.thumb

tos:     .word 0x5000
reset:  .word start

.text
.thumb_func
start:
        ldr r0, num1
        ldr r1, num2
        add r2, r1, r0

stop:
        b stop

num1:   .word 5
num2:   .word 6
```

Let's save this file as ```hello.s``` as we'll build upon this file as we journey towards a Hello, World program.

You can see how we use the load instruction to load numbers into the cpu registers, add them and save the result into another cpu register:


```assembly
  ldr r0, num1
  ldr r1, num2
  add r2, r1, r0
```

There are a few other things in our assembly code that need a bit of context. 
At the very beginning, you see ```.syntax unified``` and ```.thumb``` which are what we call assembly directives. These are "psuedo-opcodes" that direct the assembler to perform operations other than assembly. Assembly involves the assembler operating on the instruction opcodes such as ```ldr``` and ```add``` ,in our case, to produce executable machine code (binary representations of these instructions that can be executed by the processor).


Coming back to assembly directives, ```.syntax unified``` is used to take advantage of a [unified syntax](https://sourceware.org/binutils/docs/as/ARM_002dInstruction_002dSet.html#ARM_002dInstruction_002dSet) for the ARM instruction set. The ARM processor family supports the ARM, Thumb and Thumb-2 instruction sets - depending on the architecture. By using ```.syntax unified```, we are instructing the assembler to read the code that follows as if it were written in Unified Assembler Language [UAL](https://www.keil.com/support/man/docs/armasm/armasm_dom1359731145130.htm). Long story hort, assembly code wrtten in UAL can be assembled into ARM or Thumb instructions - there is no need to write the assembly separately for each instruction set.
Now, it follows that ```.thumb``` directive instructs the assembler to assemble the code that follows into Thumb instructions.


```.text``` marks the beginning of the code/text section, which is where, as you know, resides the code and constants (within the executable). ```.thumb_func``` directive informs the assembler that the function that follows is to be assembled as thumb - I know this sounds superfluous given we've used ```.thumb``` to say the same about all of the code in the file. However, I have seen assemblers generate ARM instructions if this is not explicitly included at the head of every function label within an assembly file.


Now, coming to labels, you can see ```start```, ```stop```, ```num1```, ```num2``` which are used to refere to a location of the instruction within memory. They are used as sort of a readable representation of a memory address. For instance, ```start``` represents the mmeory address where the instruction ```ldr r0, num1``` is located in memory. The labels ```num1``` and ```num2``` represent the memory address where the ```.word```(s) 5 and 6 are stored. The ```.word``` directs the assembler to reserve space in memory to hold the value that follows it. Before we move on, notice how at the label ```stop```, we have ```b stop``` - which is an infinite loop so the program hangs until we turn-off/reset the microprocessor.

There are a couple of labels that we haven't talked about yet, ```tos``` and ```reset```. The ARM Cortex-M3 processor, when reset reads two words from memory at:
1. Address ```0x00000000```: into the stack pointer register - r13
2. Address ```0x00000004```: reset vector (which is copied into the program counter register r15) from where execution begins.

These lables are mark the first two words - with ```tos``` tentatively set to ```0x5000``` representing the top of stack (hence tos) and the reset vector pointing to ```start``` which is where our code begins (and consequently so will the execution of our code). This constitutes what we call the **interrupt vector table** or **vector table**. More on this later.

### Assembly and Linking

Let's assemble this file and then link it to generate an executable - elf.

```arm-none-eabi-as -g -mcpu=cortex-m3 -o part_1.o part_1.s```    
**-g** - generates debug information.  
**-mcpu** - assembles code for a given cpu - in this case, cortex-m3.

```arm-none-eabi-ld -Ttext=0x0 -o part_1.elf part_1.o```    
**-Ttext** - specifies the start address of the text segment. We need to keep it at 0x0 because because Cortex-M3 at startup begins by reading from address 0x0.    

You'll see the linker here ```arm-none-eabi-ld``` complain:    
**arm-none-eabi-ld: warning: cannot find entry symbol _start; defaulting to 0000000000000000**    
because it expects a ```_start``` symbol/label from where execution begins. We'll eventually use ```_start``` to lable the start of execution.   

The executable generated is in the ELF (executable and linkable format) - one that can be loaded and excuted in a Linux or Unix-like environment. However, we're dealing with a bare-metal system with no operating system running whatsoever. This would require us to use a raw binary dump of the elf executable which can be run on our system:

```arm-none-eabi-objcopy -O binary part_1.elf part_1.bin```  
**-O** - generate output of specified format - in this case, binary.

If you notice the output files generated so far - part_1.o, part_1.elf, part_1.bin - the .o and .elf files span between a few hundred to a few kilobytes - owing to their elf format (headers, footers etc.) whereas the .bin file is about 28 bytes representing the actual amount of code necessary to add two numbers and save the result in a register on a cortex-M3 cpu. To run this on our bare-metal system (as simulated by qemu):

```qemu5.1-system-arm -M lm3s6965evb -kernel part_1.bin -nographic -monitor telnet:127.0.0.1:1234,server,nowait```  
**-M**  - specifies the simulated machine on which to run the binary - in this case, lm3s6965 board.  
**-kernel** - the "kernel" or "executable" to run - in binary format.  
**-nographic** - run as a command-line application output everthing on terminal. The serial port is also re-directed to terminal.  
**-monitor** - set-up the QEMU monitor interface to examine the simulated machine running your binary. In this case, it is set up on ```localhost```(127.0.0.1) and port 1234. ```server, nowait``` referes to QEMU setting up a telnet server but not waiting on a connection to run the executable.   

In another terminal window:   

```telnet localhost 1234```

to connect to the telnet server hosted by the qemu machine, you'll see:

```
$ telnet localhost 1234
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
QEMU 5.1.0 monitor - type 'help' for more information
(qemu)
```

Now, to see if our program has run successfully, try examining the registers on the simulated machine:

```
(qemu) info registers
R00=00000005 R01=00000006 R02=0000000b R03=00000000
R04=00000000 R05=00000000 R06=00000000 R07=00000000
R08=00000000 R09=00000000 R10=00000000 R11=00000000
R12=00000000 R13=00005000 R14=ffffffff R15=00000012
XPSR=41000000 -Z-- T priv-thread
FPSCR: 00000000
```   

Notice how ```R00=00000005 R01=00000006``` have our inputs and the result is stored in ```R02=0000000b``` (0x0b == 11) - our program works!  
Take a look at ```R13=00005000 R15=00000012``` - we've seen earlier that ``` R13``` represents the stack pointer and ```R15```, the program counter. The value stored in stack pointer is the value at address ```0x00000000``` labelled ```tos```. The program counter points to the label ```stop``` as we have set up a loop forever by branching to ```stop```.  

Earlier, we mentioned that on reset, the value at address ```0x00000000``` is copied onto the stack pointer ```R13``` and the value at address ```0x00000004``` is copied onto the program counter ```R15``` which is where execution will begin. We've seen the first statement in action. To see how execution begins at the address copied from ```0x00000004```, run the binary on QEMU but halt execution:

```qemu5.1-system-arm -s -S -M lm3s6965evb -kernel part_1.bin -nographic -serial /dev/null```  
**-S** - freeze (or stop) execution at startup.  
**-s** - used instead of -gdb tcp::1234 or running gdbserver in the qemu prompt.  
**-serial** - redirect serial output to a char device - in this case /dev/null.  

open up another terminal and run:

```arm-none-eabi-gdb part_1.elf```  

connect to the gdbserver on ```localhost:1234```  

```
(gdb)target remote localhost:1234
Remote debugging using localhost:1234
start () at part_1.s:10
10	        ldr r0, num1
(gdb) info registers
r0             0x0	0
r1             0x0	0
r2             0x0	0
r3             0x0	0
r4             0x0	0
r5             0x0	0
r6             0x0	0
r7             0x0	0
r8             0x0	0
r9             0x0	0
r10            0x0	0
r11            0x0	0
r12            0x0	0
sp             0x5000	0x5000
lr             0xffffffff	-1
pc             0x8	0x8 <start>
xpsr           0x41000000	1090519040
```  

see how execution has stopped excatly at startup represented by label ```start``` which is also the value in the program counter ```R15```. You may say, how convenient that start of execution in my example is where I begin any assembly instruction at all. Well, if you don't believe me, move ```start``` to any other address or even change the ```reset``` label to point to another address and try examining the results via the debugger!  

We can go on to do much more with assembly instructions this way, say array/string manipulation, searchig, sorting etc. We can even program the system to write to and read from peripherals, respond to interrupts, set up timers and so on. But doing so in assembly language is rather tedious and here's where higher level programming langauge such as C comes into the picture.  

### Getting our C legs:

**What do we need to have running on our system to be able to execute a program whose source is in C?**  
This is what's normally referred to as the C runtime environment. In fact, in a lot of source code for embedded systems, you may have come across files named ```crt0.s``` or ```crt0.c``` - which represents this environment/setup programmed for the particular hardware in assembly or even C (take that chicken and egg!). This runtime configures the hardware appropriately to be able to run the program generated by a C compiler.  

In order to execute a program written in C, our system needs to have the following capabilities:  

1. maintain a region of memory to hold variables with global scope that are initialized - (extern and static in C) - where they can be read/modified - **data segment**
2. maintain a region of memory to hold variables with local scope (restricted to the function/block in which they are defined) - where they can be read/modified. **stack**
3. manage storing/retrieving arguments to functions as well as the return address of the function. - **stack**
4. clear the memory region to hold uninitialized variables with global scope. **bss segment**

You remember these "segments" being referenced in your last interview don't you? Arranged like this?  

![program memory layout]({{ site.baseurl }}/images/program_mem_layout.png "program memory layout")  

This figure above raises the question - where in the memory do these segments reside and how do we instruct the hardware to manage these? For this, we'll need a copy of the data sheet for the hardware of choice, LM3S6965 ([here](https://www.ti.com/lit/ds/symlink/lm3s6965.pdf?ts=1602791587961&ref_url=https%253A%252F%252Fwww.google.com%252F)) and a linker script.  

If we look at the memory map in Table 2-4, addresses in the range ```0x00000000 to 0x0003FFFF``` (256kB) represent the on-chip flash (non-volatile memory) and ```0x20000000 to 0x2000FFFF``` of on-chip SRAM (volatile memory). Since code doesn't have to change at runtime, the ```text``` segment can continue residing in the on-chip flash i.e. the code doesn't have to be relocated to any other memory region (although we could if we wanted to).  

The ```data``` and ```bss``` segments however, will have to be relocated to the volatile memory - these hold variables with global scope which will be modified routinely. The volatile memory region will also hold the stack - which is where local variables, arguments and return addresses will be present and modified. It is fairly easy to configure where the stack - the address held at memory location ```0x00000000``` initializes the stack pointer - in the ARM Cortex-M3. The stack as defined by the ARM Architecture Procedure Call Standard [AAPCS](https://developer.arm.com/documentation/ihi0042/j/?lang=en) requires the stack to be _full-descending_, i.e. the stack "grows" downwards in memory. Let's then set the initial value of the stack pointer to the highest address in the volatile memory space ```0x2000FFFF```. You may be wondering what if the stack continues to grow and overwrite the ```data``` and ```bss``` segments that are also in the volatile memory space? Yes, it is possible and there are ways to detect this. We'll discuss that later.  

Now that we know where each section of the program will reside, we'll need a way to ensure that this is set up before we can run any C program (executable) on our system. How do we get these various sections to be placed where we want them? That's where the linker script comes in. Recall how we linked our object file to get an executable ```.elf``` file, where we specified the start address of the text segment. The linker, ```arm-none-eabi-ld``` can also accept a linker script written in a specialized scripting language specific to the ld application. This script can be used to specify memory regions corresponding to various segments in accordance with the memory map - which the linker understands and complies with.










