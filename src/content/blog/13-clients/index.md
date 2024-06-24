---
title: "Future Bitcoin Clients"
date: "May 29 2024"
description: "Privacy anyone?"

---

I remember when I was first grappling with _Mastering Bitcoin_, I had to read the sections on light clients more than a few times. I only went back to read the concepts again because I found the concept of having a pseudo Bitcoin node on an iPhone to be cool, and in scrounging around the internet for more information on how this implementation would look, I found a [video](https://youtu.be/7FWKc8lM4Ek?si=svv5gcl0yBWGA1Pn) by Olaoluwa speaking about BIP-157/BIP-158 compact filters -- circa 2017. He definitely sold me on the vision by describing Bitcoin developers walking around and paying for things using compact block filter based clients. So what happened? 

Well, first, what is a compact filter? A compact block filter is just a compressed Bitcoin block that low-resource devices can easily check for inclusions of transactions they care about without downloading the block from a Bitcoin full-node up front. The catch is there may be some false positives when a client tests against this filter for inclusions, but there are also some great advantages when checking against these compact filters for things we care about. Namely, we can check a filter for any inclusions in a set of addresses in _linear time_, so we can scan the blockchain efficiently as we download these filters from peers. What is the point? From the point of view of the full-node operator, all they can see is the client requested an entire block to download. The anonymity set of a block is quite large, so the user has a reasonably high degree of confidence no single entity knows what scripts they own. Okay, that sounds great, but is anyone using compact filters in production? Not surprisingly, Olaoluwa and Lightning Labs built the [Neutrino](https://github.com/lightninglabs/neutrino) client in Golang that uses compact filters instead of full blocks to index the blockchain. You can choose Neutrino as a backend for [LND](https://github.com/lightningnetwork/lnd), and it works just fine. The only caveat is that Golang isn't normally suited for mobile environments, at least not compared to the ability to build native Swift and Kotlin code using Rust's FFI.

So why don't popular mobile wallets offer a compact filters backend? Well, the short and sad answer may be that people don't really give a shit where their balances come from. For better or for worse, most people just care that the number on their screen looks correct when they check their balance. That being said, the entrepreneurial side of me is speculating that users will care about where their block data is coming from in the future, which is why I am working on a compact filters Bitcoin client in Rust called [Kyoto](https://github.com/rustaceanrob/kyoto). The theme here is Bitcoin is designed as the currency of enemies. If Bitcoin continues to become a battle-hardened settlement layer of commerce amongst mutually untrusted parties, the stakes will become much higher over time.

#### Block Fees, Custodianship, and the Future

The silver lining of the Ordinals craze, which I know nothing about besides a surface level association with the acronym **NFT**, is that we have a great look at what settlement will be like when block demand is high. Further still, if Bitcoin sees increased adoption as we head into the future, the unit economics don't scale such that each user will own their own UTXO, or Lightning Network channel for that matter. While the future remains far away, it appears that Chaumian mints, whether they be federated or centralized, will be a potential solution for scaling Bitcoin. 

The primary complaint about Bitcoin-backed mints, is the mint can introduce inflation through fractional reserve custodianship. Sound familiar? From that description, it sounds like Bitcoin would suffer the same fate as gold in the turn of the previous century into the 1970s. Unlike gold, however, Bitcoin reserves are auditable. So auditable, in fact, that a light client running on an iPhone could query for the reserves of the local credit union.

How might this work? Over a secure communication channel, the mint may update members of any scripts that the mint is using to reserve their balance sheet. Users, who do not want to reveal their membership of a mint to an untrusted entity, may query Bitcoin full-nodes for compact filers to get the revealed balance of the mint without revealing the transactions they are interested in. Associated with the scripts sent over the secure commication channel, the mint may also send signatures proving they possess the key(s) for the scripts. With this information, community members may audit their mint's balance sheet in a private and untrusted way, all from their couch on their phone. Of course, this does not fully solve the problem. Mints can still mint new ecash tokens with only a fraction of their reserves, but ratios of yesteryear, say 5% reserves, would quickly be discovered under this scheme. 

#### Improvements, Attacks

Improvements in attacks on Lightning channels have made leaps and bounds, but there is an interesting problem that has nothing to do with the Lightning Network that crops up in production. Most, if not all, adversarial actions taken against a channel are met with a swift broadcast of a transaction to the Bitcoin network. But, what if the software isn't exactly _connected_ to the Bitcoin network when trying to broadcast this dispute settlement mechanism? Unless a user is connected to a home node, a channel with keys held on a mobile device is most likely getting their data from a single entity with an identifable handful of IP addresses. Single sources of truth are subject to denial of service attacks, and a well-captialized malicious actor could open channels with users of a certain wallet or data provider, only to DOS the server they use, preventing the user from broadcasting revocation transactions, ultimately stealing their money.

I think a realistic viewpoint is this will be a problem for the _Uncle Jims_ of the future. But as of right now, many individual users have Lightning channels open, and I think channel numbers will continue to grow, especially for routes between large economic nodes. In an ideal world, underpinning every Lightning node with a channel of substantial value is a software that can broadcast revocations directly to the peer-to-peer network. A compact filters client naturally implements this functionality, as broadcasting to the peer-to-peer network after rolling BIP-157/BIP-158 is a low lift.

#### Precious, Precious Data

The majority of current mobile wallets source their data from either Blockstream's Esplora, Electrum, or a proprietary server. The goal of this section is not to bash what a great impact these projects have had on the user experience for Bitcoin holders. In fact, a potentially hot take of mine is the majority of the userbase will continue to use these data sources with no issues. The problem is, you never truly know what these entities will do with your requests for information about a set of Bitcoin addresses. While that is not an inherit problem, it becomes a problem if this data is sold to third parties (i.e. a company). Why? Well the best example is variable pricing for customers _based on_ the amount of money they have. Imagine an airline company buys information from a Bitcoin data provider, and charges you more money for an economy seat than the other passengers in economy because they know you have the chops for it. That wouldn't be very nice, and that is certainly not the future I would like to live in. If Bitcoin becomes the currency to settling debts between nation-states instead of settling coffee payments, I expect users will exercise much more care when disclosing _who_ knows how big their pile of corn is. 