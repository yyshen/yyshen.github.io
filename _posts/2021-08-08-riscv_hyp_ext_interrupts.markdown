---
layout: post
title: "RISCV Hypervisor Extension: Interrupts"
date: 2021-08-08 23:32:00
categories:
tags: riscv, system, virtualisation
---

I am working on RISCV hypervisor extension support for seL4 microkernel in my spare time. See [https://github.com/SEL4PROJ/sel4-riscv-vmm-manifest]
if you are interested.
Injecting interrupts is an important task of a hypervisor. The current RISCV hypervisor extension draft defines several CSRs for the interrupts.
I read the specification several times and still have not got the relations of the CSRs right. I will try to summarise my understanding here to
help myself and (maybe) you.

The **hvip** (hypervisor virtual interrupt pending) CSR is used by the hypervisor to inject an interrupt to VS-mode. Settting **VSEIP** (bit 10),
**VSTIP** (bit 6), or **VSSIP** (bit 2) indicates that an exteranl interrupt, a timer interrupt, or a software interrupt is pending for VS-mode.

**hip** indicates pending interrupts for VS-level interrupts (**VSEIP** (10), **VSTIP** (6), and **VSSIP** (2)) and HS-level interrupts (**SGEIP** (12)).
**hie** has the enable bits (**VSEIE** (10), **VSTIE** (6), **VSSIE** (2), and **SGEIE** (12)) for the corresponding pending interrupt bits. an interrupt
***i*** will be taken in HS-mode if bit ***i*** is set in both hip and hie.

**VSEIP** is read-only in **hip** and is the logical OR of the following:
1. bit **VSEIP** of **hvip**;
2. the bit of **hgeip** selected by **hstatus.VGEIN**; and
3. any other platform-specific exteranl interrupt signal directed to VS-level.

**VSTIP** is read-only in **hip** and it is the logical OR of **hvip**.**VSTIP** and any other paltform-spcific timer interrupt signal directed to VS-level.

**VSSIP** in **hip** is an alias of the same bit in **hvip**.

**hip.SGEIP** and **hie.SGEIE** are for guest external interrupts for HS-level. **SGEIP** is read-only in **hip** and is 1 if and only if the bitwise
logical-AND of CSEs **hgeip** and **hgeie** is nonzero in any bits.

The priority of HS-mdoe interrupts from high to low is **SEI**, **SSI**, **STI**, **SGEI**, **VSEI**, **VSSI**, **VSTI**.

**hgeip** and **hgeie** are CSRs for direct assignment of a virtual machine and a physical device. Each bits summarises all the pending interrupts
for a virtual hart. The hypervisor needs to query the platform interrupt controller to distinguish pending interrupts from multiple devices. Therefore,
this requires supports from platform interrupt controllers. The max number of virtual harts that can receive direct interrupts on a physical hart is
limited by the **GEILEN**, which could be zero. Bits GEILEN:1 are writable in **hgeie** and **hgeip** if it is not 0. Note that the assignment of
direct interrupts to a VM is not discussed in the specification. Another important paragraph is:" **hgeie** selects the subset of guest external
interrupts that cause a supervisor-level (HS-level) guest external interrupt. The enable bits in **hgeie** do not affect the VS-level external
interrupt signal selected from **hgeip** by **hstatus**.VGEIN." This means that a hypervisor can get notifications for guest external interrupts
targetting the virtual harts that are not the current one running on a physical hart by HS-level guest external interrupts. The **hstatus**.VGEIN
selects the current virtual hart that receives direct interrupts. Thus, the hypervisor should also switch that field when switching from one
virtual hart to another. This function is currently unused by seL4.

**vsip** and **vsie** are the VS-mode's **sip** and **sie**. When bit 10 of **hidelege** is 0, **vsip.SEIP** and **vsie.SEIE** are read-only 0. Else,
they are the aliases of **hip.VSEIP** and **hie.VSEIE**. The same applies to other bits as well. For seL4, these delegations are enabled. Therefore,
seL4 mainly uses **hvip** to inject interrupts without changing the the corresponding alias bits in **hip** and **hie**.





