---
layout: post
title: "What does Apple's TSO mode implement?"
summary: "Apple's chips can run any thread under a TSO model. What model does it actually enforce given that the thread actually runs Arm instructions? How does a TSO thread interact with an Arm thread? And something about Arm's atomics along the way."
---

Apple's M-series chips have a TSO mode, which Rosetta turns on for the threads that run translated x86 code. Asahi Linux exposes the mode to user programs, and the x86 emulator [FEX-Emu](https://fex-emu.com/FEX-2306/) uses it. A proposal to add this interface to mainline Linux was [declined](https://lkml.iu.edu/2404.2/05657.html), in part because [no specification says](https://lists.infradead.org/pipermail/linux-arm-kernel/2024-May/926618.html) which memory model the mode implements.

[Apple's documentation](https://developer.apple.com/documentation/virtualization/accelerating-the-performance-of-rosetta) says only that the mode gives "the x86-64 TSO memory model". This leaves two questions open.

First, what memory model does a TSO-mode thread *actually* enforce? A thread in TSO mode has Arm counterparts of x86's operations: loads, stores, locked read-modify-writes, and a full fence. But it is still an Arm thread, and it can also run Arm instructions that x86 does not have, such as acquire loads and release stores. How are these ordered with the thread's other accesses? I call this question *sequential composition*.

Second, what is the memory model of the machine when threads of both kinds share memory? I call this *parallel composition*; it is the question of my [SIGARCH post on compound memory models](https://www.sigarch.org/compound-memory-models/).

## In short

I tested both questions on an M1 and an M2 running Linux, with 936 litmus tests ([report](https://arxiv.org/abs/2611.21406), [artifact](https://github.com/vijay-nagarajan/apple-tso-litmus)).

- **Sequential composition.** TSO mode appears to be x86-TSO, and for every instruction on a TSO thread the chips seem to follow one simple rule: keep TSO's orders and add Arm's.
- **Parallel composition.** Programs that mix Arm and TSO threads follow the compound model, in which each thread continues to enforce the orders of its own mode.
- **Read-modify-writes (RMWs).** Along the way, Arm's RMWs turn out to order more than their rules suggest, and TSO mode treats Arm's two kinds of RMWs differently.

## Sequential composition

Consider store buffering with both threads in TSO mode (Figure 1). T0 has a full barrier between its store and its load, so the question is only about T1: a store followed by an acquire load (`LDAR`).

<div class="litmus-pair">
<div>
<table><thead><tr><th>T0 <span class="tso">(TSO)</span></th><th>T1 <span class="tso">(TSO)</span></th></tr></thead><tbody>
<tr><td><code>St x = 1</code></td><td></td></tr>
<tr><td><code>DMB</code></td><td></td></tr>
<tr><td><code>r1 = ld y</code> (0)</td><td></td></tr>
<tr><td></td><td><code>St y = 1</code></td></tr>
<tr><td></td><td><code>r2 = acq ld x</code> (?)</td></tr>
</tbody></table>
<p class="panel-label">(a) With a plain store</p></div>
<div>
<table><thead><tr><th>T0 <span class="tso">(TSO)</span></th><th>T1 <span class="tso">(TSO)</span></th></tr></thead><tbody>
<tr><td><code>St x = 1</code></td><td></td></tr>
<tr><td><code>DMB</code></td><td></td></tr>
<tr><td><code>r1 = ld y</code> (0)</td><td></td></tr>
<tr><td></td><td><code>rel St y = 1</code></td></tr>
<tr><td></td><td><code>r2 = acq ld x</code> (?)</td></tr>
</tbody></table>
<p class="panel-label">(b) With a release store</p></div>
</div>

*Figure 1. Store buffering on two TSO threads: T0 has a full barrier (`DMB`), and T1 a store followed by an acquire load (`LDAR`). Initially x = y = 0; a value in parentheses is the value a load reads. Must r2 read 1? (a) With a plain store, no. (b) With a release store (`STLR`), yes.*

On a TSO thread every store is already ordered after the thread's earlier accesses, as an Arm release is. On Arm, a release store is also ordered before a later acquire load (`LDAR`), whereas in TSO a store is not ordered before a later load. So is a plain store on a TSO thread ordered before an acquire load? It is not: the weak outcome appears 10<sup>5</sup> to 10<sup>6</sup> times in 6&times;10<sup>8</sup> runs. A release store, on the other hand, remains ordered before an acquire load.

This generalizes. On a TSO thread, every Arm instruction keeps x86-TSO's orders and adds Arm's: two accesses stay in order if either model keeps them in order. Both chips follow this sequential composition in all 272 tests whose threads all run in TSO mode. The chips are stronger than the models in a few cases, in both modes; for example, Arm threads also keep a load ordered before a later store. The [report](https://arxiv.org/abs/2611.21406) has the details.

## Parallel composition

Now let's mix the two kinds of threads. Figure 2 is message passing where the producer is an Arm thread with a release store, and the consumer is a TSO thread with plain loads. Does it work as expected?

| T0 <span class="arm">(Arm)</span> | T1 <span class="tso">(TSO)</span> |
|---|---|
| `St data = 1` | |
| `rel St flag = 1` | |
| | `r1 = ld flag` (1) |
| | `r2 = ld data` (?) |

*Figure 2. Message passing from an Arm thread to a TSO thread. Initially data = flag = 0. Does r2 read 1?*

The compound model says yes: the Arm release keeps the producer's stores in order, and TSO keeps the consumer's loads in order. Neither chip showed r2 = 0 in 6&times;10<sup>8</sup> runs. More generally, I ran 529 tests that mix Arm and TSO threads, including three- and four-thread tests such as WRC, RWC and IRIW under every assignment of modes. The compound model forbids the outcome of 348 of them, and neither chip showed any of the 348.

## Read-modify-writes

Arm has two kinds of atomic read-modify-write (RMW) instructions: atomic memory operations (AMOs), such as `SWP` and `LDADD`, and load-linked/store-conditional (LL/SC) pairs, `LDXR` and `STXR`. x86 has AMOs, and a conditional one, `CMPXCHG` (Intel specifies that a `CMPXCHG` whose comparison fails still writes back the value it read, so it too behaves as an AMO). A TSO thread on an Apple chip has all of these variants, which makes it an interesting place to study them. But first, something about Arm.

Take store buffering again and replace each thread's load with a release swap (`SWPL`) on the same location (Figure 3). Can both swaps read 0?

| T0 <span class="arm">(Arm)</span> | T1 <span class="arm">(Arm)</span> |
|---|---|
| `St x = 1` | |
| `r1 = rel swp y, 2` (0) | |
| | `St y = 1` |
| | `r2 = rel swp x, 2` (?) |

*Figure 3. Store buffering on Arm with release swaps in place of the loads. Initially x = y = 0. Can r2 also read 0?*

It may look like they can. On T0, the release keeps the store to x before the swap's write, but not before its read. So the swap's read could seem to happen early, like a plain load in store buffering. But atomicity means that no other store to y can come between the swap's read and its write. To other stores to y, therefore, the read and the write look like a single access. So ordering the store to x before the swap's write in effect orders it before the read too. The same holds on T1, and the two swaps cannot both read 0 (as in store buffering with a fence on each side). Arm's official model agrees (checked with the herd tool), and neither chip ever showed r1 = 0 and r2 = 0 in the test of Figure 3. The same holds for an LL/SC pair with a release store-conditional, if the store-conditional succeeds (if it fails, it writes nothing, and the outcome is allowed).

We first pointed out orders of this kind, for TSO, in [PLDI'13](https://doi.org/10.1145/2491956.2462196). They became a point of discussion when the RISC-V memory model was being defined. Dan Lustig, who led the working group, pointed me to Will Deacon's 2014 [arm64 mappings of the Linux atomics](https://github.com/torvalds/linux/commit/8e86f0b409a44193f1587e87b69c5dcf8f65be67), which rely on the same idea to save one of two barriers.

RISC-V went a bit further. Replace each thread's store with an acquire swap instead, and keep the loads (Figure 4). Can both loads read 0?

| T0 | T1 |
|---|---|
| `r1 = acq swp x, 1` (0) | |
| `r2 = ld y` (0) | |
| | `r3 = acq swp y, 1` (0) |
| | `r4 = ld x` (?) |

*Figure 4. Store buffering with acquire swaps in place of the stores. Initially x = y = 0. Can r4 also read 0? On RISC-V (`AMOSWAP.aq`), no. On Arm (`SWPA`), yes.*

In RISC-V's model an AMO is treated as a single, indivisible memory operation. Its acquire thus orders the whole AMO, write included, before the later load. The two loads therefore cannot both read 0. In Arm's model the acquire orders only the AMO's read. Arm's atomicity does not help here. It keeps other stores out from between the read and the write, but not other threads' reads. So Arm allows r4 = 0, and the chips show it, albeit rarely (it never appeared in 6&times;10<sup>8</sup> runs, but did in runs about 40 times longer).

That brings us back to TSO mode. On a TSO thread, every AMO acts as a full fence on both chips, as x86's RMWs do, whatever its ordering annotation (even a `CAS` whose comparison fails). An LL/SC pair, however, is not a fence, although atomicity still orders it after an earlier store (as in Figure 3), even without a release, if its store-conditional succeeds.

The [report](https://arxiv.org/abs/2611.21406) describes all of these with evidence, and the [artifact](https://github.com/vijay-nagarajan/apple-tso-litmus) includes the herd models and all 936 litmus tests with their results. 
