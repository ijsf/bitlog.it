# ⚡ AX88772B on Windows 98: Teaching an Old Driver New Tricks

💾 For retro enthusiasts, sometimes life is hard.

Imagine for a moment, you've got your whole new retro PC build just about set up: Windows 98 SE has finally installed after six retries, and your meticulously collected Win9x drivers are all rigged in place without any more dreaded Blue Screens Of Death. You plug in your network cable, and... nothing. Tumbleweeds.

Turns out, sometime long ago, the onboard network controller and the RJ45 port stopped talking to each other, and there's no way you're going to get things going again with these two.

Another PCI NIC card is, of course, out of the question. There's only one PCI slot, and it's been taken up by some mighty retro graphics card.

What do you do? Well, it's modern times, right? Plug in a USB Ethernet adapter.

Turns out, none of these modern adapters have Windows 98 drivers. Except for one or two manufacturers, maybe?

So you buy this particular adapter with an AX88772 chip, which has been mentioned a few times here and there on some forums. The manufacturer stopped making Windows 98 drivers about 20 years ago, but there was just one last driver from '05 for this particular chip. You somehow get the drivers installed.

Nothing.

Surprise: your AX88772 adapter reveals itself to be an AX88772**B**. Just one little character of difference, but with a different attitude entirely. The hottest chip on the block in '15. After all, times have changed, computers have become faster. And it doesn't quite want to work with the 10-year-old driver you have, because that one was made for the AX88772A, the dinosaur chip from '05, by now obsolete and ancient.

What to do?

Time to bring out the tools. 🛠️

---

## Down the Rabbit Hole

The ASIX AX88772B controller is a newer revision of the original AX88772/AX88772A controller for which there is an original Win9x driver available from '05. Unfortunately, the B variant breaks a few things in a way that make it incompatible with the original driver, resulting in a broken network stack.

With a bit of clever binary patching however, we can create a driver that actually works for these newer chips. Here's a breakdown of this little adventure, which was only written *after* the fact to get to the point more quickly.

In reality, quite a few midnight hours were spent trying out different and ultimately useless patches and trying to prevent faults and BSODs because of wrong x86 opcodes.

But here's the gist without all of that:

---

## What Changed (And What Broke)

A bit of digging around in datasheets and Linux/BSD drivers shows a few potential problems with the newer chip:

<br>

### ❶ Wrong RX Control Register Setting

One of the configuration registers (RX Control Register at `0x10`) was changed. And not in a good way.

The RX Control Register is two bytes long. On the **AX88772A**, it looks like this:

```
Bit 7   Bit 6   Bit 5   Bit 4   Bit 3   Bit 2   Bit 1   Bit 0
SO      Res     AP      AM      AB      0       AMALL   PRO

Bit 15  Bit 14  Bit 13  Bit 12  Bit 11  Bit 10  Bit 9   Bit 8
0       0       0       LPBK    Res     Res     MFB1    MFB0
```

On the **AX88772B**, it was changed to this:

```
Bit 7   Bit 6   Bit 5   Bit 4   Bit 3   Bit 2   Bit 1   Bit 0
SO      ARP     AP      AM      AB      0       AMALL   PRO

Bit 15  Bit 14  Bit 13  Bit 12  Bit 11  Bit 10  Bit 9   Bit 8
0       0       0       0       0       RH3M    RH2M    RH1M
```

The important fields:

| Field | Meaning |
|-------|---------|
| MFB0, MFB1 | Maximum Frame Burst: controls the maximum USB bulk transfer size for received packets (00=2KB, 01=4KB, 10=8KB, 11=16KB) |
| RH1M | RX Header Mode 1: when set, prepends a 4-byte header to each packet in the RX buffer |
| RH2M | RX Header Mode 2: when set, aligns IP header to 32-bit boundary in receive buffer |
| RH3M | RX Header Mode 3: when set, includes hardware checksum value in RX header |

This would've been fine, if not for the fact that in an AX88772A driver, MFB was always set to `0x3` (all bits high). Which means that in the AX88772B, RH1M and RH2M will be set high. As it turns out, setting these two fields changes the way the controller sends any received (RX) packets to the driver. Because we're going to be changing an AX88772A driver, none of that parsing logic should change if we can help it. So those fields need to be set to zero explicitly.

<br>

### ❷ MFB Moved to a Separate Command

It doesn't just end there. The MFB fields were not just removed from RX Control, they were actually moved into a separate command. Of course, that command couldn't be found in any of the datasheets, but the open-source ASIX drivers showed it was necessary to actually set the buffer sizes (e.g. to 16KB), like before: `AX_QCTCTRL` at address `0x2A`.

<br>

### ❸ Reset Sequence Changed

The bringup of the actual controller, and specifically its internal PHY (Physical Layer Device, a chip implementing the lowest OSI networking layer) had changed as well.

Resetting the AX88772 chips involves manipulating the Software Reset Register (at `0x20`) and some explicit waiting times in between, in a specific bringup sequence. The relevant fields:

| Field | Meaning |
|-------|---------|
| IPRL | Internal PHY Reset Latch: when cleared, holds the internal PHY in reset state |
| IPPD | Internal PHY Power Down: when set, powers down the internal PHY |

The **AX88772A** reset sequence, described in the datasheet, more or less looks something like this:

```
SW_RESET ← 0x40 (IPPD)      Power down internal PHY
wait 500ns

SW_RESET ← 0x00             Clear power down, PHY starts coming up
wait 160ms

SW_RESET ← 0x28 (IPRL)      Release PHY from reset
wait 500ns
```

The wait times are minimums, and IPPD and IPRL can be combined, so there are some variations possible.

Looking at the **AX88772B** reset sequence, we see this:

```
SW_RESET ← 0x40 (IPPD)      Power down internal PHY
wait 500ns

SW_RESET ← 0x00             Clear power down, PHY starts coming up
wait 600ms                  ← AX88772B needs 10× longer!

SW_RESET ← 0x28 (IPRL)      Release PHY from reset
wait 500ns
```

The reset wait time is more than 3× longer. And this makes sense. Here's a new flavour chip, with some new features, and it just needs a little more time when booting up after a reset sequence.

To be fair, in different open-source drivers, we see different sleep times and variations of register fields being used. It is not completely clear what works and what doesn't. But sticking to the datasheet recommendations is probably smart: so increase the sleep time, to make sure the controller bringup works reliably.

<br>

### ❹ RX Header Length Bits

Even with all the above changes, there's one more breaking change, and it's a big one. While we made sure not to use this new RX packet header format in ❶, the AX88772B still breaks the header in a sneaky way.

In the **AX88772A**, the packet length field of the RX header looks something like this:

```
┌─────────────────┬─────────────────┐
│    len (16)     │    ~len (16)    │
└─────────────────┴─────────────────┘

len  → 16-bit packet length (0–65535)
~len → one's complement
```

In other words, 2 + 2 bytes with the same data (packet length) but one in one's complement. This lets the driver do some error checking to make sure the length isn't corrupted.

In the **AX88772B**, some of those fields were cannibalized to encode other data, probably some checksum:

```
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│    len (11)     │      ???        │   ~len (11)     │      ???        │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘

len → 11-bit packet length (0–2047)
```

The packet length was shortened to 11-bit. This also means that an AX88772A driver doesn't mask off those 5 removed bits. And since those bits contain other data, the packet length becomes garbage. The packets get corrupted. The network stack stops working.

---

## The Easy Part

The AX88772A driver bundle is tiny. What a difference from today's drivers. Just an `.INF` file, and a `.SYS` file, taking up a measly 19 kilobytes.

The `.INF` contains driver information that lets it register the driver for a particular USB vendor ID (`0B95`) and product ID (`772A`). Now for the AX88772B, that product ID changed to `772B`, so the driver wouldn't respond to our USB adapter. So we change a few things around, like these:

```ini
[ControlFlags]
ExcludeFromSelect = USB\VID_0B95&PID_772B

[USB]
%ax88772B.DeviceDesc% = ax88772B.Ndi,USB\VID_0B95&PID_772B
```

And a few other places in the `.INF`: basically anything that references `772A` is changed to `772B`. After changing this, the driver lets you install it through the Windows Device Manager.

It installs, and seems to work... but the network doesn't do anything.

Time to open up Ghidra and disassemble the driver.

---

## Disassembly Time

In the Windows 98 era, there were already several supported driver formats and abstraction layers, for example:

- **PE** (Portable Executable): The 32-bit `.SYS` format which, in a way, is an earlier version of the `.SYS` format still used today.
- **LE** (Linear Executable): The 16-bit or 32-bit format, with its origins in Windows 3.x and OS/2.
- **VxD**: The driver model which became common during the Windows 3.x and OS/2 era and provided backwards compatibility with Windows 95.
- **WDM** (Windows Driver Model): An early version of the driver model which still exists today and went on to become the de-facto driver model in later versions of Windows.
- **NDIS** (Network Driver Interface Specification): A standardized API that creates an abstraction between the OS network stack and the network devices.

When we look at `AX88772.SYS` the first thing we notice is that it is a 32-bit LE. Looking at the imports reveals the type of driver we're dealing with. With `HAL.DLL`, `NDIS.SYS`, `NTOSKRNL.EXE`, `USBD.SYS` being imported, this turned out to be an NDIS driver. Also noteworthy is the `NTOSKRNL.EXE` import, which wouldn't be present on Windows 9x and likely indicates that this driver is some kind of hybrid for both Win9x and WinNT.

Luckily, the driver is pretty small. Only around 18 KiB, and Ghidra gives us about 22 functions after analysis. That's manageable.

Since this is a USB device, most of the commands are going to be fired through the USB subsystem. And as we've seen above, it is exactly these USB commands that we need to "tweak" in order to make things work again.

---

## Patch #1: The Initialization Function

Somewhere among the 22 functions is one function that does a lot of actions to bring up the link. In fact, it starts by querying some NDIS parameters (accessible from Network in the Control Panel) so it must be pretty close to where we need to be. We call it `asix_miniport_initialize` because it sits right in between NDIS and the physical device, also called the "miniport" domain.

What makes this function so interesting, is that it touches the broken registers in ❶ and ❸.

### The RX Control Problem

Here's the instruction that sets the RX Control Register:

```asm
00013156  mov  word ptr [eax+0x60bab], 0x380
```

This value eventually becomes a USB command to register `0x10`. In readable form:

```c
// Set RX Control Register (0x10) to 0x0388:
//   SO=1 (Start Operation), AB=1 (Accept Broadcast), MFB=11b (16KB burst)
// But on AX88772B, MFB bits become RH1M=1, RH2M=1, breaking RX!
asix_write_cmd(0x10, 0x0388, 0, 0, NULL);
```

The problem is that `0x0300` in the upper bits, which sets MFB on the AX88772A, gets reinterpreted as RH1M and RH2M on the AX88772B, enabling the new RX header format that the driver doesn't understand.

As pointed out in ❶, that `0x0300` shouldn't be there for AX88772B compatibility. We need to patch this value to `0x0088` (clearing the upper byte) so that RH1M and RH2M stay zero, keeping the RX header format compatible with what the driver expects.

### The Reset Timing Problem

Then, somewhere in the same function is the reset sequence, a sequence of USB commands to the Software Reset Register that brings up the link:

```asm
; SW_RESET ← 0x40 (IPPD) Power down internal PHY
000133a0  call  asix_write_cmd
000133a5  push  10000
000133aa  call  [NdisMSleep]

; SW_RESET ← 0x00 Clear all bits  
00013423  call  asix_write_cmd
00013428  push  60000              ; Problem: should be 600000!
0001342d  call  [NdisMSleep]

; SW_RESET ← 0x28 (IPRL) Release PHY from reset
00013578  call  asix_write_cmd
```

Curiously, the reset sequence here is already shorter than the original AX88772A datasheet specifies: 60 ms (60000 µs) while 160 ms was recommended. And then the AX88772B datasheet wants it to be 600 ms (600000 µs). It's pretty clear that the `60000` microseconds is just about 10 times too small.

**The fix:** An easy patch: just change `60000` to `600000` in the binary. Find the bytes `60 EA 00 00` (60000 in little-endian) and replace with `C0 27 09 00` (600000 in little-endian).

---

## Patch #2: The Hardware Reset Function

There's one particular function that seems to handle the PHY reset and powerup, we'll call it `asix_hw_reset` because it involves low-level PHY powerup. It also writes the PHY's "medium mode" or link type, e.g. 10BASE-T, 100BASE-TX, Full-duplex, Half-duplex.

Looking at the open-source drivers, this is usually the place where the undocumented `AX_QCTCTRL` register at `0x2A` from ❷ is found, so likely the place where we should add it as well. This register apparently controls the buffer queue settings, allowing the AX88772B to use AX88772A-compatible packet buffering behavior. Without setting this register, the chip may use different internal buffering that breaks compatibility with the older driver.

The best place to add it, is at the end of the function. Unfortunately... there's no space to insert instructions (!) and as you can see in the instruction listing above for the USB commands, we need quite a bit of space.

### Making Space

Here's a trick we can use. Looking at the last few instructions in this function firing off a `asix_write_cmd`:

```asm
0001168a  mov  byte ptr [esi+0x60a55], 0xa
00011691  mov  word ptr [esi+0x60a56], bx
00011698  mov  word ptr [esi+0x60a58], bx
0001169f  mov  dword ptr [esi+0x60a20], ebx
000116a5  mov  dword ptr [esi+0x60a30], ebx
000116ab  call asix_write_cmd
```

We can replace these last few instructions with shorter but identical equivalents, just by juggling different registers:

```asm
0001168a  mov  byte ptr [esi+0x60a55], 0xa
00011691  mov  dword ptr [esi+0x60a56], ebx   ; Combines two MOVs
00011697  mov  dword ptr [esi+0x60a20], ebx
0001169d  mov  dword ptr [esi+0x60a30], ebx
000116a3  call asix_write_cmd
```

Same thing, but one instruction less. This leaves about 8 bytes, not enough space to insert our USB command, but just enough room to insert a `CALL` to somewhere else:

```asm
000116a8  call patch1
```

### Scavenging Dead Code

The hunt then goes on to find a piece of empty code section to insert our 75 bytes worth of instructions for a single USB command to `AX_QCTCTRL`. Unfortunately, there's barely empty space at all.

So instead, it's possible to resort to something else: we try to find a piece of code, a branch of instructions, that isn't ever used and is big enough to contain our patch. Luckily, in the `asix_miniport_initialize` function from earlier, there's a huge code branch apparently meant for some odd configuration that we can cordon off. Here's the original conditional:

```asm
0001322c  cmp  word ptr [eax+0x60b90], 0x10
000132a6  jnz  LAB_00013486
```

This particular value at `[eax+0x60b90]` seems to indicate if we're dealing with a device with an Internal PHY (as is the case) or an external PHY. This will never jump. And the branch that it would never jump to, is huge: 203 bytes. This is plenty of space to create a place where we can insert our precious instructions. So let's steamroll over this `JNZ` and pave it with `NOP`s:

```asm
000132a6  nop
000132a7  nop
000132a8  nop
000132a9  nop
000132aa  nop
000132ab  nop
```

The entire range from `0x13486` to `0x13551` is now available to us, to make the function `patch1` we `CALL`ed to earlier to set up the `AX_QCTCTRL` register at `0x2A` properly:

```asm
patch1:
00013486  push edi
00013487  push esi
00013488  mov  word ptr [esi+0x60a0e], 0x17
00013491  mov  word ptr [edi], 0x50
00013496  mov  dword ptr [esi+0x60a24], ebx
0001349c  mov  dword ptr [esi+0x60a2c], ebx
000134a2  mov  dword ptr [esi+0x60a28], ebx
000134a8  mov  byte ptr [esi+0x60a54], bl
000134ae  mov  byte ptr [esi+0x60a55], 0x2a      ; AX_QCTCTRL command
000134b5  mov  dword ptr [esi+0x60a56], 0x80018000
000134bf  mov  dword ptr [esi+0x60a20], ebx
000134c5  mov  dword ptr [esi+0x60a30], ebx
000134cb  call asix_write_cmd
000134d0  ret
```

This follows the same conventions from earlier: pushing `ESI` and `EDI` as arguments for `asix_write_cmd` and setting up the USB command structure using `MOV`s. The calling convention of this patch function is terrible, but it works. After all, its only purpose is to be called at one place and one place only: at the end of `asix_hw_reset`.

---

## Patch #3: Where Packets Go to Die

This leaves us with one more problem to solve: the broken RX header length bits in ❹. Even after having patched all of the above, things still wouldn't work, and the RX header turned out to be most likely culprit.

Somewhere in the driver, a function could be found that seemed to be handling a queue, incrementing counters, juggling bits and then sending things along. It seemed a likely candidate for either an RX or TX function.

What gave it away as the RX function handling RX data was a `0xFFFE` bitwise AND operator and two variables being compared to `0xFFFF`. These were constants also found in the Linux driver's RX functions:

```asm
0001112c  movzx ecx, word ptr [ecx+0x2]
00011137  add   ecx, edx
0001113f  cmp   ecx, 0xffff
00011145  jnz   LAB_00011196
```

In more readable form:

```c
// Add packet length and its one's complement
if (packet_length + ones_complement_length != 0xFFFF) {
    // Header validation failed: corrupted packet, exit loop
    break;
}
```

All of this is sitting inside a loop (hence the `JNZ`) and is pretty much a smoking gun: drop out of the loop if these two values don't add up to `0xFFFF`. This matches the AX88772A's 16-bit packet length error check with its one's complement value.

### The Fix

First, these values need to be masked with `0x7FF` to make sure only the first 11 bits are ever used, and the packet length isn't getting messed up by those higher 5 bits that are something else entirely in the AX88772B.

Though, the effect of this masking is that we have to change the `CMP` constant of `0xFFFF` to `0x7FF` as well:

```asm
0001113f  cmp  ecx, 0x7ff
```

That leaves us with the masking. There's one problem: no space to insert instructions. Luckily, we still have plenty of space left after our last `patch1` function. The approach here is similar but different: we cut out a few instructions to make space for a `CALL`, paste those instructions in our patch function, and then add the instructions we actually need.

Tracing back where the packet length is first being read out (`ECX` points to the packet length after this):

```asm
0001110d  movzx ecx, bx
00011110  add   ecx, edx
```

A little bit of cutting, as the `CALL` takes up exactly the same space:

```asm
0001110d  call  patch2
```

And then pasting those instructions at our new call destination immediately after the `RET` from `patch1`:

```asm
patch2:
000134d1  movzx ecx, bx
000134d4  add   ecx, edx
```

And the instructions that we actually need to mask the packet length (still pointed to by `ECX`):

```asm
000134d6  and   word ptr [ecx], 0x7ff
000134db  and   word ptr [ecx+0x2], 0x7ff
000134e1  ret
```

And that's it, our packet length should be good to go.

---

## The Complete Patch

Of course, something like Ghidra is a tremendous help to reassemble the instructions and find the right opcodes. But that's really all the patching that's needed. And it's easy too: just replace the opcodes with the right ones and save the file. No checksums. No driver signing. Remember this is Windows 98 Second Edition, after all.

Here's a summary of all the patches applied:

| Patch | Location | Change | Purpose |
|-------|----------|--------|---------|
| INF | Multiple | `772A` → `772B` | Allow driver to match AX88772B device |
| RX_CTL | `0x13156` | `0x0380` → `0x0080` | Clear RH1M/RH2M bits for compatible RX format |
| Sleep | `0x13428` | `60000` → `600000` | Increase PHY reset wait time to 600ms |
| QCTCTRL | `0x116A8` | Insert `CALL` | Add buffer queue configuration command |
| patch1 | `0x13486` | New code | AX_QCTCTRL command (0x2A = 0x8000/0x8001) |
| Dead code | `0x132A6` | `JNZ` → `NOP×6` | Disable dead code branch to make room |
| RX fixup | `0x1110D` | Insert `CALL` | Redirect to patch2 for length masking |
| patch2 | `0x134D1` | New code | Mask RX length to 11 bits |
| Validation | `0x1113F` | `0xFFFF` → `0x07FF` | Fix header validation for 11-bit length |

And with that, the AX88772B works on Windows 98 Second Edition. Time to browse the web at a blistering 100 Mbps... or whatever speed this little device will give you, which turns out to be around 3–6 MB/s for large files. But hey, it works!

👾 Happy hacking.

**Find the accompanying Win9x driver here: [ijsf/AX88772B_WinME98SE](https://github.com/ijsf/AX88772B_WinME98SE).**

© 2026