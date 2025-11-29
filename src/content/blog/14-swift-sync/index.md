---
title: "Implementing SwiftSync"
date: "October 14 2025"
description: "Speeding up initial block download"
---

My first `bitcoind` initial block download (IBD) took around a week. To be fair, I was using a very old external hard disk, but waiting that long to use an application isn't exactly a great UX. I hadn't given it much thought since, but since coming to [2140](https://2140.dev/), IBD has been top of mind. My colleague Ruben developed a [novel scheme](https://gist.github.com/RubenSomsen/a61a37d14182ccd78760e477c78133cd) for speeding up IBD when starting a new bitcoin node. This blog walks through his proposal, SwiftSync, and some of the unexpected findings I had along the way when implementing it.

# Intuition

From a high level, SwiftSync creates IBD speed-ups by enabling two advantages. One, blocks do not have to be downloaded sequentially to arrive at the UTXO set. This is a big one, as IBD goes from a single-threaded architecture to a multi-threaded, multi-peer task. As the resources of your machine scale, so should your IBD efficiency. Two, UTXOs that are spent later do not get written to disk. For many users, this will also create massive gains in efficiency, especially if they are not aware of the `dbcache` flag for `bitcoind`. I'll dive into how this works. 

Short aside, I will only be considering the "assume valid" version of SwiftSync, but the scheme may be extended to fully-validating IBD.

SwiftSync builds on the observation that the sum of all outputs ever created minus the sum of all inputs ever created is equivalent to the UTXO set.

`all outputs - all inputs = UTXO set`

Put another way, by subtracting all the UTXOs from the left-hand-size, we have:

`all outputs - UTXO set - all inputs = empty set`

This is a powerful fact, as it implies that the set of inputs and the set of outputs should be equivalent sets, as long as we _omit_ outputs that are in the UTXO set. From here we can use some simple math to develop a neat way of doing IBD.

Let's say somebody gives you a file that informs you of what outputs will be present in the UTXO set at a particular block hash, call this the stop hash. Using this file of hints, you may start keeping track of the set of all inputs and the set of all outputs by doing the following:

1. Initialize a 256-bit number to `0`, call this an aggregate, `agg`.
2. For each block, in no particular order:
	1. Take the hash of the `OutPoint` for all inputs.
	2. Represent these hashes as 256-bit numbers.
	3. Subtract these values from `agg`, modulo 256-bits.
	4. If a `TxOut` is **not** in the UTXO set, derive its `OutPoint` and hash it.
	5. Represent these hashes as 256-bit numbers.
	6. Add these values to `agg`, modulo 256-bits.
	7. If a `TxOut` is in the UTXO set, add it to the UTXO index.
	8. If all blocks fetched below the block hash in the file, stop
3. If `agg` is `0`, success. Else, fail (and default back to normal IBD from genesis).

What's going on here? In this scheme we are using properties of addition over a finite field of integers. Given an integer `n` and field order `m`, then for any starting value `a` where `a < m`:

```
(a + n - n) % m == a
```

This is obvious, but it is important to point this equation out. In our SwiftSync example, we set `a` equal to `0`, and instead of a single `n`, we add and subtract a set of values, `I = {a_0, ... a_n}`. The same statement is true:

```
(0 + I - I) % m == 0
```

With this single 256-bit aggregate, we can say the following: If someone gave us the correct hints for the UTXO set, the input `OutPoint` we hash and subtract from the aggregate will be the same as the output `OutPoint` we hash and add to the aggregate, in which case our aggregate will be `0` when arriving at the stop hash. At this point, you may want to review the algorithm once more.

# Observations

Taking for granted where this file of UTXO hints comes from, what are the properties of this protocol? For one, it does not matter if we first subtract an `OutPoint` hash before knowing about its corresponding `TxOut`. This is because of associativity over addition:

```
((a - n) + n) % m == ((a + n) - n) % m
```

This fact enables the parallelism described in the previous paragraph. When downloading blocks, as long as an input has a corresponding output at some point in the history, `agg` will be `0` in the end state. This does not account for coinbase maturity.

As for omitting unnecessary UTXO set writes, this is implied, as we know what should be in the UTXO set ahead of time with the file of hints.

Now examining the file of hints, is there a way for an adversary to cheat? My opinion is no, not practically. A client doing this style of IBD can pre-sync the chain of block headers to the stop hash before downloading the blocks, which commits to the block's transactions via the merkle root. In my rudimentary analysis, if an adversary wanted to lie and add a UTXO for themselves that does not exist in this chain of most work, the merkle root of the fake block would not match the one pre-committed in the header chain. If it did, this adversary controls an absurd amount of hashrate, and we have bigger problems to worry about.

Griefing is possible however. Imagine an adversary creates a hint file for an invalid UTXO set state. The client would not be aware this hint file is bogus until the end of IBD, at which point `agg` is not zero. The only information the client knows at this point is that their UTXO set is invalid for the given block history. They will have to restart the sync, or re-index rather, from genesis. I would argue this isn't a big deal practically speaking. If this hint file is signed and attested by a group of developers, these developers will put their reputation on the line. If they share something faulty, then they will no longer be trusted as a source of the file.

# Findings

I implemented this method of IBD [here](https://github.com/2140-dev/swiftsync). You may try it for yourself by running the [`binary`](https://github.com/2140-dev/swiftsync/tree/master/node). In completing this proof-of-concept, I ran into some unexpected results. 

When using my machine at home, the primary factor limiting my IBD speed using SwiftSync appeared to be my bandwidth. Even with 32 peers and many threads working, the throughput lead to somewhat underwhelming results at home. Contrasting this with a Hetzner server with similar specifications, I was pleasantly surprised. With the same IBD configuration, the one-gigabit connection for the Hetzner machine was able to chew through blocks extremely fast - I was able to do IBDs in around 1 hour 45 minutes on average. Take this in contrast with my laptop. At home, IBD took roughly 8 hours without SwiftSync. With SwiftSync, there was still an improvement, although it came in at roughly 4 hours in the best case.

This has definitely left an impression on me, as I disregarded bandwidth as a "solved bottleneck" in my mind prior to the project. On the contrary, any savings we can glean from wire-representation of data appears to be meaningful. I plan to follow-up with research on transaction representations in the near future.

# Practical Next Steps

I built this project on the Rust `bitcoinkernel` crate, which I am very hopeful will lead to interesting projects in the future, and even bolster existing projects like Floresta and `btcd`. That being said, we are a long-ways out from seeing user adoption of alternate clients from Bitcoin Core. So what chance does this method of IBD have of getting into `bitcoind`? From a multiple-peer parallel block download architecture, my outlook is somewhat grim. The logic for reading a block from the wire, validating it, updating the UTXO set, and writing the block to disk all occurs in a single function, `ProcessBlock`. This coupling to sequential block processing poses the main limitation in adding a fully parallelized SwiftSync implementation in Bitcoin Core, but there may still be significant improvements when using a more neutered version. If blocks are processed sequentially, IBD can still be sped up by removing unneeded UTXO writes using a hint file. This optimization alone should have great performance improvements, and does not require a large refactor. If Bitcoin Core starts with this simpler implementation, future versions can graduate to the fully parallel IBD SwiftSync has to offer. In fact, `theStack` already has a [patch](https://github.com/theStack/bitcoin/tree/ibd_booster_v0) you can review that implements SwiftSync over a single peer.



