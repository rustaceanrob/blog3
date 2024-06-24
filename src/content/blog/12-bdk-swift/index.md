---
title: "Getting Started with Swift and BDK"
date: "April 15 2024"
description: "The transition guide for React Native developers."

---


When deciding how to build a mobile application, a consequential consideration is whether to pick a cross-platform or native solution. Small development groups frequently reach for frameworks like Flutter or React Native, as both platforms allow for quick prototyping, releases and updates. In this article I am going to make the case for building a native mobile application in Swift, what benefits this development approach offers both your team of developers and your end-users, and how to transition to Swift with experience in React Native. 

#### How Rust is Changing Development

Before talking about Swift, it is important to note how Rust is changing the workflow for development teams in a great way. Since the Mozilla team has launched [UniFFI](https://mozilla.github.io/uniffi-rs/), the viability of building Swift and Kotlin apps in small teams has increased significantly. BDK is a great example of how UniFFI can concentrate the core business-related logic into a single Rust codebase, and allow app developers to focus on the experience they are delivering their users. Rust and UniFFI allow for a greater specialization of small teams, with core engineers focused on Rust, and mobile engineers specializing in the user experience on iOS or Android platforms. To read more on UniFFI and the approach BDK takes when building UniFFI libraries, check out [this article](https://bitcoindevkit.org/blog/bindings-scope/) on the BDK blog.

#### The Advantages of Building Natively

Brace yourself for some opinionated anecdotes. I have experience building mobile applications in both React Native and Swift, and I prefer my developer experience building an iOS app directly with Swift. Different teams will have different opinions, but this is what I have found when comparing the experience between the two platforms.

To me, native applications look better. By using [SwiftUI](https://developer.apple.com/Xcode/swiftui/), padding, fonts, buttons, and other user interface objects are directly accessible to developers, often usable in applications with _no modifications_. There are hundreds of prebuilt user interface components that are polished and responsive with no additional imports. Animations that users are accustomed to in other apps, but may not notice directly, are incorporated as first-class support when building on Swift and SwiftUI. 

As a more general philosophy when building applications that handle money, the user should and deserves to feel comfortable while using the application. Swift and SwiftUI helps small development teams achieve a friendly, professional, and effective user interface without having to form the building blocks from scratch. When targeting specific platforms, users get the experience of a product that was built by a large engineering team, yet it might be built by only a few bitcoin developers with a keen eye for design.

Authentication, whether biometrics or passcodes, secret storage, cryptographic primitives, and many more APIs are directly available to native developers. In the case of Swift, the team at Apple maintains these APIs directly, and integration into Swift apps can be as little as a few lines of code. 

Although I find it quite annoying that Apple gate-keeps the ability to build iOS apps behind Apple computers, their tooling for building apps is powerful. Xcode displays live previews of your app screens, so you can tune the finest details and iterate quickly. There are simulators for each iOS device, and testing the logic of your application is robust. 

Enough talking about Swift, the rest of the article will be focused on how to get up and running with BDK for those that have no prior iOS programming knowledge, but potentially a background in React Native. We will have a look at a few snippets of code from Matthew Ramsden's [Example Swift Wallet](https://github.com/bitcoindevkit/BDKSwiftExampleWallet) in the BDK collection of repositories. 

#### Requirements

To follow along using Matthew's Github project, you will need an Apple computer running macOS Sonoma 14.0 or later and [Xcode](https://developer.apple.com/Xcode/) 15.0 or later. To install Xcode if you do not have it already, you may find Xcode on the App Store. Next clone the repository: 

- `git clone https://github.com/bitcoindevkit/BDKSwiftExampleWallet.git`

Now open Xcode in the folder you just cloned. 

#### SwiftUI Components

Let's have a look at a screen that comes up frequently in bitcoin applications: the screen to receive bitcoin, found at `BDKSwiftExampleWallet/View/ReceiveView.swift`

```swift
import BitcoinUI
import SwiftUI

struct ReceiveView: View {
    @Bindable var viewModel: ReceiveViewModel
    @State private var isCopied = false
    @State private var showCheckmark = false

	var body: some View {
		...
	}
```

We start by importing the necessary libraries at the top of our file, and then we define a `struct` that extends `View`. A `View` only has one requirement, that it contains a `body`. For those coming from React Native, this concept should feel quite familiar. Indeed, we often use the `<View></View>` JSX tag in a React Native application to define sections of the screen. The outer-most SwiftUI `View` is where you define what the user sees in your application. 

How about those property wrappers, `@Bindable` and `@State`? Staring with `@State`, we are telling SwiftUI to keep track of the variable we define, and update the user interface when changes to that variable occur. More familiarity to React Native! This definition maps directly to the `useState` hook we use so frequently in RN. And `@Bindable`? `@Bindable` is similar to `@State`, except the `ReceiveViewModel` publishes changes that `@Bindable`listens for, and the `viewModel` defined as a variable _may also_ call methods in the `ReceiveViewModel` to carry out business logic. This principle of separating the application logic from the application screens is known as the [Model-View-ViewModel](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel) architecture. We will explore a `ViewModel` in the next section. 

What makes SwiftUI user interface components is the ability to combine the _appearance_ and _behavior_ of a component with modifiers. To see how to modify the appearance of a component in our view, let's observe how we display the address to the user. 

```swift
VStack(spacing: 8) {
	Image(systemName: "bitcoinsign.circle.fill")
		.resizable()
		.foregroundColor(.bitcoinOrange)
		.fontWeight(.bold)
		.frame(width: 50, height: 50, alignment: .center)

	Text("Receive Address")
		.fontWeight(.semibold)
}
.font(.caption)
.padding(.top, 40.0)
```

While it might look intimidating at first, the modifier pattern of SwiftUI is my favorite feature. There are no style sheets or CSS tags, we simply add the appearance we would like to modify by chaining modifiers on a `View` component. 

Similarly, we can define behavior on SwiftUI components too. The button to copy the address of the user demonstrates this well:

```swift
Button {
	UIPasteboard.general.string = viewModel.address
	isCopied = true
	showCheckmark = true
	DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
		isCopied = false
		showCheckmark = false
	}
} label: {
	HStack {
		withAnimation {
			Image(systemName: showCheckmark ? "checkmark" : "doc.on.doc")
		}
	}
	.fontWeight(.semibold)
	.foregroundColor(.bitcoinOrange)
}
```

This `Button` takes a function closure that can define arbitrary behavior when the user interacts with the button. Here, we copy the address to the clipboard and animate a checkmark on the screen. The power of SwiftUI is on display here, as we animated an icon, copied information to the user clipboard, and styled a button with less than 15 lines of code. All with _zero_ external dependencies. 

#### Defining Behavior with ViewModels

We saw earlier that we can have SwiftUI listen for changes on our `viewModel` variable using the `@Bindable`, and we can use this `viewModel` to execute logic for our `View`. Now we turn to the `ViewModel` defined in `BDKSwiftExampleWallet/View Model/ReceiveViewModel.swift`

```swift
import BitcoinDevKit
import Foundation
import Observation

@Observable
class ReceiveViewModel {
    let bdkClient: BDKClient
	var address: String = ""
	var receiveViewError: BdkError?
    var showingReceiveViewErrorAlert = false
    
    init(bdkClient: BDKClient = .live) {
		self.bdkClient = bdkClient
	}
	...
}
```

We import our necessary libraries as usual, and wrap the `ReceiveViewModel` class in the `@Observable` property wrapper. Now, in our `View`, we may call the associated methods in `ReceiveViewModel` and listen for changes on those variables. The only method in the `ReceiveViewModel` is a function to `getAddress` to receive bitcoin.

```swift
func getAddress() {
	do {
		let address = try bdkClient.getAddress()
		self.address = address
	} catch let error as WalletError {
		self.receiveViewError = .Generic(message: error.localizedDescription)
		self.showingReceiveViewErrorAlert = true
	} catch let error as BdkError {
		self.receiveViewError = .Generic(message: error.description)
		self.showingReceiveViewErrorAlert = true
	} catch {
		self.receiveViewError = .Generic(message: "Error Getting Address")
		self.showingReceiveViewErrorAlert = true
	}
}
```

We will talk about the `bdkClient` variable in the next section, but for now we know that it encapsulates our wallet logic in a single structure. After taking a look at this error handling, it is easy to see why we break out the core functionality into a new file. The conept of a `ViewModel`is similar to hooks in React Native, however I find the `ViewModel` approach far simpler. We define a structure with some behavior for our users, and allow our `View` to invoke functions and listen for changes on the variables in the `ViewModel`.

#### The Wallet

Finally, the part you've been waiting for. How do we use BDK? The client for interacting with BDK in the app is found at `BDKSwiftExampleWallet/Service/BDK Service/BDKService.swift`

```swift
import BitcoinDevKit
import Foundation

private class BDKService {
    static var shared: BDKService = BDKService()
    private var balance: Balance?
    private var blockchainConfig: BlockchainConfig?
    var network: Network
    private var wallet: Wallet?
    private let keyService: KeyClient
	...
```

The `BDKService` contains all of our wallet related functionality in a structure known as a singleton. The singleton pattern is used for defining a class that should only be used once throughout the application, which explains the `static` variable that defines a `BDKService` within the class itself. The semantics of singletons are not too important for our application, but we see that some important global variables are defined in the `BDKService`, such as the `balance` of the wallet or our chain source, `blockchainConfig`. Initializing the `blockchainConfig` is simple. 

```swift
init(
	keyService: KeyClient = .live
) {
	let storedNetworkString = try! keyService.getNetwork() ?? Network.testnet.description
	let storedEsploraURL =
		try! keyService.getEsploraURL()
		?? Constants.Config.EsploraServerURLNetwork.Testnet.mempoolspace
	self.network = Network(stringValue: storedNetworkString) ?? .testnet
	self.keyService = keyService
	let esploraConfig = EsploraConfig(
		baseUrl: storedEsploraURL,
		proxy: nil,
		concurrency: nil,
		stopGap: UInt64(20),
		timeout: nil
	)
	let blockchainConfig = BlockchainConfig.esplora(config: esploraConfig)
	self.blockchainConfig = blockchainConfig
}
```

Ignoring the `keyService` for a second, we see how to define the chain source to an Esplora client. Next we can load the wallet stored in the iOS secure storage and set the `wallet` within the `BDKService`. 

```swift
private func loadWallet(descriptor: Descriptor, changeDescriptor: Descriptor) throws {
	let wallet = try Wallet(
		descriptor: descriptor,
		changeDescriptor: changeDescriptor,
		network: network,
		databaseConfig: .memory
	)
	self.wallet = wallet
}
```

Now we should update our local wallet database with the Esplora client. We do this to ensure we do not reuse a user's address.

```swift
func sync() async throws {
	guard let config = self.blockchainConfig else {
		throw WalletError.blockchainConfigNotFound
	}
	guard let wallet = self.wallet else {
		throw WalletError.walletNotFound
	}
	let blockchain = try Blockchain(config: config)
	try wallet.sync(blockchain: blockchain, progress: nil)
}
```

The majority of this function is ensuring our configuration and wallet have been properly defined. All it takes to sync our local database to the chain is a fallible call to a wallet method `sync`.

When the user requests their new address to receive bitcoin, we simply call a method on `self.wallet`.

```swift
func getAddress() throws -> String {
	guard let wallet = self.wallet else {
		throw WalletError.walletNotFound
	}
	let addressInfo = try wallet.getAddress(addressIndex: .new)
	return addressInfo.address.asString()
}
```

That is all there is to it. Make sure to explore the rest of the file for more BDK related logic! At this point you may be wondering, we defined the `BDKService`, but where is the `BDKClient` used in the `ViewModel`? We define a `BDKClient` that exposed methods on the `static var shared: BDKService = BDKService()` defined in our `BDKService`. This pattern allows for mock testing and single instantiations of a `BDKService`. 

#### Closing Remark

BDK demonstrates how Rust can be used to separate the core logic of an application from the user interface. Part of what makes BDK so powerful is the ability to build bitcoin applications in a wide variety of languages with uncompromising functionality. To deliver the optimal experience for our users, it is important to consider the foundations we set early in our development. When you choose BDK as part of the core of an application, you can concentrate on what the end user sees, instead of the wallet functionality. In my experience, Swift and SwiftUI are the best tools to deliver an iOS experience to users, and it can a great development experience too.