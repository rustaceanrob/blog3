---
title: "Building APIs in Rust"
date: "September 21 2024"
description: ""

---

I recently released a `v0.1.0` for my project Kyoto. If you aren't familiar with Kyoto, it is a Bitcoin pseudo-node, which implements part of the peer-to-peer messaging layer and requests blocks for a wallet. Without underlying wallet software to make use of the Bitcoin blocks, Kyoto is mostly worthless; however, in combination with wallet software, Kyoto is a powerful tool to increase user privacy at a very low cost. The ultimate goal of the project is to allow users a viable alternative to running a full archival node, but, to achieve that goal,  developers have to incorporate Kyoto into their software. This symbiotic relationship between my project and other projects is why I needed to focus on great documentation and a great API for Kyoto.

So what makes a great library? I am going to ignore the obvious, which I consider to be detailed documentation, unit tests, integration tests, bench-marking when applicable, avoiding panics, and reasonable project design. From my experience, I will share some more subtle things that helped me get to my first release.

## Use your own code

This one seems strange or obvious. You may be thinking, _how can I avoid using my code when writing tests?_ You would be correct. But writing tests often checks for edge cases, simple scenarios, or faulty behavior. I encourage you to write multiple _examples_. Writing an end-to-end example forces you to use every aspect of your API as intended for the next developer. I encountered a number of questions for myself while doing this, that I may not have otherwise caught:
- Should the fields on this public struct also be public?
- How am I expected to use the structs I am given?
- Do the methods make sense? Their inputs? Their outputs?
- Is the process of programming with the library intuitive? 
- Are there any obvious API holes when building an application? 

I recommend writing examples as soon as you have public structures exposed. You will be forced to update them, sure, but I see that as a benefit. If you are committing to an API change, you will be _forced_ to consider how it effects your end-user. 
## Many types need wrapping

It is tempting to work directly with types you use from dependencies. For instance, when parsing a `headers` message read off of the wire, the result is a `Vec<Header>`. While this representation is exactly correct, in that a `Vec<Header>` is the message a peer will send us, it is insufficient in capturing all the constraints that are required when receiving block headers from peers. Notably, this `Vec<Header>` must:
- Adhere to the network difficulty adjustment
- Each header must satisfy its own target work requirement
- Each header must have a timestamp greater than the median time of the last 11 blocks

These checks can bloat a function quickly, and would make the function hard to reason about, or make changes to. Instead, a new structure should be introduced that can validate a list of headers based on these requirements. Often, this is rightfully called a "new-type" pattern, where additional requirements are imposed on a primitive or collection. The result is a program that is far easier to reason about. First, we can parse the `Vec<Header>` into the new type, which I called `HeadersBatch`. The `HeadersBatch` has methods exposed that do each one of these validation checks. The function responsible for syncing the block headers may simply call these methods, making for code that is more readable and modular. In a library of significant code size, this was a crucial pattern to adopt, or else the library would be unreasonably hard to reason about and maintain.
## Mind the `clone

If you listen to any talk about Rust, the speaker will often say something along the lines of "Why Rust?" and follow that question by saying Rust is fast and memory conservative. I don't agree with that completely. Rust _allows_ developers to write fast and memory conservative code, but Rust also allows for memory hungry programs too. It is easy to add `clone` to a struct as you are prototyping, but I try to avoid this even in early stages. If the program requires a `clone` to compile, oftentimes the function logic should be reconsidered. Using `clone` is a signal to me that my program may not be correct, not an edge case. 

It may not seem like a big deal, but it is why I use Rust in the first place. When choosing a language like Swift or Go, _everything_ is reference counted. To beat the performance of garbage collected languages, it is important to realize a reference counter will outperform `clone`, and calling `clone` is not something to take lightly. Especially for large projects, the learning curve for Rust can be incredibly steep. Swift, Kotlin, or Go may be a better option to bring on more developers, so choosing Rust is no small consideration. To respect your own time and other contributors, make the most out of Rust, even if the compiler allows you to make memory-hungry choices. 

Of course, that doesn't mean `clone` has to be avoided completely, but I suggest setting a boundary. Something like the following: I will only `clone` a struct if it is less than 50 bytes of memory.
## Get creative

A central theme of this project was memory-conservation. While I have thought about memory consequences before in previous code and libraries, the implications were far more understandable in those instances. I was dealing on the scale of fixed sized buffers, and small vectors. This project brought far more complexity, including a `tokio` runtime, parsing large messages and sharing them across threads, and managing large collections in the form of `BTreeMap` and `Vec`. To make sense of this overhead, I had to explore the tools available to me. 

I turned to my favorite learning platform, YouTube, to find some inspiration. Sure enough, I found an interesting profiling tool, `tokio-console`, that allowed me to profile the memory usage of my application. It was an essential tool in early development stages, when the structure of my project was not yet defined. And again, on YouTube I found a video from Apple on profiling programs in Xcode. When the project had matured more, I leveraged Xcode to see how a mobile application would perform with a node running in the background.

## Takeaways

- Write examples early.
- Create abstraction, even if simple.
- Consider other languages. Not everything has to be in Rust.
- When using Rust, use it well. Only `clone` small objects.
- Find new ways to make your project fun and exciting. 