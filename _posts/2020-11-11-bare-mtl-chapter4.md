---
layout: posts
title: "Chapter Four: Time and tide..."
date: 2020-11-11
---  

Our system is now running software - every bit of which has been written by us - using which we can interact with it (in terms of accepting our keyboard inputs to display the same on the console). We did this by programming the uart device and the interrupt controller.  

The actual hardware comes with a variety of peripherals and interfaces to potentail peripherals that can be controlled by our processor onboard. Things like GPIO, I2C, SSI, SPI, PWM, ADC, ethernet etc. In order to use these interfaces and peripherals or even run useful applications, one of the prerequisites happens to be the ability to keep/measure time (not to mention the fact that this is something that is achievable with the use of an emulator).  

Time keeping on any hardware is achieved with the use of clock and timer hardware available on boad. Programming these bits of hardware and being able to keep time accurately will be the subject of this chapter. If you recall the time when we started programming the uart driver, we had assumed that the system clock frequency was about 12.5 MHz by default and we had enabled the clock signal to drive the uart peripheral. It's now time to throw the assumptions out the window and actually set up a system clock and timer driver. We'll then use these to build a simple scheduler.  

### I'm a little cuckoo clock...

The hardware lm3s6965 comes with a set of clock sources (crystal - also called xtal - oscillators) on board that can be controlled via SFRs (what else?!). These clocks are similar in operation to your mum and dad's quartz wristwatches (yes, in the age of the smartwatch, quartz stays to mum and dad) and are part of the system control. More details on the various clock sources and their properties can be found in section 5.2.4 of the data sheet. Utlimately, the internal system clock (SysClk) which drives the various peripherals and aides in keeping time is derived from one of these clocks on board. Figure 5-5 on the data sheet gives us a good look at how the various clock sources are arranged and their relation with the system clock.  

Controlling the clock and getting a stable accurate system clock frequency is key to operating the peripherals as well as for time keeping. As with other peripherals discussed earlier, we'll need to program the SFRs corresponding to the system control (specifically, clock control). The data sheet has a great follow along for programming the clock control.  

The source code for this section is available [here](https://github.com/varun-venkatesh/bare-metal-arm/tree/master/src/chapter4).

[//]: # (This may be the most platform independent comment)  

The Run-Mode Clock Configuration (RCC) and Run-Mode Clock Configuration 2 (RCC2) registers provide control for the system clock. The configurations include picking an oscillator from the ones available and the crystal used with it - should the main oscillator be selected (remember, internal oscillators are on-board and don't need any extrnal crystal source - this makes them less complicated and also less accurate). It also allows for the use of a PLL (Pase Locked Loop) which esentially multiplies the clock signal from the selected oscillator source putting out a signal at 400 MHz. The PLL output is subject to what is called a system divider to get a frequency of the users choosing (prior to this, the PLL frequency is divided by 2 == 200 MHz). Additional configurations include - clock gating, clock settings for the PWM module.  

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

Once we've set the clock to an appropriate frequency, we can go on to use it in our drivers that need some manner of clocking - for instance the UART. If you remember, we assumed that the default SysClk operated at 12.5 MHz. With the clock now set to a frequency of our choosing, we can replace that assumption with the actual SysClk frequency. To get this, we'll need to again look into registers ```RCC\RCC2``` (programmatically, ofcourse) and ascertain the frequency of the SysClk, like so:  

```C
/* This is function is for finding the system clock frequency.
 * Helps with setting time-periods/time-outs.
 * Refer: http://www.ti.com/lit/ds/symlink/lm3s6965.pdf 
 * Section 5-5 Pages 195-201
 */
uint32_t sysctl_getclk(void)
{
    uint32_t tmp_rcc, tmp_rcc2, tmp_pllcfg, clk_rt;

    // Read from the config registers RCC/RCC2
    tmp_rcc = sysctl->RCC;
    tmp_rcc2 = sysctl->RCC2;

    // determin the oscillator used (and if RCC2 overrides RCC)
    switch((tmp_rcc2 & SYSCTL_RCC2_USERCC2) ?
           (tmp_rcc2 & SYSCTL_RCC2_OSCSRC2_MASK) :
           (tmp_rcc & SYSCTL_RCC_OSCSRC_MASK))
    {
        case SYSCTL_RCC_OSCSRC_MOSC:
        {
            clk_rt = xtal_freq[(tmp_rcc & SYSCTL_RCC_XTAL_MASK) >> SYSCTL_RCC_XTAL_SHIFT];
            break;
        }

        case SYSCTL_RCC_OSCSRC_IOSC:
        {
            clk_rt = FREQ_12MHZ;
            break;
        }
        case SYSCTL_RCC_OSCSRC_IOSC_DIV4:
        {
            clk_rt = (FREQ_12MHZ)/4;
            break;
        }
        case SYSCTL_RCC_OSCSRC_30KHZ:
        {
            clk_rt = FREQ_30KHZ;
            break;
        }
        case SYSCTL_RCC2_OSCSRC2_32KHZ:
        {
            clk_rt = FREQ_32KHZ;
            break;
        }

    }

    // check if BYPASS is off, if it is, find the PLL frequency
    if(((tmp_rcc2 & SYSCTL_RCC2_USERCC2) != 0 && (tmp_rcc2 & SYSCTL_RCC2_BYPASS2) == 0) ||
       ((tmp_rcc2 & SYSCTL_RCC2_USERCC2) == 0 && (tmp_rcc & SYSCTL_RCC_BYPASS) == 0))
    {
        // Read PLLCFG to find the Fvalue and RValue
        tmp_pllcfg = sysctl->PLLCFG;
        
        // Compute PLLFreq = OSCFreq * F / (R + 1)
        clk_rt = clk_rt * 
                ((tmp_pllcfg & SYSCTL_PLLCFG_FVAL_MASK) >> SYSCTL_PLLCFG_FVAL_SHIFT) / 
                (((tmp_pllcfg & SYSCTL_PLLCFG_RVAL_MASK) >> SYSCTL_PLLCFG_RVAL_SHIFT) + 1);
    }

    /* apply the SYSDIV value to compute the frequency of the system clock.
     * if BYPASS is OFF, then PLL is ON, which means the PLL frequency computed earlier
     * must be divided by 2 (Refer:  http://www.ti.com/lit/ds/symlink/lm3s6965.pdf 
     * Section 5.2.4.2. Apply the appropriate SYSDIV factor - as in Table 5-5 and Table 5-6
     */
    if(tmp_rcc & SYSCTL_RCC_USESYSDIV)
    {
        if((tmp_rcc2 & SYSCTL_RCC2_USERCC2))
        {
            if((tmp_rcc2 & SYSCTL_RCC2_BYPASS2) == 0)
            {
                clk_rt = clk_rt/2;
            }

            clk_rt = clk_rt / 
                     (((tmp_rcc2 & SYSCTL_RCC2_SYSDIV2_MASK) >> 
                      SYSCTL_RCC2_SYSDIV2_SHIFT) + 1);
        }
        else 
        {
            if((tmp_rcc & SYSCTL_RCC_BYPASS) == 0)
            {
                clk_rt = clk_rt/2;
            }

            clk_rt = clk_rt / 
                     (((tmp_rcc & SYSCTL_RCC_SYSDIV_MASK) >> 
                      SYSCTL_RCC_SYSDIV_SHIFT) + 1);
        }
    }

    return (clk_rt);
}
```  

To cut this long story short (as this can be drawn from the register description and the clock control operation), what we do is to firstly determine the clock source in use (and also whether ```RCC``` or ```RCC2``` is where the source is selected - note that for source selection and system divider values, ```RCC2``` overrides ```RCC```). If the clock source happens to be the main oscillator, the clock-source frequency can be obtained using the ```XTAL``` field. For all other sources, the clock-source frequency comes pre-determined. We then check if PLL is enabled via ```BYPASS``` use the ```PLLCFG``` register to translate the clock-source frequency obtained into the corresponding PLL frequency (See  - although PLL generates a clock signal of frequency 400 MHz, it can vary around that value depending on teh clock-source frequency (See Table 22-10). Finally, we check ```USESYSDIV``` and apply the system divider ```SYSDIV``` to the clock frequency we have so far (again, if PLL is enabled, we divide the frequency by 2 before applying ```SYSDIV```).  

We can now use this method to derive our IBRD and FBRD values for the uart baudrate.  

```C
/* Set the uart baudrate of the uart device*/
static void uart_set_baudrate(uint32_t baudrate)
{
    uint32_t sysclk, brdi, brdf, dvsr, remd;

    /* Refer http://www.ti.com/lit/ds/symlink/lm3s6965.pdf 12.3.2 */
    sysclk = sysctl_getclk();

    dvsr = (baudrate * 16u);

    /* integer part of the baud rate */
    brdi = sysclk/dvsr;

    /* fractional part of the baudrate */
    remd = sysclk - dvsr * brdi;
    brdf = ((remd << 6u) + (dvsr >> 1))/dvsr;

    uart0->IBRD = (uint16_t)(brdi & 0xffffu);
    uart0->FBRD = (uint8_t)(brdf & 0x3ffu);
}
```  

Notice how we replaced the hard-coded macro ```UART_DFLT_SYSCLK``` with the clock-frequency obtained using ```sysctl_getclk()```.  
There's one last thing we can do with the system control register i.e. enable clocking for the ```UART0``` (or any of the peripherals that need clocking). This is done with the help of the Clock Gating Control Registers. There are 6 of these registers available to us - ``` RCGC1, SCGC1, DCGC1, RCGC2, SCGC2, DCGC2``` - each one controlling the clock gating in the various modes of operation. Since we won't be dealing with the Sleep/Deep-Sleep modes here, we'll look at ```RCGC1/RCGC2```. From thier descriptions, it appears that ```RCGC1``` controls clock gating for UARTs. Let's enable this for our ```UART0``` and use it in place of the hacky ```set_clk_uart0()``` from chapter 3.  

```C

/* This function helps us enable clocking for a peripheral whose 
 * base address is passed as a parameter. This is done by appropriatley configuring
 * RCGC1. Refer: http://www.ti.com/lit/ds/symlink/lm3s6965.pdf Section 5-5 Pages 220-222
 */
void sysctl_periph_clk_enable(uint32_t periph)
{
    switch(periph)
    {
        case UART0_BASE:
            sysctl->RCGC1 |= SYSCTL_RCGC1_UART0;
            break;
        case UART1_BASE:
            sysctl->RCGC1 |= SYSCTL_RCGC1_UART1;
            break;
        case UART2_BASE:
            sysctl->RCGC1 |= SYSCTL_RCGC1_UART2;
            break;
        default:
            break;
    }

}

```  

We can then call this in ```uart_init()``` in place of ```set_clk_uart0()```.  

To put this in use, let's call ```sysctl_setclk(clk_cfg1, clk_cfg2)``` in our initialization function ```main()``` (we've now moved main out of ```serial_print.c``` into a new ```init.c``` function as we'll be initializing more than just the uart). The variable ```clk_cfg1, clk_cfg2``` represent the configuration required to operate the clock at the desired frequency.  

```C
    /* Set the system clock to the PLL with the main oscillator as the source
     * with the crystal frequency set to 8 MHz. 
     * Divide the PLL output clock frquency by a factor of 12.
     * Turn off the (unused) internal oscillator. This is to configure a system clock of 16.67 MHz.
     */
    clk_cfg1 = (SYSCTL_PLL_SYSCLK | SYSCTL_RCC_USESYSDIV | SYSCTL_RCC_SYSDIV_11 | 
               SYSCTL_RCC_XTAL_8MHZ | SYSCTL_RCC_OSCSRC_MOSC | SYSCTL_RCC_IOSCDIS);
    clk_cfg2 = 0;

    sysctl_setclk(clk_cfg1, clk_cfg2);
```  

### ... Tick tock, tick tock...  

Other than driving peripherals that need a clocking/synchronizing signal, we need clock to measure time and say trigger actions on the expiry of a pre-determined interval. This is where timers come into play. As you may have read in the data sheet, the LM3S6965 provides among other things a System timer (SysTick) described in Section 3.1. It operates by counting down from a value (max 24-bits == 16,777,215) at the end of which vector number 15 is triggered - where we can handle the expiry of the timer to perform some useful action. Some of the uses are detailed in the section. We'll use it to build a simple scheduler in the next chapter.  

Like any other module/peripheral, programming SysTick involves manipulating it's SFRs:

**STCTRL**: SysTick Control and Status is a control and status counter to configure its clock, enable the counter, enable the SysTick interrupt, and determine counter status.
**STRELOAD**: SysTick Reload Value holds the reload value for the counter, used to provide the counter's wrap value.
**STCURRENT**: SysTick Current Value holds current value of the counter.  

On enabling the the SysTick via ```STCTRL``` the timer counts down on each clock pulse (the clock can be configured to the SysClk or an external clock source) from the value present in ```STRELOAD``` to zero - at which point the SysTick Exception is generated. Following this, the counter wraps around to the ```STRELOAD``` - rinse and repeat.  Clearing ```STRELOAD``` puts a stop to the counter on the next wrap around. ```STCURRENT``` can be used to monitor the current value of count - writing to this clears it and also the ```COUNT``` bit in ```STCTRL```. All of this and more is available in the register descriptions.  

Let's go on an program the SysTick. We'll do this in such a way that we can print messages at some pre-determined interval - which is a great way to verify our SysClk and SysTick programming.  

As usual, we start off by creating a structure to hold the SFRs and initialize a member to point to the SySTick base address. We'll define functions to enable/disable SysTick and set the clock source to SysClk. We'll also write fucntions to enable/disable the SysTick interrupt.  

```C
void systick_enable(void)
{
    systick->STCTRL |= STCTRL_CLKSRC | STCTRL_ENABLE;
}

/* Function to enable the SysTick timer 
 * by clearing the ENABLE bit and 
 */
void systick_disable(void)
{
    systick->STCTRL &= ~(STCTRL_ENABLE);
}

/* Function to enable the SysTick timer interrupt 
 * by setting the INTEN bit and 
 */
void systick_irq_enable(void)
{
    systick->STCTRL |= STCTRL_INTEN; 
}

/* Function to disable the SysTick timer interrupt 
 * by clearing the INTEN bit and 
 */
void systick_irq_disable(void)
{
    systick->STCTRL &= ~(STCTRL_INTEN);
}

```  

Now to the fun part, let's set up ```STRELOAD``` to hold the count. The count can be any 24-bit number which is counted down on every clock pulse, but in order to be useful, it should represent some measure of time (seconds, milliseonds etc). Since we power the SysTIck with SysClk, whose frequency we know, let's find what the count value is for a given time period (number of clock pulses per time period).  

```C

/* Utility function to convert the
 * the time period in milliseconds to
 * the timer period based on the system clock frquency
 */
static uint32_t systick_millisec_to_timer_period(uint32_t millisec)
{
    /* this scheme of dividing the clock frequency by 1000
     * to calculate cycles per millisec rather than multiply clock frequency
     * with millisec and divide by 1000 - is to avoid the overflow in the latter case.
     */
    uint32_t period = (sysctl_getclk()/1000u) * millisec;
    return period;

}

/* Function to use the time period in milliseconds provided 
 * as the SysTick countdown value -by first converting the
 * period in milliseconds to the timer period.
 */
void systick_set_period_ms(uint32_t millisec)
{
    /* TODO: check to ensure that the period is 
     * no greater than 2^24 (- uses 24 bits)
     */
    uint32_t count = systick_millisec_to_timer_period(millisec);
    systick->STRELOAD = count;
}

```  

Now, onto changes in our startup and init to accomodate SysTick.  

```C
    /* Let's set systick period to be 0.5 seconds =>
     * system clock frequency divided by 2.
     */
    systick_set_period_ms(500u);

    /* Let's enable the systick timer and it's interrupt */
    systick_irq_enable();
    systick_enable();
```

We set the SysTick period to be ```500ms``` and enable the SysTick interrupt followed by the SyStick itself (with the SysClk as it's clock source). We'll then remove the association of the ```_SysTick_Handler``` to the ```Unused_Handler``` so that we can define ```_SysTick_Handler``` to do what we want. Let's define it like so, in the ```systick.c```:  

```C
/* The SysTick interrupt handler - currently configured to print 
 * the number of Systick ticks elapsed at the rate of every 20 ticks
 */
void _SysTick_Handler(void)
{
    tick_count++;
    if(tick_count % 20u == 0)
    {
        serial_put_uint(tick_count);
        serial_puts(" time ticks have elapsed!\n");
    }
}
```  

What we're doing here on every SysTick interrupt (which is when this is invoked) is counting 20 ticks and printing a message of the number of ticks that have elapsed. Since each tick represents the count in ```STRELOAD``` which in turn represents the number of clock pulses generated every ```500ms``` (as set in ```systick_set_period_ms(500u)```), the message must appear every ```500ms * 20 == 10s```. If you build this and run qemu as follows:  

```qemu-system-arm -M lm3s6965evb -kernel system.bin -nographic -monitor telnet:127.0.0.1:1234,server,nowait | gawk '{ print strftime("[%H:%M:%S]"), $0 }' ```

You will see the system time prefixing each console print and be able to verify if indeed the SysTick messages appear every 10s - which verifies both our clock configuartion as well as SysTick.  

### References:  

Apart from the data sheet, I found that the ARM's hardware abstarction layer implemenatation [CMSIS](https://github.com/ARM-software/CMSIS_5/tree/develop/Device/ARM/ARMCM3) was pretty useful particularly when finding out what delays were to be used to settle the oscillator and PLL.  


