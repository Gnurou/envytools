.. _hwsq:

==================
Hardware sequencer
==================

.. contents::


Introduction
============

HWSQ is a very limitted programmable "microcontroller" present on NV17:NV20
and NV25:GF100 cards. Its capabilities include: setting/clearing several
register bits all around the card, waiting a fixed amount of time, waiting
for one of a small predefined set of events [NV41+ only], and writing
arbitrary values to arbitrary MMIO registers [NV41+ only].

There is no control flow, no GPRs, no arithmetic. It's only possible to
execute a very simple preconfigured series of commands from start to finish.

Most HWSQ registers reside inside of :ref:`PBUS MMIO area <pbus-mmio>`.


.. _hwsq-mmio:

MMIO registers
==============

no annotation - NV17:GF100
[1] NV17:NV41
[2] NV41:G80
[3] G80:GF100
[4] G92:GF100

============== ==================== ==========
Address        Variants             Name
============== ==================== ==========
001098 bit 3   NV17:NV20 NV25:GF100 HWSQ_ENABLE
001098 bit 4   NV17:NV20 NV25:GF100 HWSQ_OVERRIDE_MODE
001304         NV17:NV20 NV25:GF100 ENTRY_POINT
001308         NV17:NV20 NV25:GF100 STATUS
00130c         NV17:NV20 NV25:GF100 TRIGGER
001310         NV17:NV20 NV25:GF100 FLAGS_0
001314         NV17:NV20 NV25:GF100 FLAGS_1
001318         G92:GF100            ENTRY_POINT_HIGH
001400:001440  NV17:NV20 NV25:NV41  CODE
001400:001480  NV41:G80             CODE
001400:001500  G80:GF100            CODE
001578         NV41:G80             EVENTS
080000:080200  G92:GF100            NEW_CODE
============== ==================== ==========

.. todo:: are EVENTS variants right?

.. todo:: cleanup, crossref

.. space:: 8 phwsq-large-code 0x1000 extended HWSQ code space

   .. todo:: writ me


Code space
==========

The HWSQ commands are stored in dedicated code RAM. The code RAM is 0x40
bytes long on NV17:NV41 cards, 0x80 bytes long on NV41:G80, 0x100 bytes long
on G80:G92, and 0x200 bytes long on G92+.

The code RAM is byte-oriented, but the MMIO registers used to access it are
word-oriented, and touch 4 bytes at once. They are treated as little-endian:
bits 0-7 of the word touch byte 0, bits 8-15 touch byte 1, 16-23 touch byte 2,
24-31 touch byte 3.

The code RAM is direct-mapped to MMIO space. On all cards, accessing a word
at MMIO 0x1400 + i, where 0 <= i < 0x100, i < code RAM size, and i is
divisible by 4, accesses bytes i..i+3 of code RAM. On G92+, due to
increased code RAM size, a new code RAM access window has been introduced
at MMIO 0x080000: MMIO 0x80000 + i, where 0 <= i < 0x200 and i is divisible
by 4, accesses bytes i..i+3 of code RAM. The old 0x1400 window still exists,
but is limitted to first 0x100 bytes of code RAM.

MMIO 0x1400 + [0..0xf] * 4: CODE [NV17:NV41]
MMIO 0x1400 + [0..0x1f] * 4: CODE [NV41:G80]
MMIO 0x1400 + [0..0x3f] * 4: CODE [G80:GF100]
MMIO 0x80000 + [0..0x7f] * 4: NEW_CODE [G92:GF100]
  Index i accesses code RAM bytes i*4, i*4+1, i*4+2, i*4+3, mapped to bits
  0-7, 8-15, 16-23, 24-31 respectively.


Opcodes
=======

The HWSQ opcodes are variable-length, most are a single byte. The first byte
determines the length of the opcode. The valid opcodes [or their first bytes]
are:

- 0x00-0x3f: wait - waits a fixed amount of PTIMER clocks [single byte]
- 0x40: addrlo - sets low 16 bits of MMIO address and executes MMIO write [3 bytes] [NV41+ only]
- 0x42: datalo - sets low 16 bits of MMIO data [3 bytes] [NV41+ only]
- 0x5f: ewait - waits for an event [3 bytes] [NV41+ only]
- 0x7f: exit - finishes HWSQ execution [single byte]
- 0x80-0x9f: unset - sets a flag to "don't touch" [single byte]
- 0xa0-0xbf: set1 - sets a flag to "override to 1" [single byte]
- 0xc0-0xdf: set0 - sets a flag to "override to 0" [single byte]
- 0xe0: addr - sets MMIO address and executes MMIO write [5 bytes] [NV41+ only]
- 0xe2: data - sets MMIO data [5 bytes] [NV41+ only]


Script execution
================

HWSQ execution has to be started manually by the host every time the script
is to be executed. The execution can begin at arbitrary point and continues
until the "exit" opcode is executed.

First, the entry point needs to be set. There are 4 entry points, selectable
when triggering the execution start. They are set through the ENTRY_POINT
registers:

MMIO 0x001304: ENTRY_POINT
  - bits 0-7: bits 0-7 of entry point 0 address
  - bits 8-15: bits 0-7 of entry point 1 address
  - bits 16-23: bits 0-7 of entry point 2 address
  - bits 24-31: bits 0-7 of entry point 3 address

MMIO 0x001318: ENTRY_POINT_HIGH [G92:GF100]
  - bit 0: bit 8 of entry point 0 address
  - bit 8: bit 8 of entry point 1 address
  - bit 16: bit 8 of entry point 2 address
  - bit 24: bit 8 of entry point 3 address

Once entry points are set and the code is uploaded, scripts can be started by
poking the TRIGGER register. The NV17:G92 HWSQ hardware has support for two
HWSQ "execution slots", with independent instruction pointers. However, they
have no support for concurrent execution: a long wait on one of the slots will
also hang the other. G92+ have only one execution slot.

MMIO 0x00130c: TRIGGER [write only]
  - bit 0: trigger type. 0 aborts execution, 1 starts execution.
  - bit 1: slot. 0 means slot B, 1 means slot A [NV17:G92 only]
  - bits 2-3: entry point selection [for start trigger only]

Execution status can be monitored through the STATUS register:

MMIO 0x001308: STATUS [read only]
  - bits 0-7: bits 0-7 of current slot A IP [instruction pointer]. The IP is the
    address of the *next* instruction to be fetched, so if HWSQ is
    currently executing a wait opcode, this will point to the byte
    after the opcode. After the script exits normally, it'll point
    to the exit instruction - exit doesn't increase the IP.
  - bit 8: 1 if HWSQ slot A is currently executing, 0 if not
  - bit 9: 1 if HWSQ slot A encountered an unknown opcode [NV41:G92 only]
  - bit 10: bit 8 of current slot A IP [G92+ only]
  - bits 16-31: like bits 0-15, but for slot B [NV17:G92 only]

When HWSQ hits an unknown opcode on NV41:G92 cards, the "illegal opcode" bit
in STATUS register is lit, and the execution hangs. The HWSQ slot is still
considered executing, however, and needs to be manually aborted. On NV17:NV41
and G92:GF100, unknown opcodes are treated as 1-byte nops.

HWSQ execution can end by hitting an "exit" opcode, or manual abort. The exit
opcode is:

Opcode: 0x7f - exit
  Stops HWSQ execution on a given slot. IP is not incremented.

Manual abort is executed by poking the TRIGGER register with type set to
abort. Note that, in some cases on wait instructions, the abort triggers
an unknown opcode condition and the script hangs instead - a second abort
is needed to clear the unknown opcode condition and actually abort the
execution.


PTIMER wait opcodes
===================

The PTIMER wait opcodes are used to insert constant delays into a script. The
delays are selectable by a simple encoding style, and are counted in PTIMER
clocks. A PTIMER clock here is considered to be the actual clock at which the
TIME_* registers are increased - ie. a single clock is what causes the TIME_*
registers to increase by 0x20.

The opcodes are:

Opcode: 0x00-0x3f - wait #wait_length shl #(wait_shift * 2)
  - opcode bits 0-1: wait_length [0-3]
  - opcode bits 2-5: wait_shift [0-15] - written in the assembler instruction
    premultiplied by 2.
 
  Delays next HWSQ opcode execution by (wait_length << (wait_shift * 2)) * 0x20
  PTIMER clocks. If PTIMER uses standard calibration values, this corresponds
  to (wait_length << (wait_shift * 2)) µs.


Flags
=====

The main purpose of HWSQ on pre-NV41 GPUs is to twiddle various bits in
registers all around the card. They're called "HWSQ flags". There are 32
flags. A flag can be in one of 3 states:

- unset: the value of corresponding register bit is unaffected
- override to 0: the value of corresponding register bit is forced to 0, and
  cannot be changed by normal means
- override to 1: the value of corresponding register bit is forced to 1, and
  cannot be changed by normal means

The current state of HWSQ flags can be accessed by the FLAGS registers:

MMIO 0x001310: FLAGS_0
  - bits 0-15: values of flags 0-15. If override is enabled for a flag, this is
    what the corresponding register bit should be forced to.
    Otherwise, it is ignored.
  - bits 15-31: override enables of flags 0-15. If the bit corresponding to
    a given flag is set, the flag is in one of the override states,
    otherwise it's in the unset state.

MMIO 0x001314: FLAGS_1
  Like FLAGS_0, but for flags 16-31.

The flags state can be modified from HWSQ scripts by using one of the flag
opcodes:

Opcode: 0x80-0x9f - unset #flag
  - bits 0-4: flag index
 
  Switches the selected flag to "unset" state.

Opcode: 0xa0-0xbf - set1 #flag
  - bits 0-4: flag index

  Switches the selected flag to "override to 1" state.

Opcode: 0xc0-0xdf - set0 #flag
  - bits 0-4: flag index

  Switches the selected flag to "override to 0" state.

The flag behavior is additionally controlled by two bits in PBUS.DEBUG_6
register:

MMIO 0x001098 bit 3: HWSQ_ENABLE
  When set to 1, flag overrides and MMIO accesses will work. When set to 0,
  HWSQ programs will execute, but flag overrides will be ignored, and MMIO
  accesses will hang until HWSQ_ENABLE is set to 1.

MMIO 0x001098 bit 4: HWSQ_OVERRIDE_MODE
  Selects the value that will be returned when reading register bits
  overriden by HWSQ flags. Values:

  - 0: READ_NORMAL - the value read from the overriden register by MMIO will
    be the pre-override value. However, the overriden values will be
    still used internally by hw.
  - 1: READ_OVERRIDE - the value read from the overriden register by MMIO
    will be the one provided by HWSQ.

The known flags are:

- 0: :ref:`60081c/60281c/CR4d b0 [NV17:G80] <nv10-gpio-lines>`
- 1: :ref:`60081c/60281c/CR4d b1 [NV17:G80] <nv10-gpio-lines>`
- 2: :ref:`60081c/60281c/CR4d b4 [NV17:G80] <nv10-gpio-lines>`
- 3: :ref:`60081c/60281c/CR4d b5 [NV17:G80] <nv10-gpio-lines>`
- 4: :ref:`680880 b28 [NV17:NV40] <pramdac-mmio>`
- 5: :ref:`682880 b28 [NV17:NV40] <pramdac-mmio>`
- 6: :ref:`680880 b29 [NV17:G80] <pramdac-mmio>`
- 7: :ref:`682880 b29 [NV17:G80] <pramdac-mmio>`
- 14: :ref:`60081c/60281c b28 [NV31:G80] <nv10-gpio-lines>`
- 15: :ref:`60081c/60281c b29 [NV31:G80] <nv10-gpio-lines>`
- 16: FB_PAUSE [NV41-] [see below]
- 25: :ref:`15fc b31 [NV41:G80] <nv10-gpio-pwm>`
- 26: :ref:`15f4 b31 [NV41:G80] <nv10-gpio-pwm>`
- 27: :ref:`10f0 b31 [NV17:G80] <nv10-gpio-pwm>`
- 28: 1084 b22 [NV17:G80]
- 29: 1084 b24 [NV17:G80]
- 30: 1084 b26 [NV17:G80]
- 31: 1084 b27 [NV17:G80]

.. todo:: 8, 9, 13 seem used by microcode!
.. todo:: check variants for 15f4, 15fc
.. todo:: check variants for 4-7, some NV4x could have it
.. todo:: check variants for 14, 15
.. todo:: doc 1084 bits


MMIO poke opcodes
=================

On NV41+ cards, HWSQ can write arbitrary values to arbitrary MMIO addresses.
This is done in two parts: first, the data value has to be set with one of
the "set data" opcodes, then the MMIO address should be set using the "set
address" opcode. The opcode setting the address also triggers the MMIO write.

If a script writes multiple MMIO registers, it may make use of the "short"
data and address opcodes. They take a 16-bit parameter, filling the high 16
bits with the high 16 bits of previously used value.

Thus, there are 2 32-bit state registers used for MMIO poke opcodes: ADDR
and DATA. Both of these registers are per-slot on GPUs that have two
executions slots. These registers are not directly accessible through MMIO.

Opcode: 0xe2 <b0:7> <b8:15> <b16:23> <b24:31> - data #imm32 [NV41+]
  This is a 5-byte opcode. First byte is the actual opcode, while the next
  bytes specify the 32-bit immediate value.

  ::

	DATA = imm32;

Opcode: 0x42 <b0:7> <b8:15> - datalo #imm16 [NV41+]
  This is a 3-byte opcode. First byte is the actual opcode, while the next
  bytes specify the 16-bit immediate value.
  DATA = (DATA & 0xffff0000) | imm16;

Opcode: 0xe0 <b0:7> <b8:15> <b16:23> <b24:31> - addr #imm32 [NV41+]
  This is a 5-byte opcode. First byte is the actual opcode, while the next
  bytes specify the 32-bit immediate value.

  ::

	ADDR = imm32;
	MMIO_WR32(ADDR, DATA);

Opcode: 0x40 <b0:7> <b8:15> - addrlo #imm16 [NV41+]
  This is a 3-byte opcode. First byte is the actual opcode, while the next
  bytes specify the 16-bit immediate value.

  ::

        ADDR = (ADDR & 0xffff0000) | imm16;
        MMIO_WR32(ADDR, DATA);

For the addr and addrlo instructions to work, and the pokes to be executed,
HWSQ_ENABLE bit has to be set to 1.


Events
======

On NV41+ cards, HWSQ can wait for certain events to happen [in addition to
plain time-based waits from older cards]. An event is a 1-bit signal coming
from some part of the GPU. There can be up to 32 events, depending on the GPU.
The current state of all events can be read by the EVENTS register:

MMIO 0x001578: EVENTS [NV41-]
  - bits 0-31: values of events 0-31.

The events can be waited on from HWSQ scripts by using the ewait opcode. HWSQ
can wait for both a 0 value and a 1 value on events.

Opcode: 0x5f <e> <v> - ewait #event #value
  This is a 3-byte opcode. First byte is the actual opcode, second byte
  specifies the event to wait on, third byte specifies the value to wait on.
  Delays next HWSQ opcode execution until given event has the given value.

The events are:

- 0: FB_PAUSED [see below]
- 1: CRTC0_VBLANK [:ref:`nv41 <pcrtc-blank>`, :ref:`G80 <pdisplay-blank>`] 
- 2: CRTC0_HBLANK [:ref:`nv41 <pcrtc-blank>`, :ref:`G80 <pdisplay-blank>`] 
- 3: CRTC1_VBLANK [:ref:`nv41 <pcrtc-blank>`, :ref:`G80 <pdisplay-blank>`] 
- 4: CRTC1_HBLANK [:ref:`nv41 <pcrtc-blank>`, :ref:`G80 <pdisplay-blank>`] 


Framebuffer pause feature
=========================

One purpose of HWSQ is memory reclocking. Memory reclocking can only be done
reliably if noone accesses the memory while it's being reclocked. Thus HWSQ
can request memory accesses to be blocked for a while. This is done by
flag #16, FB_PAUSE. This pause functionality is present on NV41:GF100 cards.

The FB_PAUSE flag, as opposed to all other flags, doesn't override any actual
register bitfield - the signal it controls can only be set via that flag.
Framebuffer pause is thus active iff flag #16 is in "override to 1" state.
FB_PAUSE is also unaffected by the HWSQ_ENABLE bit.

Framebuffer pausing doesn't work immediately and the HWSQ has to wait until
the framebuffer is actually paused. The FB_PAUSED event is provided for that.
The FB_PAUSED event is set to 1 iff framebuffer pause has been requested by
HWSQ and completed by memory controller.

On NV41:G80, framebuffer pause will indefinitely block all accesses to memory
until it's unpaused. This includes accesses from the host via BAR1, BAR2, 
PEEPHOLE, and the PRAMIN range.

On G80+, framebuffer pause not only blocks memory accesses, it additionally
blocks all accesses to the GPU from host - including MMIO accesses.
