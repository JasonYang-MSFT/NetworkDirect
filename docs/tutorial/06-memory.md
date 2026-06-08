# Memory Regions & Windows

RDMA hardware reads and writes user memory directly. The OS therefore
has to know up-front which pages are eligible: they must be pinned, the
NIC must hold a translation for them, and the NIC must enforce the
caller's intent (read-only? remote-writable?). NetworkDirect models this
with two interfaces:

- **`IND2MemoryRegion` (MR)** — a long-lived pinned view of one buffer.
- **`IND2MemoryWindow` (MW)** — a re-bindable sub-range of an MR you can
  hand to a peer.

> Reference: [IND2MemoryRegion](../IND2MemoryRegion.md),
> [IND2MemoryWindow](../IND2MemoryWindow.md).

## 1. Mental model

```mermaid
flowchart LR
    Buffer[(Your buffer)] -. Register .-> MR[IND2MemoryRegion]
    MR -.->|GetLocalToken| LocalSGE[ND2_SGE.MemoryRegionToken]
    MR -.->|GetRemoteToken| RemoteRDMA[Peer's Read/Write target]
    MR -. QP.Bind .-> MW[IND2MemoryWindow]
    MW -.->|GetRemoteToken| RemoteRDMA2[Peer's Read/Write target<br/>scoped + revocable]
```

Three takeaways:

- One MR pins one buffer. Re-registering a new buffer needs a new MR.
- The **local token** identifies the region to the NIC for your *own*
  Send / Receive / Read / Write requests.
- The **remote token** is what you give a peer so it can RDMA-Read or
  RDMA-Write your memory. Two ways to get one:
  - From the MR directly — peer can touch the entire region.
  - From an MW bound to a sub-range — peer can touch only that slice,
    and you can revoke access at any time.

## 2. Registering memory

Two steps, just like in [Getting Started](./01-getting-started.md):

```cpp
IND2MemoryRegion* pMr = nullptr;
pAdapter->CreateMemoryRegion(IID_IND2MemoryRegion, hFile,
                             reinterpret_cast<void**>(&pMr));

OVERLAPPED ov{};
ov.hEvent = CreateEvent(nullptr, FALSE, FALSE, nullptr);

ULONG flags = ND_MR_FLAG_ALLOW_LOCAL_WRITE
            | ND_MR_FLAG_ALLOW_REMOTE_WRITE;

HRESULT hr = pMr->Register(buffer, length, flags, &ov);
if (hr == ND_PENDING)
    hr = pMr->GetOverlappedResult(&ov, /*wait=*/ TRUE);
```

After Register completes:

```cpp
UINT32 localToken  = pMr->GetLocalToken();   // for your own SGEs
UINT32 remoteToken = pMr->GetRemoteToken();  // give to the peer
```

### Why a two-step API?

`CreateMemoryRegion` is cheap (just an object allocation). `Register`
might:

- Pin pages in physical memory (allocates the pinning bookkeeping).
- Build a NIC-side address translation table for them.
- Trigger an IOMMU mapping.

Each of those can take real time. That's why `Register` is overlapped —
you can pipeline registrations with other useful work. Larger buffers
mean longer registrations. The [ndmrrate](../../src/examples/ndmrrate)
example measures the cost in registrations-per-second.

### Cap: `MaxRegistrationSize`

Each MR can pin at most `MaxRegistrationSize` bytes (see
[ND2_ADAPTER_INFO](../IND2Adapter.md#nd2_adapter_info-structure)).
Larger buffers need multiple MRs and multiple SGEs. The adapter's
overall pinned-memory budget is also finite — check
`ND_INSUFFICIENT_RESOURCES` and back off.

## 3. The access flags decide what's legal

You set flags **at registration** and they cannot be changed later.

| Flag | Allows |
|------|--------|
| `ND_MR_FLAG_ALLOW_LOCAL_WRITE` | The NIC can DMA *into* the buffer — required for any buffer that will be a Receive target or RDMA-Read sink. |
| `ND_MR_FLAG_ALLOW_REMOTE_READ` | A connected peer can issue RDMA Read against this MR. |
| `ND_MR_FLAG_ALLOW_REMOTE_WRITE` | A connected peer can issue RDMA Write against this MR. |
| `ND_MR_FLAG_RDMA_READ_SINK` | Marks the MR as the destination for *initiator-side* RDMA Read. Required by some adapters when you call `Read` and want results stored here. |
| `ND_MR_FLAG_DO_NOT_SECURE_VM` | Skip `MmSecureVirtualMemory`. Use only when the buffer's lifetime is guaranteed longer than the MR (or when the buffer is AWE memory). |

A short decision table for the most common cases:

| Buffer purpose | Flags |
|----------------|-------|
| Hold your own send payload (Send only) | `0` (you don't need local-write — the NIC reads from it) |
| Hold your own receive payload (Receive target) | `ND_MR_FLAG_ALLOW_LOCAL_WRITE` |
| Sink of an RDMA Read you initiate | `ND_MR_FLAG_ALLOW_LOCAL_WRITE \| ND_MR_FLAG_RDMA_READ_SINK` |
| Source of an RDMA Read the peer initiates | `ND_MR_FLAG_ALLOW_REMOTE_READ` |
| Destination of an RDMA Write the peer initiates | `ND_MR_FLAG_ALLOW_LOCAL_WRITE \| ND_MR_FLAG_ALLOW_REMOTE_WRITE` |

> The `ndrping` server picks its flag set based on which RDMA op the
> client is going to issue — read [its `RunTest`
> method](../../src/examples/ndrping/ndrping.cpp) for the worked
> example.

The peer is responsible for asking for the right operation; mismatches
surface as `ND_ACCESS_VIOLATION` on the initiator's CQ.

## 4. Tokens

Two tokens, two consumers.

### Local token

```cpp
UINT32 localToken = pMr->GetLocalToken();
ND2_SGE sge{ buffer, length, localToken };
pQp->Send(ctx, &sge, 1, 0);
```

Used in *every* SGE you build for *your own* requests (Send, Receive,
Read, Write, Bind). The NIC uses it to validate the virtual address
and pick the right pinning context.

### Remote token

```cpp
UINT32 remoteToken = pMr->GetRemoteToken();
// Marshal {remoteToken, (UINT64)buffer} into a message and Send to the peer.
```

Used by the *peer* when it issues Read/Write against your MR (or MW —
see below). The value is opaque — providers and adapters use whatever
encoding they like. **It should be stored in network byte order if you
ever send it across architectures with different endianness.**

There is no built-in exchange. The standard practice is to put a small
struct into the connector's private data, or to do a single Send right
after CompleteConnect:

```cpp
struct PeerInfo {
    UINT32 remoteToken;
    UINT64 remoteAddress;
};

PeerInfo info{ pMr->GetRemoteToken(), reinterpret_cast<UINT64>(buffer) };
ND2_SGE sge{ &info, sizeof(info), pMr->GetLocalToken() };
pQp->Send(ctx, &sge, 1, ND_OP_FLAG_INLINE);
```

[ndrping](../../src/examples/ndrping/ndrping.cpp) demonstrates this
exact pattern.

## 5. Memory windows: scoped, revocable remote access

If you want to expose only a slice of an MR — or revoke remote access
without deregistering the whole MR — use an MW.

```mermaid
sequenceDiagram
    participant App
    participant QP as IND2QueuePair
    participant MW as IND2MemoryWindow

    App->>App: CreateMemoryRegion + Register (large buffer)
    App->>App: CreateMemoryWindow (state: invalidated)
    App->>QP: Bind(MR, MW, &buf[off], len, ALLOW_WRITE)
    Note over MW: Now valid; remote token usable.
    App->>App: Send remote token + address to peer
    Note over App: Peer issues RDMA Write into MW
    App->>QP: Invalidate(MW)
    Note over MW: Invalidated; remote access denied.
```

Code:

```cpp
IND2MemoryWindow* pMw = nullptr;
pAdapter->CreateMemoryWindow(IID_IND2MemoryWindow,
                             reinterpret_cast<void**>(&pMw));

// Bind a sub-range. This is a CQ-driven operation — its completion
// arrives like any other QP work request.
pQp->Bind(/*ctx*/ nullptr, pMr, pMw,
          buffer + offset, lengthInWindow,
          ND_OP_FLAG_ALLOW_WRITE);

// You can call GetRemoteToken immediately — don't have to wait for
// Bind to complete to obtain the token, only to use it from the peer.
UINT32 mwToken = pMw->GetRemoteToken();
```

When you're done:

```cpp
pQp->Invalidate(/*ctx*/ nullptr, pMw, 0);
```

Important constraints:

- A given MW is bound to **one** QP at a time. Invalidate to re-bind on
  another QP.
- The peer's access ends the moment the Invalidate completes. In-flight
  Read/Write requests against the now-invalid window fail and tear down
  the connection — coordinate the handoff with a message.
- The flags you pass to `Bind` (`ND_OP_FLAG_ALLOW_READ`,
  `ND_OP_FLAG_ALLOW_WRITE`) must be a *subset* of what the underlying
  MR allows (`ND_MR_FLAG_ALLOW_REMOTE_*`).
- The MR must remain registered as long as **any** MW is bound to it —
  `Deregister` returns `ND_DEVICE_BUSY` until you invalidate everything.

When **not** to use an MW:

- The peer always has access to the whole MR. Just hand them the MR's
  remote token — one fewer round trip during connection setup.
- You only do Send/Receive. MWs are exclusively for one-sided RDMA.

The reference doc points out that on InfiniBand devices Type 2 windows
are preferred over Type 1 — that's a provider implementation detail you
don't have to worry about as a consumer.

## 6. Deregistering

`Deregister` is the inverse of `Register`. It is overlapped because
it may take real work to undo the pinning:

```cpp
OVERLAPPED ov{}; ov.hEvent = CreateEvent(nullptr, FALSE, FALSE, nullptr);
HRESULT hr = pMr->Deregister(&ov);
if (hr == ND_PENDING)
    hr = pMr->GetOverlappedResult(&ov, TRUE);
```

Rules:

- Invalidate any bound MWs first or `Deregister` returns `ND_DEVICE_BUSY`.
- After deregistration, the local and remote tokens are dead. Any
  in-flight request that uses them will fail.
- Failing to deregister before releasing the `IND2MemoryRegion` leaks
  pinned pages — they stay locked against your process until the
  adapter object is released.

## 7. Performance: registration is expensive, cache it

Three rules of thumb:

1. **Register once, use forever.** If you have a long-lived application
   buffer, register it once at startup and reuse the same MR for every
   transfer that touches it.
2. **Pre-register pools.** Many high-performance ND apps reserve a big
   buffer at startup, register it, and sub-allocate from there for
   per-message use. Sub-allocations only need to derive a `Buffer +
   Length + localToken` — no per-message Register call.
3. **Use inline for tiny sends.** Sends below `InlineRequestThreshold`
   can be issued with `ND_OP_FLAG_INLINE`, which bypasses the
   per-request token lookup entirely. The SGE's `MemoryRegionToken`
   can be invalid in that case.

The [ndmrlat](../../src/examples/ndmrlat) and
[ndmrrate](../../src/examples/ndmrrate) benchmarks exist specifically to
measure Register latency and throughput on your adapter. Run them once,
remember the numbers, and design accordingly.

## 8. Handling failures

| HRESULT | Meaning | Action |
|---------|---------|--------|
| `ND_INSUFFICIENT_RESOURCES` (Register) | Pinned-memory budget exhausted. | Deregister something or wait. |
| `ND_INVALID_PARAMETER` (Register) | Buffer larger than `MaxRegistrationSize`. | Split across multiple MRs. |
| `ND_ACCESS_VIOLATION` (Register) | Buffer pointer / length is invalid. | Programming error. |
| `ND_DEVICE_BUSY` (Deregister) | An MW is still bound to this MR. | Invalidate and retry. |
| `ND_ACCESS_VIOLATION` on a completion | The peer used the wrong access for the MR/MW. | Verify the flags both sides registered with. |

## 9. End-to-end: server publishes a 1 MiB buffer

Putting everything together — the same shape the `ndrping` server uses:

```cpp
// 1. Open the adapter, get the overlapped file handle.
//    (see Getting Started)

// 2. Allocate and register the buffer with remote-write access.
const size_t LEN = 1 << 20; // 1 MiB
char* buf = static_cast<char*>(HeapAlloc(GetProcessHeap(), 0, LEN));

IND2MemoryRegion* pMr = nullptr;
pAdapter->CreateMemoryRegion(IID_IND2MemoryRegion, hFile,
                             reinterpret_cast<void**>(&pMr));

OVERLAPPED ov{}; ov.hEvent = CreateEvent(nullptr, FALSE, FALSE, nullptr);
HRESULT hr = pMr->Register(buf, LEN,
                           ND_MR_FLAG_ALLOW_LOCAL_WRITE
                         | ND_MR_FLAG_ALLOW_REMOTE_WRITE,
                           &ov);
if (hr == ND_PENDING)
    pMr->GetOverlappedResult(&ov, TRUE);

// 3. (Optional) carve out a memory window scoped to the first 4 KiB.
IND2MemoryWindow* pMw = nullptr;
pAdapter->CreateMemoryWindow(IID_IND2MemoryWindow,
                             reinterpret_cast<void**>(&pMw));
pQp->Bind(nullptr, pMr, pMw, buf, 4096, ND_OP_FLAG_ALLOW_WRITE);
// Bind completion will show up on the CQ; you can grab the token now.

// 4. Tell the client.
struct PeerInfo {
    UINT32 remoteToken;
    UINT64 remoteAddress;
} info { pMw->GetRemoteToken(), reinterpret_cast<UINT64>(buf) };

ND2_SGE sge{ &info, sizeof(info), pMr->GetLocalToken() };
pQp->Send(nullptr, &sge, 1, ND_OP_FLAG_INLINE);

// 5. ...client does RDMA Writes into buf[0..4096)...

// 6. Revoke and shut down.
pQp->Invalidate(nullptr, pMw, 0);
pMw->Release();

pMr->Deregister(&ov);
pMr->GetOverlappedResult(&ov, TRUE);
pMr->Release();
HeapFree(GetProcessHeap(), 0, buf);
```

Next: [Advanced patterns](./07-advanced-patterns.md) — flow control,
shared receive queues, pipelining, and error handling for production
workloads.
