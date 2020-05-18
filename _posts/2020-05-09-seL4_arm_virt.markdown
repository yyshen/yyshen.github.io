---
layout: post
title: Arm Virtualisation on seL4 
date: 2020-05-09 18:11
category: 
author: Yanyan Shen
tags: [seL4, Arm, virtualisation]
summary: 
---

 I have been working on the Armv8 virtualisation support for the
 [seL4](https://github.com/seL4/seL4) microkernel in the past few years, and I
 think it would be useful to write a series articles about it. This is the first
 installment; I will mostly focus on the Armv8 architecture, but Armv7 is not
 so different.

 More information about the seL4 microkernel is available at https://sel4.systems/

### Overview
 In the non-secure world, Armv8 A processors have three **exception levels**:
 EL0, EL1, and EL2. EL0 is where normal application code runs, and EL1 is for
 operating system kernels (e.g., Linux). EL2 is designated for hypervisors.
 When seL4 is configured with virtualisation support, it runs at EL2 as a
 hypervisor. EL2 provides system registers that can be used to configure and
 control the software running at EL1 and EL0. For instance, the `VTTBR_EL2`
 register contains the root page table for stage-2 translations. The stage-2
 page tables are used to further translate the intermediate physical addresses
 to physical addresses. Another important system register is `HCR_EL2`
 (hypervisor configuration register), which provides configuration controls for
 virtualisation features. We will look into related fields in these system
 registers should the need arise.

 When seL4 is used a hypervisor, it provides the following functionalities that
 allow developers to construct virtual machines (**VMs**). 

  1. A new kernel object type called virtual CPU (**VCPU**) is introduced so
  that the owner of a capability to a VCPU kernel object can inspect and
  manipulate the EL1 system registers of the VCPU. A VCPU kernel object is
  associated with a thread control block (**TCB**), which is the scheduling
  entity. The usual EL0 regsiters can be access through the TCB capability.
  
  2. The page table mapping and unmapping system calls work at the stage-2
  translation instead of the stage-1 translation. This means that for native
  seL4 applications running at EL0, stage-1 translation is disabled, but stage-2
  translation controlled by `VTTBR_EL2` is in charge. 
  
  3. System calls for injecting virtual interrupts to a VCPU. This requires the
  hardware support for the generic interrupt controller (**GIC**). Currently,
  GICv2 and GICv3 are supported.
  
There is a missing piece in the puzzle: system memory management unit
(**SMMU**) version 2 is not included in the master branch of the kernel,
but it is work in progress.

As a microkernel, seL4 only provides mechanisms for building VMs. A user-mode
(EL0) virtual machine manager (VMM) is responsible for creating and managing a
VM with the provided systems calls. Therefore, the complicated code for handling
VM exits, device emulation, injecting virtual interrupts, and other tedious
tasks are all implemented in user-mode, without introducing significant new
code to the trusted computing base (TCB). Therefore, a bug in the VMM may
crash the VMM and the VM managed by the VMM, but not the microkernel or other
applications.

