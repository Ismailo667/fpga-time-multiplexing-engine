# FPGA Time-Multiplexing Engine

A hardware hypervisor that runs a design larger than the chip by paging
blocks in and out of a single reconfigurable zone at runtime.

A large RTL design is cut into sequential blocks that run one after another
in the same reconfigurable region. Each block runs, saves its state into a
small static memory, and is then physically overwritten by the next block,
which picks that state back up and continues. The chip behaves as if it were
larger than it is by spreading the design across time instead of space.

## Highlights

* Pure fabric finite state machine orchestrates the whole swap. No CPU, no
  soft core, no operating system.
* Runs on a bare Artix-7 (Digilent Basys 3), with no processing system, no
  PCAP and no external memory. Reconfiguration goes through the internal
  ICAPE2 configuration port, driven directly from logic.
* Partial bitstreams are cached in on-chip block RAM, loaded once over UART.
* A static context memory, protected by a hardware isolation barrier,
  survives being surrounded by reconfiguration so state crosses each swap
  intact.

## Status

Proof of concept validated on silicon: a block writes a tag into the context
memory, the next block reads it back after the swap, repeated across hundreds
of thousands of alternations with no corruption.

## Source

This repository is a public overview. The RTL source is kept private and is
available on request.
