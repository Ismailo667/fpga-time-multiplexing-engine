# Architecture overview

Four pieces cooperate:

1. **Static hypervisor** a small state machine that runs the active block,
   isolates the memory boundary, triggers reconfiguration and wakes the next
   block, with guard cycles on each side of the swap.
2. **Bitstream cache** on-chip block RAM holding the partial bitstreams,
   filled over UART from a host script.
3. **ICAPE2 streamer** an AXI-Stream front end to the configuration port,
   including the byte level bit ordering the port requires.
4. **Context mailbox and bus decoupler** a static dual-port memory that
   survives the swap, guarded by an isolation barrier so the reconfiguring
   region cannot corrupt it.

## The freeze-time trade-off

On this family the configuration memory has no shadow register, so writing a
block overwrites the active fabric and computation must pause for the whole
transfer. The pause scales with how much fabric is reconfigured, which sets a
hard trade-off between full-chip blocks and fast swaps. Amortizing the swap
with batched work is the practical lever on this silicon.
