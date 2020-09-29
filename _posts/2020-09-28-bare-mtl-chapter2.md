---
layout: posts
title: "Chapter Two: Hello... Is it me you're looking for?"
date: 2020-09-28
---

Now, that you have your setup ready, let's get coding. Remember, this is a bare-metal system - it has no software on-board and you have to do everything.

We know that the hardware includes an ARM Cortex-M3 microcontroller, so we should be able to use ARM assembly instructions to get something done for us, say, find the sum of two numbers. That sounds like a good starting point.

```assembly
.text

num1: .word 5
num1: .word 6

start:
  mov r0, num1
  mov r1, num2
  add r2, r1, r0

stop:
  b stop
  
```
