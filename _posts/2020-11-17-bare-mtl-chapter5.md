---
layout: posts
title: "Chapter Five: ... Walk and chew gum at the same time"
date: 2020-11-17
---  

Apart from drivign the uart and demonstarting to us that we've configured a usable clock and timer, the timer constructed can also be used to set up a simple scheduler.

### The basics  

What is the need for a scheduler?  
A scheduler is a program tha allows us the ability to run **multiple quasi-independent** programs on our system - in a seemingly simultaneous manner. Now, that's a mouthful!  

Why would we need to run these **multiple quasi-independent** programs at the same time? Can't we have one single program do everthing we need - like what we have now?  

