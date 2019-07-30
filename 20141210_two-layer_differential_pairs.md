# Two layer differential pairs

    2014/12/10 Cecill Etheredge // ijsf

---

For one of our high-performance hardware projects, I have been diving into the myths and facts of differential pairs and PCB routing in general. The more documentation I dug up, however, the more questions popped up, leaving me with only a single question: does anyone really know what they're talking about?

![](images/20141210-pcb1.png)

Those who are into this subject may recognize the above: dozens of application notes, guidelines and presentations from established vendors (such as Freescale, TI, Agilent, and many others) and PCB consultants, with nearly all of them contradicting one another on key issues.

A few short examples:

* Isolated ground planes vs single ground planes.
* Tight differential coupling vs loose differential coupling.
* Ferrite beads on supply lines vs no ferrite beads at all.
* Avoiding 45 or 90 degree trace bends vs route your traces any way you like.

First of all, most of these docs and notes have a certain degree of truth in them. They either work for their own specific cases, at the time the research was written, and at the bandwidth and frequency that is most common for their field of application. For example, many of the above constraints are true, but they only play a significant role once the frequency reaches a certain amount of gigahertz.

The second thing to note is that although these application notes, guidelines and presentations often contain some form of empirical evidence.. they are not academic research and should therefore be taken with a grain of salt.

The initial reason for writing this text was to shine some light on our problem of differential routing: differential pairs, but on two layer boards.

None of the usual docs contained any information on using differential pairs on two layer boards, apart from claims such as "it's bad for signal integrity" but then failing to mention why. And that set aside, I started looking in academic papers and patents instead and found something academical that described exactly what I was looking for:

[A Low Cost Dual-CPW Differential Line for Two-Layer PCBs](http://ieeexplore.ieee.org/xpls/abs_all.jsp?arnumber=4560016) by Chi et al. (2008) from ASUSTeK Computer Inc. Definitely worth a read.
