---
layout: post
title:  "The Power of Pre-loading"
date:   2015-01-28 22:01:00
categories:
tags: arm, memory, assembly
---
When I was running [mbw][1] on an ARM platform, specifically a [SABRE Lite][2]
development board, I noticed that the results using `memcpy` supplied by 
default libc library is much faster than the word-by-word copy. This is actually
not a surprise. I was wondering how I could achieve similar speed. After some
searches, I found that ARM provided an excellent document about 
[What is the fastest way to copy memory on a Cortex-A8][3], which concludes that
NEON memory copy with PLD is the fastest. I repeat the [code](#neon) below for your
convenience. As you can see, `PLD` instruction tries to pre-fetch the data from
memory. Please note that the offset in the `PLD` instruction, it means that
the `PLD` is actually preparing data for the next round of `VMDM` and `VSTM`
pair. The code hopes that the processor can overlap preparing data for next
round of copy with current copy instructions, and hopefully when the next
round of copy starts, the required data is already in cache. The following
NEON instructions first load the data to 8 registers `d0` to `d7` from
the source, and then the data is stored to destination memory. The exclamation
marks after the `r0` and `r1` registers are used to increase the source
and destination addresses in `r0` and `r1` automatically after each load and
store.

<center><a name="neon">NEON memcpy</a> </center>
{% highlight gas %}
NEONCopyPLD:
    PLD {r1, #0xc0]
    VLDM r1!, {d0-d7}
    VSTM r0!, {d0-d7}
    SUBS r2, r2, #0x40
    BGE NEONCopyPLD
{% endhighlight %}



As expected, the `NEONCopyPLD` does achieve higher memory bandwidth, but it
is still not comparable with the libc `memcpy`. Of course I am so curious 
about the reason, so I compiled the mbw with static linking and disassembled
the binary to find out why. The standard library `memcpy` uses `ldm` and `stm` 
instructions,  which also operate on multiple general-purpose registers. 
Basically the [code](#ldm_stm) demonstrates the basic idea and skips a lot of checks
on alignment or length. The main difference is that the code uses multiple `pld`
instructions before actually copies the data. My understanding (I could be wrong)
is that multiple `pld` instructions are using different execution units
of the processor to pre-fetch data from memory while the `ldm` and `stm` instructions
are copying data. One single `pld` may finish too early so that the pre-fetch
unit is idling instead of preparing for the subsequent data. Please let me know 
if you know exactly what is going on under the hood. The `pld` instruction
should not trigger synchronous data abort the address to be pre-fetched can not
be translate to a physical address.

<center><a name="ldm_stm">LDM/STM memcpy</a></center>
{% highlight c %}
void *memcpy(void *dest, const void *src, size_t n)
{
    __asm__ __volatile__(
            "push   {r3-r10}        \n"
            "1:                     \n"
            "pld    [%1]            \n"
            "pld    [%1, #28]       \n"
            "pld    [%1, #60]       \n"
            "pld    [%1, #92]       \n"
            "pld    [%1, #124]      \n"
            "ldmia  %1!, {r3-r10}   \n"
            "stmia  %0!, {r3-r10}   \n"
            "ldmia  %1!, {r3-r10}   \n"
            "stmia  %0!, {r3-r10}   \n" 
            "subs   %2, %2, #0x40   \n"
            "bge    1b              \n"
            "pop    {r3-r10}        \n"
            :
            : "r"(dest), "r"(src), "r"(n)
            : "memory"
            );
}
{% endhighlight %}

The end result is that I achieve similar speed as the standard C library `memcpy`.
Actually a little faster since the code skips so many checks. You may download
the [mbw][1] benchmark and try it out yourselves. This experiment shows the importance
of memory pre-fetching, and that is where the title comes from. 

[1]: https://github.com/yyshen/mbw
[2]: http://boundarydevices.com/product/sabre-lite-imx6-sbc/
[3]: http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.faqs/ka13544.html

