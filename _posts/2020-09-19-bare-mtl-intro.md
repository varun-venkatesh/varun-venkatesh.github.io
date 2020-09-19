---
layout: posts
title: "Bare-Metal programming on ARM Cortex-M. No hardware needed! - Intro"
date: 2020-09-19
---

Bare-Metal programming and no hardware needed?! How cool is that?!
The wonders of modern technology, eh?

So, this "tutorial" is not unlike the many scattered around the internet. I chose to put my experiences into words so that it serves as an exhaustive reference (for me and anyone out there). I will also be citing some of the material I've used (in case anyone is interested).

**What is this bare-metal programming?**  
As the name suggests (kind of), it invloves programming a system - microcontroller and peripherals - which has no software (namely, an operating-system/bootloader) on board. 

**But isn't this re-inventing the wheel, and doing a pretty bad job at that, considering the  commercially available/open-source software out there?**  
Maybe... but as I have found out much later in my career, it has quite a bit of academic value - it certainly helped me understand  things that I had hitherto, taken for granted.
Also, what you see on job-boards as _board-bring up_, _BSP board support packages_, _device driver development_ - involves a fair bit of bare-metal programming (there's your dollars-and-cents angle).

**Why did I choose ARM?**  
Mostly because it's so prevalant in the embedded world and it isn't going away any time soon (of course, there's RISC-V now and that may change - which is why I plan on doing this with RISC-V too).

**What do I mean by "No hardware needed"?**  
I cheated there, well, just a wee bit. You do need a pc/laptop (preferably Linux) and I will be using QEMU to simulate the hardware. But! you certainly don't need to go out, buy a development board and tinker around with it. You can if you want to, but you don't really have to. (Just remember, I haven't tested any of this with actual hardware - which is for much later).

**What do I have to know to start off?**  
Just your way around a Linux machine, some C and assembly programming (not much and you can always use references and search when your stuck - I did that a lot) and not much else really. 

I'd like to go about this in parts (or chapters, as I'll call them) starting with the setup and followed by the actual business of programming the simulated harware.

All references to commands and code snipptes will be ```fomatted like code``` (as is common in  programmer-related literature). Larger code snippets will reside in blocks such as this:

```c
int large_code_snippet(void)
{
    /*larger than a single line of code*/
    char *message = "This is a large code snippet";
    return 0;
}
```

And finally, a note to the eager-beavers out there, all of the source code is available [here](https://github.com/varun-venkatesh/bare-metal-arm).

See you in the [next chapter](https://varun-venkatesh.github.io/2020/09/19/bare-mtl-chapter1.html).
