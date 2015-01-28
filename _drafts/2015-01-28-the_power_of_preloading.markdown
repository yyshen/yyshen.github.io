---
layout: post
title:  "The Power of Preloading"
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
NEOM memory copy with PLD is the fastest. I repeat the code below for your
convenience. As you can see, `PLD` instruction tries to prefetch the data from
memory. Please note that the offset in the `PLD` instruction, it means that
the `PLD` is actually preparing data the the next round of `VMDM` and `VSTM`
pair. The code hopes that the processor can overlap preparing data for next
round of copy with current copy instructions, and hopefully when the next
round of copy starts, the required data is already in cache. 

{% highlight gas %}
NEONCopyPLD:
    PLD {r1, #0xc0]
    VLDM r1!, {d0-d7}
    VSTM r0!, {d0-d7}
    SUBS r2, r2, #0x40
    BGE NEONCopyPLD
{% endhighlight %}


{% highlight c %}
__asm__ __volatile__(
        "ldr %r1        \n"
        ::: "memory");
{% endhighlight %}

[1]: https://github.com/yyshen/mbw
[2]: http://boundarydevices.com/product/sabre-lite-imx6-sbc/
[3]: http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.faqs/ka13544.html

