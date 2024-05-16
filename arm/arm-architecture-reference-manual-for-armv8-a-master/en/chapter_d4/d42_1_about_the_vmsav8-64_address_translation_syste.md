## D4.2.1 About the VMSAv8-64 address translation system

[`中文版`](../../zh/chapter_d4/d42_1_about_the_vmsav8-64_address_translation_syste.html)

The Memory Management Unit (MMU) controls address translation, memory access permissions, and memory attribute determination and checking, for memory accesses made by the PE.

The general model of MMU operation is that the MMU takes information about a required memory access, including an input address (IA), and either:

 * Returns an associated output address (OA), and the memory attributes for that address.
 * Is unable to perform the translation for one of a number of reasons, and therefore causes an exception to be generated. This exception is called an MMU fault. An MMU fault is generated by a particular stage of translation, and can be described as either a stage 1 MMU fault or a stage 2 MMU fault.

The process of mapping an IA to an OA is an address translation, or more precisely a single stage of address translation.

The architecture defines a number of translation regimes, where a translation regime comprises either:

 * A single stage of address translation. This maps an input virtual address (VA) to an output physical address (PA).
 * Two, sequential, stages of address translation, where:
  - Stage 1 maps an input VA to an output intermediate physical address (IPA).
  - Stage 2 maps an input IPA to an output PA.

The translation granule specifies the granularity of the mapping from IA to OA. That is, it defines both:

 * The page size for a stage of address translation, where a page is the smallest block of memory for which an IA to OA mapping can be specified.
 * The size of a complete translation table for that stage of address translation.

The MMU is controlled by System registers, that provide independent control of each address translation stage, including a control to disable the stage of address translation. [The effects of disabling a stage of address translation on page D4-1677](#) defines how the MMU handles an access for which a required address translation stage is disabled.

> **NOTE:**

> * In the ARM architecture, a software agent, such as an operating system, that uses or defines stage 1 memory translations, might be unaware of the second stage of translation, and of the distinction between IPA and PA.
> * A more generalized description of the translation regimes is that a regime  always comprises two sequential stages of translation, but in some regimes the stage 2 translation both:
>   - Returns an OA that equals the IA. This is called a flat mapping of the IA to the OA.
>   - Does not change the memory attributes returned by the stage 1 address translation.

For an access to a stage of address translation that does not generate an MMU fault, the MMU translates the IA to the corresponding OA. System registers are used to report any faults that occur on a memory access.

This section describes the address translation system for an implementation that includes all of the Exception levels, and gives a complete description of translations that are controlled by an Exception level that is using AArch64.

Figure D4-2 shows these translation stages and translation regimes when EL3 is using AArch64.

![](figure_d4_2.png)

[ARMv8 VMSA naming on page D4-1643](#) gives more information about the options for the different stages of address translation shown in Figure D4-2, and:

 * Chapter G4 The AArch32 Virtual Memory System Architecture describes:  
   - The translation stages and translation regimes when EL3 is using AArch32.
   - Any stages of address translation that are using VMSAv8-32 when EL3 is using AArch64.
 * [The implemented Exception levels and the resulting translation stages and regimes on page D4-1679](#) describes the effect on the address translation model when some Exception levels are not implemented.

Each enabled stage of address translation uses a set of address translations and associated memory properties held in memory mapped tables called translation tables. A single translation table lookup can resolve only a limited number of bits of the IA, and therefore a single address translation can require multiple lookups. These are described as different levels of lookup.

Translation table entries can be cached in a Translation Lookaside Buffer (TLB).

As well as defining the OA that corresponds to the IA, the translation table entries define the following properties:
 * Access to the Secure or Non-secure address map, for accesses made from Secure state.
 * Memory access permission control.
 * Memory region attributes.

For more information, see [Memory attribute fields in the VMSAv8-64 translation table format descriptors on page D4-1699](#).

The following subsections give more information:
 * ARMv8 VMSA naming.
 * VMSA address types and address spaces.
 * About address translation on page D4-1644.
 * The VMSAv8-64 translation table format on page D4-1644.


### ARMv8 VMSA naming

The ARMv8 VMSA naming model reflects the possible stages of address translation, as follows:

| | |
| -- | -- |
| VMSAv8  | The overall translation scheme, within which an address translation has one or two stages. |
| VMSAv8-32 | The translation scheme for a single stage of address translation that is managed from an Exception level that is using AArch32. |
| VMSAv8-64  | The translation scheme for a single stage of address translation that is managed from an Exception level that is using AArch64. |


### VMSA address types and address spaces  

A description of the VMSA refers to the following address types.

> **NOTE:**  
 These descriptions relate to the VMSAv8 description and therefore give more detail than the generic definitions given in the glossary.


#### Virtual address (VA)

An address used in an instruction, as a data or instruction address, is a Virtual Address (VA).

> **NOTE:**  
> This means that an address held in the PC, LR, SP, or an ELR, is a VA.  

> In AArch64 state, the VA address space has a maximum address width of 48 bits. With a single VA range this gives a maximum VA space of 256TB, with VA range of 0x0000_0000_0000_0000 to 0x0000_FFFF_FFFF_FFFF.

> However, for the EL1&0 translation stage the VA range is split into two subranges, one at the bottom of the full 64-bit address range of the PC, and one at the top, as follows:

> * The bottom VA range runs up from address 0x0000_0000_0000_0000. With the maximum address width of 48 bits this gives a VA range of 0x0000_0000_0000_0000 to 0x0000_FFFF_FFFF_FFFF.
* The top VA subrange runs up to address 0xFFFF_FFFF_FFFF_FFFF. With the maximum address width of 48 bits this gives a VA range of 0xFFFF_0000_0000_0000 to 0xFFFF_FFFF_FFFF_FFFF. Reducing the address width for this subrange increases the bottom address of the range.

> This means that there are two VA subranges, each of up to 256TB.

> Each translation regime, that takes a VA as an input address, can be configured to support fewer than 48 bits of virtual address space, see [Address size configuration on page D4-1646](#).


#### Intermediate physical address (IPA)

 In a translation regime that provides two stages of address translation, the IPA is:
* The OA from the stage 1 translation.
* The IA for the stage 2 translation.

In a translation regime that provides only one stage of address translation, the IPA is identical to the PA. Alternatively, the translation regime can be considered as having no concept of IPAs.

The IPA address space has a maximum address width of 48 bits, see [Address size configuration on page D4-1646](#).


#### Physical address (PA)

The address of a location in a physical memory map. That is, an output address from the PE to the memory system.

The EL3 and Secure EL1 Exception levels provide independent definitions of physical address spaces for Secure and Non-secure operation. This means they provide two independent address spaces, where:
* A VA accessed in Secure state can be translated to either the Secure or the Non-secure physical address space.
* When in Non-secure state, a VA is always mapped to the Non-secure physical address space.

Each PA address space has a maximum address width of 48 bits, but an implementation can implement fewer than 48 bits of physical address. See [Address size configuration on page D4-1646](#).


### About address translation

For a single stage of address translation, a Translation table base register (TTBR) indicates the start of the first translation table required for the mapping from input address to output address. Each implemented translation stage shown in [VMSAv8 AArch64 translation regimes, translation stages, and associated controls on page D4-1642](#) requires its own set of translation tables.

For the EL1&0 stage 1 translation, the split of the VA mapping into two subranges requires two tables, one for the lower part of the VA space, and the other for the upper part of the VA space. [Example use of the split VA range, and the TTBR0_EL1 and TTBR1_EL1 controls on page D4-1671](#) shows how these ranges might be used.

[Controlling address translation stages on page D4-1645](#) summarizes the system control registers that control address translation by the MMU.

A full translation table lookup is called a *translation table walk*. It is performed automatically by hardware, and can have a significant cost in execution time. To support fine granularity of the VA to PA mapping, a single IA to OA translation can require multiple accesses to the translation tables, with each access giving finer granularity. Each access is described as a level of address lookup. The final level of the lookup defines:
 * The high bits of the required output address.
 * The attributes and access permissions of the addressed memory.

Translation table entries can be cached in a Translation Lookaside Buffer, see [Translation Lookaside Buffers (TLBs) on page D4-1729](#).


### The VMSAv8-64 translation table format

Stages of address translation that are controlled by an Exception level that is using AArch64 use the VMSAv8-64 translation table format. This format uses 64-bit descriptor entries in the translation tables.

> **NOTE:**  
This format is an extension of the VMSAv8-32 Long-descriptor translation table format originally defined by the ARMv7 Large Physical Address Extension, and extended slightly by ARMv8. VMSAv8-32 also supports a Short-descriptor translation table format. [Chapter G4 The AArch32 Virtual Memory System Architecture](#) describes both of these formats.

The VMSAv8-64 translation table format provides:
* Up to four levels of address lookup.
* Input addresses of up to 48 bits.
* Output addresses of up to 48 bits.
* A translation granule size of 4KB, 16KB, or 64KB.
