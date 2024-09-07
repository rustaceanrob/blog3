---
title: "Kyoto: a Bitcoin node in your pocket."
description: "More privacy for more people."
date: "Apr 2 2024"
repoURL: "https://github.com/rustaceanrob/kyoto"
---

Running a full archival Bitcoin node is a resource intensive endeavour that only a few passionate individuals choose to take on. Many users and developers rely on third party chain-oracles 
to inform them about the transactions related to their wallet. There are a number of concerns with this, not only from a privacy point-of-view, as the service knows the user's entire 
transaction history, but servers may also go down, get hacked, or otherwise become unresponsive. Especially in the context of the Lightning Network, this can result in a loss of user funds.
Kyoto is a light-client that attempts to bridge the gap between a Bitcoin full-node and a trusted third-party. While the premise sounds simple, there is a lot more happening behind the curtain
of Kyoto than one might think.

Kyoto is built upon the [BIP157](https://github.com/bitcoin/bips/blob/master/bip-0157.mediawiki) and [BIP158](https://github.com/bitcoin/bips/blob/master/bip-0158.mediawiki) specifications, which
describe how Bitcoin full-nodes can encode data about blocks in a clever way that preserves privacy for light-clients like Kyoto when requesting blocks. Moreover, Kyoto speaks the Bitcoin P2P protocol, meaning a Kyoto node handles, sends, and responds to messages sent by other nodes. Similar to a web server, Kyoto spawns and manages multiple asynchronous threads to handle multiple peers, and processes a number of different Bitcoin P2P messages to recreate the blockchain history, resolve disputes, and scan for transactions. Kyoto also leverages encrypted messaging with an integration of a library maintained by yours truly (and my buddy Nick).