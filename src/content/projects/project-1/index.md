---
title: "Encrypted Peer-to-Peer Messaging."
description: "A library to support secret message exchange between Bitcoin nodes."
date: "Apr 2 2024"
repoURL: "https://github.com/rust-bitcoin/bip324"
---

## BIP324 Encrypted Transport Protocol

[BIP324](https://github.com/bitcoin/bips/blob/master/bip-0324.mediawiki) describes an encrypted communication protocol over the Bitcoin P2P network. Encrypted messages offer a number of benefits over plaintext communication, even though the data exchanged over the Bitcoin P2P network is public to some degree. For instance, plaintext message tampering without detection is trivial for a man in the middle (MitM) attacker. Additionally, a nefarious actor may associate metadata such as IP addresses and transaction origins without explicitly having to connect directly to peers. BIP 324 - "V2" - transport forces nefarious observers to actively connect to peers as opposed to passively observing network traffic, and makes packet tampering detectable. Furthermore, V2 messages over TCP/IP look no different from random noise, making Bitcoin P2P packets indistinguishable from other network packets. 

To minimize dependencies and prepare a codebase that fits the [Rust Bitcoin Community](https://github.com/rust-bitcoin) code standards, all encryption is built from scratch, including the [__ChaCha20__](https://cyberw1ng.medium.com/understanding-chacha20-encryption-a-secure-and-fast-algorithm-for-data-protection-2023-a80c208c1401) and [__ChaCha20Poly1305__](https://en.wikipedia.org/wiki/ChaCha20-Poly1305) encryption algorithms. While __ChaCha20__ is actually quite understandable in its algorithmic implementation, the __Poly1305__ algorithm proved to be a challenging rabbit hole. This repository touches many programming topics, from TCP, multithreading, cryptography, and writing library code capable of a variety build targets.
