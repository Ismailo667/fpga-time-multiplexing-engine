# FPGA Time-Multiplexing Engine

> A hardware hypervisor that runs a design larger than the chip by paging
> blocks of logic in and out of a single reconfigurable zone at runtime.

![status](https://img.shields.io/badge/status-proof%20of%20concept-success)
![target](https://img.shields.io/badge/target-Artix--7%20XC7A35T-blue)
![language](https://img.shields.io/badge/HDL-SystemVerilog-orange)
![license](https://img.shields.io/badge/license-MIT-lightgrey)

## Table of contents

- [What it does](#what-it-does)
- [Why it is hard: no SoC](#why-it-is-hard-no-soc)
- [Architecture](#architecture)
- [Execution flow](#execution-flow)
- [The core problem: freeze time](#the-core-problem-freeze-time)
- [Validation](#validation)
- [Hardware and tools](#hardware-and-tools)
- [Roadmap](#roadmap)
- [Source](#source)

## What it does

Some RTL designs are simply too big for the FPGA you can afford. The usual
answer is to buy a bigger part. This project asks a different question: what
if a large design is cut into sequential blocks `B1 -> B2 -> ... -> Bn`, and
those blocks take turns running in the *same* reconfigurable region, paging
silicon in and out the way an operating system pages memory?

Each block runs, saves its outputs and state into a small static memory, and
is then physically overwritten by the next block. The incoming block reads
that state back and continues the computation. The chip behaves as if it were
larger than it physically is, by spreading the design across time instead of
space. This is hardware virtualization by time-multiplexing.

The engine that makes this safe and automatic is a hardware hypervisor: a
pure logic sequencer that decides when to freeze the active block, isolate
shared memory, reprogram the region, and wake the next block.

## Why it is hard: no SoC

The comfortable way to do runtime reconfiguration is on a SoC-class FPGA,
where a hardened CPU drives a dedicated configuration port (PCAP) and a DMA
engine streams bitstreams from external DRAM. This project deliberately runs
on none of that. The target is a bare Artix-7 on a Digilent Basys 3: the
smallest and slowest speed grade, no processing system, no external memory.

Every part of the hypervisor therefore had to be built in fabric logic:

| Usual SoC approach            | What this project does instead                 |
| ----------------------------- | ---------------------------------------------- |
| CPU or soft core sequences it | Pure hardware finite state machine, no CPU     |
| PCAP configuration port       | Internal ICAPE2 port, driven directly by logic |
| Bitstreams streamed from DRAM | Partial bitstreams cached in on-chip block RAM |
| Vendor reconfiguration IP     | Custom SystemVerilog, built from scratch       |

Nearly every design decision below is a consequence of removing the
processing system and rebuilding its job in raw logic.

## Architecture

```
        UART (host loader)
             |
             v
   +---------------------+        +------------------------+
   |  Bitstream cache    |        |  Static hypervisor FSM |
   |  (on-chip BRAM)     |<------>|  run / isolate / swap  |
   +----------+----------+        +-----------+------------+
              |                               |
              v                               v
   +---------------------+        +------------------------+
   |  ICAPE2 streamer    |------->|  ICAPE2 config port    |
   |  (AXI-Stream front) |        |  rewrites the region   |
   +---------------------+        +-----------+------------+
                                              |
                                              v
   +-------------------------------------------------------+
   |  Reconfigurable region  (B1, then B2, ...)            |
   |         |                            ^                |
   |         v   writes state   reads state|               |
   |  +------------------------------------------+         |
   |  |  Context mailbox + bus decoupler (static)|         |
   |  |  survives the swap, isolation barrier    |         |
   |  +------------------------------------------+         |
   +-------------------------------------------------------+
```

Four pieces cooperate:

1. **Static hypervisor.** A small finite state machine, with no CPU involved,
   that runs the active block, locks the memory boundary, triggers
   reconfiguration, and wakes the next block, with guard cycles on each side
   of the swap to let pipelines drain and the new logic settle.
2. **Bitstream cache.** On-chip block RAM that holds the partial bitstreams,
   filled once over UART from a host script. This replaces the DRAM a SoC
   would stream from.
3. **ICAPE2 streamer.** An AXI-Stream front end to the internal configuration
   port, including the byte-level bit ordering the port requires and live
   monitoring of the port status flags.
4. **Context mailbox and bus decoupler.** A static dual-port memory that is
   never reconfigured, so it persists across swaps. A single isolation signal
   forces the reconfigurable region's memory accesses off in hardware during
   the swap, so a half-configured region cannot corrupt shared state with
   glitches.

## Execution flow

The hypervisor walks a short cycle of states:

```
RUN_BLK1  ->  FENCE_ISOLATE  ->  ICAP_STREAM  ->  DESYNC_SETTLE  ->  RUN_BLK2
  ^                                                                      |
  +----------------------------------------------------------------------+
```

1. **RUN_BLK1** the active block computes and writes its state to the mailbox.
2. **FENCE_ISOLATE** the memory boundary is locked and the write pipeline is
   allowed to drain, so no partial write is in flight when the region changes.
3. **ICAP_STREAM** the next block's partial bitstream is pushed from the cache
   into the configuration port, overwriting the region.
4. **DESYNC_SETTLE** guard cycles let the freshly configured logic stabilize.
5. **RUN_BLK2** isolation is released and the new block starts, reading the
   mailbox state left by its predecessor.

## The core problem: freeze time

On this FPGA family the configuration memory has no shadow register, so there
is no double buffering at the cell level. The moment a frame is written
through the configuration port, it overwrites the active fabric. Computation
must therefore be frozen for the entire duration of the transfer.

The length of that freeze is just bitstream size divided by configuration
bandwidth. A configuration frame on this family is 101 words of 32 bits, and
the port is 32 bits wide at a nominal 100 MHz, which is 400 MB/s. Overclocking
the port, well documented in the fast-reconfiguration literature, can roughly
double that before it saturates.

The uncomfortable part is that the freeze scales with how much fabric is
reprogrammed, and the whole point of this project is to let each block use as
much of the chip as possible. For a near-full-chip region the freeze is in the
millisecond range:

| Configuration clock | Bandwidth | Freeze for a full-chip region |
| ------------------- | --------- | ----------------------------- |
| 100 MHz (nominal)   | 400 MB/s  | around 134 µs                   |
| 200 MHz (overclock) | 800 MB/s  | around 66 µs                 |

So there is a hard trade-off: you can have whole-chip blocks or you can have
fast swaps, but not both at once on this silicon. The practical lever is to
batch enough work per block to amortize the swap, or to shrink what actually
changes per swap by factoring a common static shell out of the reconfigurable
core. Naming this limit honestly is part of the project.

## Validation

The proof of concept was validated on real hardware, running in an infinite
loop:

- **Functional test.** One block writes an integrity tag into the context
  memory, the next block reads it back after the swap and checks it. This ran
  across hundreds of thousands of alternations with no memory corruption, so
  the static mailbox genuinely survives being surrounded by reconfiguration.
- **Stress test.** The swap was driven at maximum rate, and a batch mode
  processed a full memory of words per block to measure the amortized
  throughput against the freeze cost.

## Hardware and tools

- **Board:** Digilent Basys 3
- **FPGA:** Xilinx Artix-7 XC7A35T, speed grade -1, no external DRAM
- **HDL:** SystemVerilog
- **Flow:** Vivado non-project (TCL) with a Dynamic Function eXchange design
- **Host loader:** Python over USB-UART

## Roadmap

- Close the reconfiguration loop on the port status flags, so a swap is only
  declared done when the silicon confirms a clean configuration rather than
  when the last word has been sent.
- Add a calibrated overclock that uses those same status flags as a live error
  monitor to find the real ceiling of a given die, then back off with margin.
- Use the vendor bitstream compression, which collapses repeated frames, to
  cut the transfer size of a sparsely used region.
- Factor a common static shell out of the blocks so that only the functional
  core is reconfigured, shrinking the freeze.

## Source

This repository is a public overview of the project. The RTL source is kept
private and is available on request.
