# NetworkDirect SPI Tutorial

A friendlier, task-oriented tour of the NetworkDirect (ND) v2 SPI. If the
[reference docs](../NetworkDirectSPI.md) feel like a wall of method signatures,
start here.

## Who this is for

You have a Windows machine with an RDMA-capable adapter (InfiniBand, iWARP,
or RoCE) and you want to write an application that uses kernel-bypass
networking to move data with the lowest possible latency and CPU cost.

You do **not** need to know:

- How to write a provider DLL — this tutorial is about *using* the SPI.
- The InfiniBand verbs API — we point out parallels where useful, but no
  prior RDMA experience is assumed.

You **do** need:

- Comfort with C/C++ on Windows.
- Familiarity with Win32 overlapped I/O and COM-style reference counting
  (`AddRef`/`Release`). The ND SPI uses both heavily.

## How to read this

The pages are designed to be read in order the first time. After that they
work as independent references.

| # | Page | What you learn |
|---|------|----------------|
| 0 | [Getting started](./01-getting-started.md) | Mental model, the 8 setup steps, build & run the smallest working client/server. |
| 1 | [Objects & lifecycles](./02-objects-and-lifecycles.md) | Every ND interface, how they relate, and the order to create and destroy them. |
| 2 | [Connection establishment](./03-connections.md) | Listener / connector handshake, private data, read limits, disconnect. |
| 3 | [Data transfer](./04-data-transfer.md) | Send/Receive vs. RDMA Read/Write, SGEs, inline, fences, silent success. |
| 4 | [Asynchronous completions](./05-completions-and-async.md) | Overlapped I/O, polling vs. notify, completion queues, IOCP integration. |
| 5 | [Memory regions & windows](./06-memory.md) | Registering buffers, access flags, exchanging tokens, memory windows. |
| 6 | [Advanced patterns](./07-advanced-patterns.md) | Credit-based flow control, shared receive queues, pipelining, error handling. |

## How this maps to the reference docs

Every concept in this tutorial links to the corresponding reference page so
you can drill down for an exhaustive parameter list whenever you want.

- Provider discovery & adapters → [IND2Provider](../IND2Provider.md), [IND2Adapter](../IND2Adapter.md)
- Asynchronous primitives → [IND2Overlapped](../IND2Overlapped.md), [IND2CompletionQueue](../IND2CompletionQueue.md)
- Memory → [IND2MemoryRegion](../IND2MemoryRegion.md), [IND2MemoryWindow](../IND2MemoryWindow.md)
- Connection setup → [IND2Listener](../IND2Listener.md), [IND2Connector](../IND2Connector.md)
- Data movement → [IND2QueuePair](../IND2QueuePair.md), [IND2SharedReceiveQueue](../IND2SharedReceiveQueue.md)

## How this maps to the examples in this repo

Each tutorial page calls out the example program that demonstrates the
concept end-to-end. All examples live under [src/examples](../../src/examples).

| Example | Demonstrates |
|---------|--------------|
| [ndcat](../../src/examples/ndcat/ndcat.cpp) | Listing every ND-capable IP address on the box. |
| [ndadapterinfo](../../src/examples/ndadapterinfo/ndadapterinfo.cpp) | Opening an adapter and dumping its capabilities. |
| [ndping](../../src/examples/ndping/ndping.cpp) | Two-sided Send/Receive with credit-based flow control. |
| [ndpingpong](../../src/examples/ndpingpong/ndpingpong.cpp) | Round-trip Send/Receive latency. |
| [ndrping](../../src/examples/ndrping/ndrping.cpp) | One-sided RDMA Read or Write throughput. |
| [ndrpingpong](../../src/examples/ndrpingpong/ndrpingpong.cpp) | RDMA Write ping-pong using polled flags in remote memory. |
| [ndmrlat / ndmrrate](../../src/examples/ndmrlat) | Memory-registration cost benchmarks. |
