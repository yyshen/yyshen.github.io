---
layout: post
title: "RISCV Hypervisor Extension: htinst and virtual instruction exception"
date: 2021-08-09 10:50:00
categories:
tags: riscv, system, virtualisation
---

RICV HE (hypervisor extension) introduces a new exception type: virtual instruction exception (code 22). A virtual instruction exception is raised instead of
an illegal instruction exception if a instruction is valid to execute in HS-mode but invalid in VS-mode due to insufficient privilege or other reasons. The
following list, not necessarily covering all the cases, is excerpted from the RISCV privileged specification.

1. in VS-mode or VU-mode, attempts to access a counter CSR when the corresponding bit in **hcounteren** is 0 and the same bit in **mcounteren** is 1;
2. in VS-mode or VU-mode, attempts to execute a hypervisor instruction (**HLV**, **HLVX**, **HSV**, or **HFENCE**), or to access an implemented hypervisor
CSR or VS CSRR when the same access (read/write) would be allowed in HS-mode;
3. In VU-mode, attempts to execute **WFI** or a supervisor instruction (**SRET** or **SFENCE**), or to access an implemented supervisor CSR when the same
access (read/write) would be allowed in HS-mode;
4. in VS-mode, attempts to execute **WFI** when **hstatus.VTW**=1 and **mstatus.TW**=0, unless the instruction completes within an implementation-specific,
bounded time;
5. in VS-mode, attempts to execute **SRET** when **hstatus.VTSR**=1; and
6. in VS-mode, attempts to execute an **SFENCE** instruction or t access **satp**, when **hstatus.VTVM**=1.

On a virtual instruction trap, **mtval** or **stval** is written the same as for an illegal instruction trap.

**htinst** provides information about the instruction trapped into HS-mode, if it is not 0. It is allowed that **htinst** is always 0, providing
no additional information about the fault. There are three possibilities if the value is not 0.

1. Bit 0 is 1, and replacing bit 1 with 1 makes the value into a valid encoding of a standard instruction. This is called a transformed instruction.
2. Bit 0 is 1, and replacing bit 1 with 1 makes the value into an instruction encoding that is explicitly designated for a custom instruction. This
is allowed only if the trapping instruction is not standard.
3. When bit 0 and 1 are both zeros, the value is a special pseudoinstruction.

It is quite relax that the value written to **htinst** for an exception type is up to implementations. For instance, for load guest-page fault, zero,
transformed standard instruction, custom value, and pseudo-instruction are all allowed. A implementation could always provide 0.

Currently, the following transformed instructions are defined:

1. For a non-compressed standard load instruction in **LB**, **LBU**, **LH**, **LHU**, **LW**, **LWU**, **FLW**, **FLD**, or **FLQ**;
2. For a non-compressed standard store instruction in  **SB**, **SH**, **SW**, **SD**, **FSW**, **FSD**, or **FSQ**;
3. For a standard atomic instruction;
4. For a standard virtual-machine load/store instruction, **HLV**, **HLVX**, or **HSV**. 

Bits 1:0 of a transformed standard instruction will be binary 01 if the trapping instruction is compressed and 11 if not. For standard basic load and
store instructions, it is sufficient to examine the encodings only.

"For guest-page faults, the trap instruction register is written with a special pseudoinstruction value if: (a) the fault is caused by an implicit
memory access for VS-stage address translation, and (b) a nonzero value (the faulting guest physical address) is written to **mtval2** or **htval**.
If both conditions are met, the value written to **mtinst** or **htinst** must be taken from the following; zero is not allowed."

| Value      | Meaning                                       |
| -----------|-----------------------------------------------|
| 0x00002000 | 32-bit read for VS-stage address translation  |
| 0x00002020 | 32-bit write for VS-stage address translation |
| -----------|-----------------------------------------------|
| 0x00003000 | 64-bit read for VS-stage address translation  |
| 0x00003020 | 64-bit write for VS-stage address translation |

Write pseudeinstructions are for updating A and/or D bits in the VS-level page table only.









