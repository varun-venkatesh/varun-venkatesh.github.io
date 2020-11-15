---
layout: posts
title: "Chapter Zero: Upon this rock I shall build my..."
date: 2020-09-20
---

Let's look at what we need in terms of a setup for this exercise, shalle we?

**The operating system you'll need.**  
I have used Linux - specifically Lubuntu 16.04. You can use a linux distribution (distro) of your choice, maybe set up a virtual Linux via [VirtualBox](https://www.virtualbox.org/) if you're running a non-Linux OS on your computer.

**Now, onto QEMU.**  
As I mentioned in the [intro](https://varun-venkatesh.github.io/2020/09/19/bare-mtl-intro.html), you don't need a development board or any external hardware. What you do need is [QEMU](https://www.qemu.org/) which can emulate ARM based harware (along with a plethora of other microprocessors). Using an emulator allows you to develop and test software conveniently - you don't really need external hardware, external tools to download/flash software, external tools to analyse/debug software. QEMU provides all of these capabilities via it's command line. Since we're interested in emulating ARM, we'll install the ```qemu-system-arm``` package. QEMU can emulate a bunch of different boad models of a given processor family - as described [here](https://www.qemu.org/docs/master/system/target-arm.html). For this exercise I have chosen the Stellaris board lm3s6965evb - powered by an ARM Cortex-M3 microprocessor. We will do well to keep the [datasheet](http://www.ti.com/lit/ds/symlink/lm3s6965.pdf) for this board handy.

**Cross-compliation?**  
Yes, like you've guessed correctly, your computer, which is likely powered by x86 or x86_64, cannot build software to run on ARM without the use of a cross-compiler. A cross-compiler is a compiler system that runs on one platform to build executables for another. GNU build tools, including but not limted to GCC, use the concept of target-triplets (architecture-vendor-operating system or bianry interface) to describe the platform. For instance, if you run ```gcc -dumpmachine``` on an x86_64 machine running Linux, you'll get ```x86_64-linux-gnu```. Since we're programming bare-metal, we'll need the ```gcc-arm-none-eabi``` toolchain. Download and install the toolchain from say [here](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads) - pick the version of your choice. What I did was to unpack and install the toolchain onto /opt and then include the compiler directory path in the $PATH environment variable. You'd do well to refer the readme.txt in the packag - gcc-arm-none-eabi-xxx/share/doc/gcc-arm-none-eabi/readme.txt

**Anything else?**  
Other than your favourtite code editor, terminal emulator and web-browser? Nothing. You're good to go.

In the [next chapter](https://varun-venkatesh.github.io/2020/09/28/bare-mtl-chapter2.html), we shall do what every programming tutorial worth it's salt starts off with... A **Hello World** program
