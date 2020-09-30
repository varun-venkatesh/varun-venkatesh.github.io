---
layout: posts
title: "Chapter Two: Hello... Is it me you're looking for?"
date: 2020-09-28
---

Now, that you have your setup ready, let's get coding. Remember, this is a bare-metal system - it has no software on-board and you have to do everything.

We know that the hardware includes an ARM Cortex-M3 microcontroller, so we should be able to use ARM assembly instructions to get something done for us, say, find the sum of two numbers. That sounds like a good starting point.

```assembly
.syntax unified
.thumb

sp:     .word 0x5000
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

There are a couple of labels that we haven't talked about yet, ```sp``` and ```reset```. The ARM Cortex-M3 processor, when reset reads two words from memory at:
1. Address 0x00000000: into the stack pointer register - r13
2. Address 0x00000004: reset vector (which is copied into the program counter register r15) from where execution begins.

These lables are mark the first two words - with ```sp``` tentatively set to 0x5000 representing the top of stack and the reset vector pointing to ```start``` which is where our code begins (and consequently so will the execution of our code). More on this later.

