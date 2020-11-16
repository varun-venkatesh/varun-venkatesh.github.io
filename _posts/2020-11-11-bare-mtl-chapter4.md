---
layout: posts
title: "Chapter Four: Time and tide..."
date: 2020-11-11
---  

Our system is now running software - every bit of which has been written by us - using which we can interact with it (in terms of accepting our keyboard inputs to display the same on the console). We did this by programming the uart device and the interrupt controller.  

The actual hardware comes with a variety of peripherals and interfaces to potentail peripherals that can be controlled by our processor onboard. Things like GPIO, I2C, SSI, SPI, PWM, ADC, ethernet etc. In order to use these interfaces and peripherals or even run useful applications, one of the prerequisites happens to be the ability to keep/measure time (not to mention the fact that this is something that is achievable with the use of an emulator).  

Time keeping on any hardware is achieved with the use of clock and timer hardware available on boad. Programming these bits of hardware and being able to keep time accurately will be the subject of this chapter. If you recall the time when we started programming the uart driver, we had assumed that the system clock frequency was about 12.5 MHz by default and we had enabled the clock signal to drive the uart peripheral. It's now time to throw the assumptions out the window and actually set up a system clock and timer driver. We'll then use these to build a simple scheduler.  

### The clock is ticking  

The hardware lm3s6965 comes with a set of clock sources (crystal - also called xtal - oscillators) on board that can be controlled via SFRs (what else?!). These clocks are similar in operation to your mum and dad's quartz wristwatches (yes, in the age of the smartwatch, quartz stays to mum and dad) and are part of the system control. More details on the various clock sources and their properties can be found in section 5.2.4 of the data sheet. Utlimately, the internal system clock (SysClk) which drives the various peripherals and aides in keeping time is derived from one of these clocks on board. Figure 5-5 on the data sheet gives us a good look at how the various clock sources are arranged and their relation with the system clock.  

Controlling the clock and getting a stable accurate system clock frequency is key to operating the peripherals as well as for time keeping. As with other peripherals discussed earlier, we'll need to program the SFRs corresponding to the system control (specifically, clock control). The data sheet has a great follow along for programming the clock control.  

The source code for this section is available [here](https://github.com/varun-venkatesh/bare-metal-arm/tree/master/src/chapter4).

[//]: # (This may be the most platform independent comment)
[//]: # (TODO: Describe the registers and setup the premise for programming them.)

We start off by mirroring the System Control Register Map using a structure like we did with ```UART``` and ```NVIC```. Programming the system control registers to configure a PLL based clock system of a ceratin frequency involves the use of the ```RCC``` register or the ```RCC2``` register if we need a larger assortment of clock configuration options. The configuration function looks like this.  

```C
/* Now we come to the business end of this module - set the system clock.
 * Refer: http://www.ti.com/lit/ds/symlink/lm3s6965.pdf 
 * Section 5.3 for how to go about.
 */
void sysctl_setclk(uint32_t cfg_rcc, uint32_t cfg_rcc2)
{
    uint32_t tmp_rcc, tmp_rcc2;
    uint32_t osc_delay = SYSCTL_FAST_OSCDELAY;

    // Get the current values of RCC and RCC2
    tmp_rcc = sysctl->RCC;
    tmp_rcc2 = sysctl->RCC2;
    

    // set BYPASS (bypass PLL) and clear USESYSDIV
    tmp_rcc |= SYSCTL_RCC_BYPASS;
    tmp_rcc &= ~(SYSCTL_RCC_USESYSDIV);
    tmp_rcc2 |= SYSCTL_RCC2_BYPASS2;

    sysctl->RCC = tmp_rcc;
    sysctl->RCC2 = tmp_rcc2;

    // check if MOSC or IOSC need turning on.
    if(((tmp_rcc & SYSCTL_RCC_IOSCDIS) != 0 && (cfg_rcc & SYSCTL_RCC_IOSCDIS) == 0) ||
       ((tmp_rcc & SYSCTL_RCC_MOSCDIS) != 0 && (cfg_rcc & SYSCTL_RCC_MOSCDIS) == 0))
    {
        // turn on the required oscillators. Don't turn any off for now
        tmp_rcc &= (~(SYSCTL_RCC_IOSCDIS | SYSCTL_RCC_MOSCDIS) |
                    (cfg_rcc & (SYSCTL_RCC_IOSCDIS | SYSCTL_RCC_MOSCDIS)));

        sysctl->RCC = tmp_rcc;

        /* Wait for the newly set oscillator(s) to settle.
         * This wait time would depend on 
         * the previous clock setting.
         */
        if((tmp_rcc2 & SYSCTL_RCC2_USERCC2))
        {
            if(((tmp_rcc2 & SYSCTL_RCC2_OSCSRC2_MASK) == SYSCTL_RCC2_OSCSRC2_30KHZ) ||
               ((tmp_rcc2 & SYSCTL_RCC2_OSCSRC2_MASK) == SYSCTL_RCC2_OSCSRC2_32KHZ))
            {
                osc_delay = SYSCTL_SLOW_OSCDELAY;
            }
        }
        else
        {
            if((tmp_rcc & SYSCTL_RCC_OSCSRC_MASK) == SYSCTL_RCC_OSCSRC_30KHZ) 
            {
                osc_delay = SYSCTL_SLOW_OSCDELAY;
            }
           
        }

        sysctl_delay(osc_delay);
    }

    // Select the crystal value and oscillator source. Clear PWRDWN to enable PLL
    tmp_rcc &= ~(SYSCTL_RCC_XTAL_MASK| SYSCTL_RCC_OSCSRC_MASK);
    tmp_rcc |= cfg_rcc & (SYSCTL_RCC_XTAL_MASK | SYSCTL_RCC_OSCSRC_MASK);

    tmp_rcc2 &= ~(SYSCTL_RCC2_USERCC2 | SYSCTL_RCC2_OSCSRC2_MASK);
    tmp_rcc2 |= cfg_rcc2 & (SYSCTL_RCC2_USERCC2 | SYSCTL_RCC2_OSCSRC2_MASK);

    tmp_rcc &= ~(SYSCTL_RCC_PWRDN);
    tmp_rcc |= cfg_rcc & SYSCTL_RCC_PWRDN;

    tmp_rcc2 &= ~(SYSCTL_RCC2_PWRDN2);
    tmp_rcc2 |= cfg_rcc2 & SYSCTL_RCC2_PWRDN2;

    //Clear the PLL lock interrupt
    sysctl->MISC |= SYSTCL_MISC_PLLIM;
    
    sysctl->RCC = tmp_rcc;
    sysctl->RCC2 = tmp_rcc2;

    //delay to ensure new crystal and oscillator source take effect.
    sysctl_delay(16);

    /* Set the USESYSDIV bit in RCC and appropriately set the system divider.
     * Disable the unused oscillator.
     */
    tmp_rcc &= ~(SYSCTL_RCC_USESYSDIV | SYSCTL_RCC_SYSDIV_MASK |
                  SYSCTL_RCC_IOSCDIS | SYSCTL_RCC_IOSCDIS);
    tmp_rcc |= cfg_rcc & (SYSCTL_RCC_USESYSDIV | SYSCTL_RCC_SYSDIV_MASK |
                  SYSCTL_RCC_IOSCDIS | SYSCTL_RCC_IOSCDIS);

    tmp_rcc2 &= ~(SYSCTL_RCC2_SYSDIV2_MASK);
    tmp_rcc2 |= cfg_rcc2 & (SYSCTL_RCC2_SYSDIV2_MASK);

    sysctl->RCC = tmp_rcc;
    sysctl->RCC2 = tmp_rcc2;

    // Enable the use of PLL by clearing the BYPASS
    if((cfg_rcc & SYSCTL_RCC_BYPASS) == 0)
    {
        // Waiting for the PLL to acquire a lock - poll for PLLRIS
        sysctl_wait_pll_lock();

        tmp_rcc  &= ~(SYSCTL_RCC_BYPASS); 
        tmp_rcc2 &= ~(SYSCTL_RCC2_BYPASS2);
    }

    sysctl->RCC = tmp_rcc;
    sysctl->RCC2 = tmp_rcc2;

    //delay to ensure system divider takes effect.
    sysctl_delay(16);
                    
}

```  

The function is written using the follow along section **5.3 Initialization and Configuration in the data sheet**. 
We start off by making local copies of ```RCC\RCC2```, setting ```BYPASS``` and clearing ```USESYS``` in the copies - esentially bypassing PLL and creating a "raw" clock source without any manipulation.We then check if the new configurations require turning on any of the on-board oscillators and do the same. We then set-up delays depending on the clock sources available in the old configuration.  

Wait... how do we measure these delays if we don't have any known clock on board to measure them? Well, in this case, we introduce delays by running a finite loop over a particular count where the processor takes a finite amount of time to process each instruction in the count up/down loop constructed. The delay helps settle the oscillators that have been turned on. And how do we know what delay works? I've chosen a sufficiently large values for delays depening on the oscillators enabled - looks like the manufacturer has also arrived at this via heuristics.  

Next, we clear the ```XTAL``` and ```OSCSRC``` fields in our local copies and set them to select the crystal and oscillator source required by the configs passed. We then clear the ```PWRDWN``` field to turn the PLL on. To set the resulting clock signal to the desired frequency, we'll need to apply the system divider ```SYSDIV``` in the config to the ```RCC\RCC2``` registers appropriately. Before this, we also clear the PLL lock interrupt - we do this to poll for a PLL lock so we can enable it. Once the ```SYSDIV``` has been written, we introduce a short delay just to ensure it is applied. Finally, we poll for the ```PLLRIS``` bit in the ```RIS``` (Raw Interrupt Status) register - which indicates a PLL lock - which is when we enable the use of PLL by clearing ```BYPASS``` in ```RCC\RCC2```.  

