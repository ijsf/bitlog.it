# ASIC roundup of open source RISC-V CPU cores

_2022/01/18 Oguz Meteer // guztech_

---

While waiting for simulation results for my final paper, I thought I'd synthesize and do place & route of several open source [RISC-V](https://riscv.org/) CPU cores for fun. Some basic information:

- All CPU cores were synthesized using a well known 65 nm PDK.
- Synopsys Design Compiler with *ultra* effort was used for synthesis.
- Cadence Innovus was used for place & route.
- A standard I/O template was generated with Innovus with a square floorplan.
- Only the CPU core with the register file and a standardized bus (Wishbone, AXI, AHB, etc.) was taken into account. No full SoCs were used to make the comparisons more fair.

## SERV

The [award-winning](https://riscv.org/blog/2018/12/risc-v-softcpu-contest-highlights/) [SERV](https://github.com/olofk/serv) CPU by [Olof Kindgren](https://twitter.com/OlofKindgren) is a bit-serial RISC-V CPU that is focussed on being as minimal as possible. It may not be the fastest CPU, but it is indeed the smallest RISC-V CPU in this roundup (and possible ever). Here, I used [SERV version 1.1.0](https://github.com/olofk/serv/tree/1.1.0) with the default configuration.

![SERV v1.10](images/serv110.png)

Maximum clock frequency: **1020 MHz**
Die area: **0.029584 mm^2**

## PicoRV32

The excellent [PicoRV32](https://github.com/YosysHQ/picorv32) CPU by [Claire Xenia Wolf](https://twitter.com/oe1cxw) is not only a very flexible core, but it's also the first formally verified RISC-V CPU! The default configuration was used.

![PicoRV32](images/picorv32.png)

Maximum clock frequency: **806 MHz**
Die area: **0.0425152 mm^2**

## Minerva

The [Minerva](https://github.com/minerva-cpu/minerva) CPU by [Lambda Concept](https://twitter.com/LambdaConcept/) is a RISC-V CPU written in [Amaranth HDL](https://github.com/amaranth-lang/amaranth) which is a hardware description language written in Python. The default configuration was used (no I$ / D$).

![Minerva](images/minerva.png)

Maximum clock frequency: **625 MHz**
Die area: **0.051076 mm^2**

If you have questions and/or constructive criticism, let me know on Twitter [@BitlogIT](https://twitter.com/BitlogIT)
