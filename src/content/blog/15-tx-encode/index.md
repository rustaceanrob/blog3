---
title: "Transaction Encodings"
date: "October 17 2025"
description: "Exploring reductions in transaction encoding size"
---

In my previous blog on implementing the `SwiftSync` protocol, I mentioned that bandwidth became a limiting factor of initial block download (IBD) speed. That spurred some thought to investigate how many bytes can be shaved off of the transaction encoding used in the bitcoin protocol, and I found some old work [here](https://github.com/TomBriar/bitcoin/blob/2023-05--tx-compression/doc/compressed_transactions.md) that explores compression techniques. This blog will walk through the transaction, and shave bytes off along the way. At the end I'll present some results using some of these techniques and some of my own. Let's begin!

# `CompactSize`

I will briefly mention a primitive used to condense data, `compactSize`. This scheme is an enumeration of sorts. Oftentimes we have a fixed precision number, say 64 bits, where we don't actually need the full 8 bytes to express the number. A `compactSize` encoding is an optimal encoding of 64 bits when numbers are frequently much smaller than the full range. For numbers less than 252 we use a single byte and interpret it as a number. If the first byte is `FD` in hexadecimal, we interpret the next 2 bytes as a 16 bit number. `FE` prefix corresponds to a 32 bit number, and finally `FF` represents a full 64 bit number. Notice uses the full precision, we require an _extra_ byte, but otherwise we save space.

# Bitmap

A bitmap is an array of binary digits that we can ascribe meaning to. This will be used in the encoding scheme as well. For example, say we have 2 bits and we want to convey user preferences. We can reserve the first bit as a boolean if they are subscribed to a mailing list, and the second bit may represent if they are a paid user. So `1 0` represents a user that subscribes to emails, but does not pay a subscription.

# Transaction Metadata

Every bitcoin transaction has a few metadata fields. The first one is a transaction `version`, which is 4 bytes in size. We are only on transaction version 3 over bitcoins long history, so let's instead express this as a 2-bit binary number. When we need transaction version 5, we can update the encoding scheme. Next is the `marker` and `flag` bytes. I am unsure of the lore here, but after the segregated witness update, these fields are fixed values. We may use a single bit to flag if `marker` and `flag` should be the typical values, which will be most of the time. Finally there is a `locktime`, which is 4 bytes wide. When the lock time is unset, these bytes are set to `0x00`, what a waste. Instead, we could use a single bit to indicate this transaction _is_ encumbered by a lock time, otherwise save the 4 bytes.

Here's where were at so far

| Field           | Old         | New              |
| --------------- | ----------- | ---------------- |
| Version         | 4 bytes     | 2 bits           |
| Marker and flag | 2 bytes     | 1 bit or 2 bytes |
| Locktime        | 4 bytes     | 1 bit or 4 bytes |
| Metdata         | Not present | 1 byte bitmap    |

# The Inputs

Inputs in bitcoin have fields `outPoint`, `nSequence`, `scriptSig`, `witness`. An `outPoint` references a previous transaction where the coins were created, and the index in which the coin was created. The index, `vout` field, is a fixed 4 bytes; however, we can do much better by using a `compactSize` here. Most of the time, transactions are 2-input and 2-output. Although this may, and hopefully will, change in the future, I still expect one byte to be sufficient in most cases to represent `vout`. 

The wacky field here is `nSequence`, which has changed semantic meaning over bitcoin's history. From a data encoding perspective, many times this field will be `0xFFFFFFFF`, `0xFFFFFFFD`, `0x00` or `0x01`, or something very similar. The best I could come up with here is to represent `0xFFFFFFAF` to `0xFFFFFFFF` as a single byte enum, `0x00` and `0x01` as enums as well, and encode all other values with 4 bytes. To read about the current use of the `nSequence` field, checkout [BIP-68](https://bips.dev/68/). 

Now for the updated table:

| Field           | Old               | New              |
| --------------- | ----------------- | ---------------- |
| Version         | 4 bytes           | 2 bits           |
| Marker and flag | 2 bytes           | 1 bit or 2 bytes |
| Locktime        | 4 bytes           | 1 bit or 4 bytes |
| Metdata         | Not present       | 1 byte bitmap    |
| `vout`          | 4 bytes per input | 1-5 bytes        |
| `nSequence`     | 4 bytes per input | 1-5 bytes        |

# The Outputs

Perhaps the most egregious waste of space is the `value` field of transaction `txOut`'s. This sucker is 8 bytes wide, representing the amount locked to the `scriptPubKey`. This may be replaced with a `compactSize`.

The final table:

| Field           | Old                | New              |
| --------------- | ------------------ | ---------------- |
| Version         | 4 bytes            | 2 bits           |
| Marker and flag | 2 bytes            | 1 bit or 2 bytes |
| Locktime        | 4 bytes            | 1 bit or 4 bytes |
| Metdata         | Not present        | 1 byte bitmap    |
| `vout`          | 4 bytes per input  | 1-5 bytes        |
| `nSequence`     | 4 bytes per input  | 1-5 bytes        |
| `value`         | 8 bytes per output | 1-9 bytes        |

# Loss-less vs lossy encoding

To my knowledge, these are all the ways in which to truncate transaction data with the previous encoding scheme fully recoverable. Backwards compatibility to the old scheme is crucial for computing transaction hashes, which in turn are used in merkle trees. More may be done that requires additional state on behalf of the client. For instance, we could replace the `txid` of an `outPoint` with the block height and the index in which the output was created. This would require the user to maintain a database to read block data on demand, as opposed to reading from the UTXO set. The write-up tagged in the beginning of the post also mentions legacy signature compression, but I am unaware of how this technique works.

# Result

Following this encoding, I ran some numbers on my `bitcoind` database. The total transaction data, excluding block headers, is 692GB on my local machine. With the encoding, this size becomes 650GB, a savings of around 6 percent (42GB). While not as ground-breaking as I'd hoped, 42GB is still significant for the average user. Some modern video games are roughly 42GB, and those can take 30 minutes to an hour depending on one's connection. You can run the numbers for yourself using [this repository](https://github.com/rustaceanrob/tx-encoding-stats).
