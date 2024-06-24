---
title: "Kick React Native"
description: "I went with Swift, and I can't go back"
date: "Jan 18 2024"
---


## React Native Pitfalls

I love React Native for bootstrapping an app. It is great to prototype, and, most importantly, get users quickly. 
**But**, it quickly becomes a pain in the ass as your app gets more complex. Forget about Expo managed workflows once you need native code. Shipping NFC, cryptographic signatures, or anything else obscure? Throw React Native in the garbage. After hours of pain and suffering, when I started my latest app that uses both of these features, I knew Expo was going to ruin my life.

## The Swift Experience

I finally picked up Swift. After experiencing both development environments, I am confident Swift app will come out more polished and familiar than 95% of React Native apps, at least built in the same amount of time. There is something about SwiftUI that is just too clean to try replicate. I don't know if it's the font, the spacing, or the colors. When you pick up Swift, you get an app that looks like Apple made it, and third party code integrates without any issues as a nice side effect. FaceID, NFC, Keychain, etc. is all accessed with minimal overhead, and your app keeps pace with the standard look users expect. 

> Try Swift and SwiftUI if you already know Rust, Typescript, or Java. 
> You are cutting out potential users, but your app will be much cleaner and easier to manage.
> Unless your userbase already exists, a blank canvas on Swift will be a far more enjoyable experience.
> Swift is a robust language with support for asynchronus tasks, threading, and shared state buit in by default.