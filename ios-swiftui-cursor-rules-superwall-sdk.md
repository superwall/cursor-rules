You are developing an iOS application utilizing the Superwall SDK for paywall and subscription management. All related code implementations should be in Swift. For all Superwall SDK questions, refer to the code samples found below. The critical parts of the SDK are documented.

If you cannot find the relevant code in the linked file, simply reply that you don't have information on that part of the SDK, and refer them to the documentation here: https://superwall.com/docs/

Here are the Superwall SDK docs:

# Superwall SDK Integration Guide

## Overview

Documentation for integrating the Superwall SDK into your iOS application. Superwall enables remote configuration and iteration of paywalls, allowing you to optimize monetization strategies without frequent app updates.

## Steps to Integrate Superwall SDK

### 1. Install the SDK

You can install the Superwall SDK using either Swift Package Manager or CocoaPods.

#### Using Swift Package Manager:

- In Xcode, navigate to `File` > `Add Packages...`.
- Enter the following URL in the search bar: `https://github.com/superwall-me/Superwall-iOS`.
- Set the dependency rule to `Up To Next Major Version` with a minimum version of `4.0.0`.
- Click `Add Package` and ensure your app target is selected.

#### Using CocoaPods:

- Ensure your project is set up with CocoaPods. If not, refer to the @CocoaPods Getting Started Guide.
- Add the following line to your `Podfile`:
  ```
  pod 'SuperwallKit', '< 5.0.0'
  ```
- Run `pod repo update` to update your local spec repo.
- Run `pod install` to install the Superwall SDK.
- Ensure your target's Build Settings > User Script Sandboxing is set to `No`.

For detailed instructions, refer to the @Superwall integration guide.

## 2. Configure the SDK

Initialize the Superwall SDK early in your app's lifecycle, typically in the `AppDelegate` or the main entry point of your SwiftUI app.

For SwiftUI based apps, initialize the SDK like this, in your `App` file:

```swift
import SwiftUI
import SuperwallKit

@main
struct MyApp: App {

    init() {
        // Use your own API key here
        Superwall.configure(apiKey: "YOUR_API_KEY")
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

For `AppDelegate` based apps:

```swift
import UIKit
import SuperwallKit

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        Superwall.configure(apiKey: "YOUR_API_KEY")
        return true
    }
}
```

Replace `"YOUR_API_KEY"` with your actual Superwall public API key, which can be found in the Superwall dashboard under `Settings` > `Keys` > `Public API Key`.

For more details, see the @Superwall SDK configuration documentation.

## 3. Registering Placements to Present a Paywall

To display a paywall, register a placement and trigger it based on user actions or app events. For example:

```swift
Superwall.shared.register(placement: "caffeineLogged") {
  store.log(amountToLog)
}
```

This will show the paywall if the user isn't subscribed (unless you've got different rules setup in the campaign filter to show it to paying users).

## Using Placement Parameters

Placement parameters allow you to send additional data when registering placements, enabling dynamic paywall customization and audience segmentation.

### Registering a Placement with Parameters

Include a `params` dictionary when registering a placement:

```swift
Superwall.shared.register(placement: "caffeineLogged", params: ["via": "logging_page"]) {
    store.log(amountToLog)
}
```

### Utilizing Placement Parameters

1. **Audience Filtering:** Use parameters to filter audiences, showing different paywalls based on where the event occurred.
2. **Templating in Text:** Parameters can populate dynamic text in paywalls:
   ```
   Hey {{user.firstName}}! FitnessAI offers tons of {{user.fitnessGoal}} workouts to help you reach your goals :)
   ```
3. **Analytics Integration:** Use parameters in analytics for cohort analysis.
4. **Dynamic Paywall Content:** Adjust paywall images or text dynamically based on parameter values.

### Registering placements with the presentation handler

You can provide a `PaywallPresentationHandler` to `register`, whose functions provide status updates for a paywall:

- `onDismiss`: Called when the paywall is dismissed. Accepts a `PaywallInfo` object containing info about the dismissed paywall, and there is a `PaywallResult` informing you of any transaction.
- `onPresent`: Called when the paywall did present. Accepts a `PaywallInfo` object containing info about the presented paywall.
- `onError`: Called when an error occurred when trying to present a paywall. Accepts an `Error` indicating why the paywall could not present.
- `onSkip`: Called when a paywall is skipped. Accepts a `PaywallSkippedReason` enum indicating why the paywall was skipped.

```swift Swift
let handler = PaywallPresentationHandler()
handler.onDismiss { paywallInfo, result in
  print("The paywall dismissed. PaywallInfo: \(paywallInfo). Result: \(result)")
}
handler.onPresent { paywallInfo in
  print("The paywall presented. PaywallInfo:", paywallInfo)
}
handler.onError { error in
  print("The paywall presentation failed with error \(error)")
}
handler.onSkip { reason in
  switch reason {
  case .holdout(let experiment):
    print("Paywall not shown because user is in a holdout group in Experiment: \(experiment.id)")
  case .noAudienceMatch:
    print("Paywall not shown because user doesn't match any audiences.")
  case .placementNotFound:
    print("Paywall not shown because this placement isn't part of a campaign.")
  }
}

Superwall.shared.register(placement: "campaign_trigger", handler: handler) {
  // Feature launched
}
```

## Viewing Purchased Products

When a paywall is presenting and a user converts, you can view the purchased products in several different ways.

### Use the `PaywallPresentationHandler`

Arguably the easiest of the options — simply pass in a presentation handler and check out the product within the `onDismiss` block.

```swift Swift
let handler = PaywallPresentationHandler()
handler.onDismiss { _, result in
  switch result {
  case .declined:
      print("No purchased occurred.")
  case .purchased(let product):
      print("Purchased \(product.productIdentifier)")
  case .restored:
      print("Restored purchases.")
  }
}

Superwall.shared.register(placement: "caffeineLogged", handler: handler) {
  logCaffeine()
}
```

### Use `SuperwallDelegate`

Next, the @SuperwallDelegate offers up much more information, and can inform you of virtually any Superwall event that occurred:

```swift Swift
class SWDelegate: SuperwallDelegate {
  func handleSuperwallEvent(withInfo eventInfo: SuperwallEventInfo) {
    switch eventInfo.event {
    case .transactionComplete(_, let product, _, _):
      print("Transaction complete: product: \(product.productIdentifier)")
    case .subscriptionStart(let product, _):
      print("Subscription start: product: \(product.productIdentifier)")
    case .freeTrialStart(let product, _):
      print("Free trial start: product: \(product.productIdentifier)")
    case .transactionRestore(_, _):
      print("Transaction restored")
    case .nonRecurringProductPurchase(let product, _):
      print("Consumable product purchased: \(product.id)")
    default:
      print("Unhandled event.")
    }
  }
}

@main
struct Caffeine_PalApp: App {
    @State private var swDelegate: SWDelegate = .init()

    init() {
        Superwall.configure(apiKey: "my_api_key")
        Superwall.shared.delegate = swDelegate
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

## Identity Management

### Anonymous Users

Superwall automatically generates a random user ID that persists internally until the user deletes/reinstalls your app.

You can call `Superwall.shared.reset()` to reset this ID and clear any paywall assignments.

If you use your own user management system, call identify(userId:options:) when you have a user’s identity. This will alias your userId with the anonymous Superwall ID enabling us to load the user’s assigned paywalls.

Calling Superwall.shared.reset() will reset the on-device userId to a random ID and clear the paywall assignments.

Note that for Android apps, if you want the userId passed to the Play Store when making purchases, you’ll also need to set passIdentifiersToPlayStore via SuperwallOptions. Be aware of Google’s rules that the userId must not contain any personally identifiable information, otherwise the purchase could be rejected.

```swift
// After retrieving a user's ID, e.g. from logging in or creating an account
Superwall.shared.identify(userId: user.id)

// When the user signs out
Superwall.shared.reset()
```

## Retrieving and Presenting a Paywall Yourself

If you want complete control over the paywall presentation process, you can use getPaywall(forPlacement:params:paywallOverrides:delegate:). This returns the UIViewController subclass PaywallViewController, which you can then present however you like. Or, you can use a SwiftUI View via PaywallView. The following is code is how you’d mimic register:

```swift
final class MyViewController: UIViewController {
  private func presentPaywall() async {
    do {
      // 1
  	  let paywallVc = try await Superwall.shared.getPaywall(
        forPlacement: "campaign_trigger",
        delegate: self
      )
   	  self.present(paywallVc, animated: true)
    } catch let skippedReason as PaywallSkippedReason {
      // 2
      switch skippedReason {
       case .holdout,
       .noAudienceMatch,
       .placementNotFound:
         break
       }
    } catch {
      // 3
      print(error)
    }
  }

  private func launchFeature() {
    // Insert code to launch a feature that's behind your paywall.
  }
}

// 4
extension MyViewController: PaywallViewControllerDelegate {
  func paywall(
    _ paywall: PaywallViewController,
    didFinishWith result: PaywallResult,
    shouldDismiss: Bool
  ) {
    if shouldDismiss {
      paywall.dismiss(animated: true)
    }

    switch result {
    case .purchased,
      .restored:
      launchFeature()
    case .declined:
      let closeReason = paywall.info.closeReason
      let featureGating = paywall.info.featureGatingBehavior
      if closeReason != .forNextPaywall && featureGating == .nonGated {
        launchFeature()
      }
    }
  }
}
```

This does the following:

Gets the paywall view controller.
Handles the cases where the paywall was skipped.
Catches any presentation errors.
Implements the delegate. This is called when the user is finished with the paywall. First, it checks shouldDismiss. If this is true then is dismissed the paywall from view before launching any features. This may depend on the result depending on how you first presented your view. Then, it switches over the result. If the result is purchased or restored the feature can be launched. However, if the result is declined, it checks that the the featureGating property of paywall.info is nonGated and that the closeReason isn’t .forNextPaywall.

## Using the Superwall Delegate

Use a Superwall delegate to help interface with 3rd party analytics, see which product was purchased on a paywall, handle custom placements and more.
Use a Superwall’s delegate to extend our SDK’s functionality across several surface areas by assigning to the delegate property:

```swift
class SWDelegate: SuperwallDelegate {
    // Implement delegate methods here
}

// After configuring the SDK...
Superwall.shared.delegate = SWDelegate()
```

Most of what occurs in Superwall can be viewed using the delegate method to respond to events, here are some examples:

```swift
class SWDelegate: SuperwallDelegate {
  func handleSuperwallEvent(withInfo eventInfo: SuperwallEventInfo) {
    switch eventInfo.event {
    case .transactionComplete(let transaction, let product, let paywallInfo):
      print("Converted from paywall originalTransactionIdentifier: \(transaction?.originalTransactionIdentifier ?? "")")
      print("Converted from paywall storeTransactionId: \(transaction?.storeTransactionId ?? "")")
      print("Converted from paywall productIdentifier: \(product.productIdentifier)")
      print("Converted from paywall paywallInfo: \(paywallInfo.identifier)")
    case .transactionRestore(let restoreType, let paywallInfo):
      print("transactionRestore restoreType \(restoreType)")
    case let .customPlacement(name, params, paywallInfo):
      // Forward Mixpanel/Ampltiude/etc
      print("\(name) - \(params) - \(paywallInfo)")
    default:
      // And several more events to use...
      print("Default event: \(eventInfo.event.description)")
    }
  }
}
```

Using the custom tap action, you can respond to any arbitrary event from a paywall:

```swift
class SWDelegate: SuperwallDelegate {
  func handleCustomPaywallAction(withName name: String) {
    if name == "showHelpCenter" {
      DispatchQueue.main.asyncAfter(deadline: .now() + 0.33) {
        self.showHelpCenter.toggle()
      }
    }
  }
}
```

You can be informed of subscription status changes using the delegate. If you need to set or handle the status on your own, use a purchase controller — this function is only for informational, tracking or similar purposes:

```swift
class SWDelegate: SuperwallDelegate {
  func subscriptionStatusDidChange(from oldValue: SubscriptionStatus, to newValue: SubscriptionStatus) {
    // Log or handle subscription change in your Ui
  }
}
```

The delegate also has callbacks for several paywall events, such dismissing, presenting, and more. Here’s an example:

```swift
class SWDelegate: SuperwallDelegate {
  func didPresentPaywall(withInfo paywallInfo: PaywallInfo) {
    // paywallInfo will contain all of the presented paywall's info
  }
}
```

### Using a PurchaseController

By default, Superwall handles basic subscription-related logic for you:

Purchasing: When the user initiates a checkout on a paywall.
Restoring: When the user restores previously purchased products.
Subscription Status: When the user’s subscription status changes to active or expired (by checking the local receipt).
However, if you want more control, you can pass in a PurchaseController when configuring the SDK via configure(apiKey:purchaseController:options:) and manually set Superwall.shared.subscriptionStatus to take over this responsibility.

​
Step 1: Creating a PurchaseController

A PurchaseController handles purchasing and restoring via protocol methods that you implement. You pass in your purchase controller when configuring the SDK:

```swift
import StoreKit
import SuperwallKit

final class SWPurchaseController: PurchaseController {
  // MARK: Sync Subscription Status
  /// Makes sure that Superwall knows the customer's subscription status by
  /// changing `Superwall.shared.subscriptionStatus`
  func syncSubscriptionStatus() async {
    var products: Set<String> = []
    for await verificationResult in Transaction.currentEntitlements {
      switch verificationResult {
      case .verified(let transaction):
        products.insert(transaction.productID)
      case .unverified:
        break
      }
    }

    let storeProducts = await Superwall.shared.products(for: products)
    let entitlements = Set(storeProducts.flatMap { $0.entitlements })

    await MainActor.run {
      Superwall.shared.subscriptionStatus = .active(entitlements)
    }
  }

  // MARK: Handle Purchases
  /// Makes a purchase with Superwall and returns its result after syncing subscription status. This gets called when
  /// someone tries to purchase a product on one of your paywalls.
  func purchase(product: StoreProduct) async -> PurchaseResult {
    let result = await Superwall.shared.purchase(product)
    await syncSubscriptionStatus()
    return result
  }

  // MARK: Handle Restores
  /// Makes a restore with Superwall and returns its result after syncing subscription status.
  /// This gets called when someone tries to restore purchases on one of your paywalls.
  func restorePurchases() async -> RestorationResult {
    let result = await Superwall.shared.restorePurchases()
    await syncSubscriptionStatus()
    return result
  }
}

```

Here’s what each method is responsible for:

Purchasing a given product. In here, enter your code that you use to purchase a product. Then, return the result of the purchase as a PurchaseResult. For Flutter, this is separated into purchasing from the App Store and Google Play. This is an enum that contains the following cases, all of which must be handled:
.cancelled: The purchase was cancelled.
.purchased: The product was purchased.
.pending: The purchase is pending/deferred and requires action from the developer.
.failed(Error): The purchase failed for a reason other than the user cancelling or the payment pending.
Restoring purchases. Here, you restore purchases and return a RestorationResult indicating whether the restoration was successful.

Step 2: Configuring the SDK With Your PurchaseController

Pass your purchase controller to the configure(apiKey:purchaseController:options:) method:].

In UIKit:

```swift
// AppDelegate.swift

import UIKit
import SuperwallKit

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
  func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {

    Superwall.configure(
      apiKey: "MY_API_KEY",
      purchaseController: MyPurchaseController.shared // <- Handle purchases on your own
    )

    return true
  }
}
```

In SwiftUI:

```swift
@main
struct MyApp: App {

  init() {
    Superwall.configure(
      apiKey: "MY_API_KEY",
      purchaseController: MyPurchaseController.shared // <- Handle purchases on your own
    )
  }
  
  var body: some Scene {
    WindowGroup {
      ContentView()
    }
  }
}
```

Step 3: Keeping subscriptionStatus Up-To-Date

You must set Superwall.shared.subscriptionStatus every time the user’s subscription status changes, otherwise the SDK won’t know who to show a paywall to. This is an enum that has three possible cases:

.unknown: This is the default value. In this state, paywalls will not show and their presentation will be automatically delayed until subscriptionStatus changes to a different value.
.active(let entitlements): Indicates that the user has an active entitlement. Paywalls will not show in this state unless you remotely set the paywall to ignore subscription status. A user can have one or more active entitlement.
.inactive: Indicates that the user doesn’t have an active entitlement. Paywalls can show in this state.
Here’s how you might do this:

```swift
import SuperwallKit

func syncSubscriptionStatus() async {
  var purchasedProductIds: Set<String> = []

  // get all purchased product ids
  for await verificationResult in Transaction.currentEntitlements {
    switch verificationResult {
    case .verified(let transaction):
      purchasedProductIds.insert(transaction.productID)
    case .unverified:
      break
    }
  }

  // get store products for purchased product ids from Superwall
  let storeProducts = await Superwall.shared.products(for: purchasedProductIds)

  // get entitlements from purchased store products
  let entitlements = Set(storeProducts.flatMap { $0.entitlements })

  // set subscription status
  await MainActor.run {
    Superwall.shared.subscriptionStatus = .active(entitlements)
  }
}
```

Listening for subscription status changes

If you need a simple way to observe when a user’s subscription status changes, on iOS you can use the Publisher for it. Here’s an example:

```swift
subscribedCancellable = Superwall.shared.$subscriptionStatus
  .receive(on: DispatchQueue.main)
  .sink { [weak self] status in
    switch status {
    case .unknown:
      self?.subscriptionLabel.text = "Loading subscription status."
    case .active(let entitlements):
      self?.subscriptionLabel.text = "You currently have an active subscription: \(entitlements.map { $0.id }). Therefore, the paywall will not show unless feature gating is disabled."
    case .inactive:
      self?.subscriptionLabel.text = "You do not have an active subscription so the paywall will show when clicking the button."
    }
  }
```

## Using RevenueCat with a PurchaseController

If you want to use RevenueCat to handle your subscription-related logic with Superwall, follow this guide.

ntegrate RevenueCat with Superwall in one of two ways:

Using a purchase controller: Use this route if you want to maintain control over purchasing logic and code.
Using PurchasesAreCompletedBy: Here, you don’t use a purchase controller and you tell RevenueCat that purchases are completed by your app using StoreKit. In this mode, RevenueCat will observe the purchases that the Superwall SDK makes. For more info see here.
​

1. Create a PurchaseController
   Create a new file called RCPurchaseController.swift or RCPurchaseController.kt, then copy and paste the following:

```swift
import SuperwallKit
import RevenueCat
import StoreKit

enum PurchasingError: LocalizedError {
  case sk2ProductNotFound

  var errorDescription: String? {
    switch self {
    case .sk2ProductNotFound:
      return "Superwall didn't pass a StoreKit 2 product to purchase. Are you sure you're not "
        + "configuring Superwall with a SuperwallOption to use StoreKit 1?"
    }
  }
}

final class RCPurchaseController: PurchaseController {
  // MARK: Sync Subscription Status
  /// Makes sure that Superwall knows the customer's entitlements by
  /// changing `Superwall.shared.entitlements`
  func syncSubscriptionStatus() {
    assert(Purchases.isConfigured, "You must configure RevenueCat before calling this method.")
    Task {

      for await customerInfo in Purchases.shared.customerInfoStream {
        // Gets called whenever new CustomerInfo is available
        let superwallEntitlements = customerInfo.entitlements.activeInCurrentEnvironment.keys.map {
          Entitlement(id: $0)
        }
        await MainActor.run { [superwallEntitlements] in
          Superwall.shared.subscriptionStatus = .active(Set(superwallEntitlements))
        }
      }
    }
  }

  // MARK: Handle Purchases
  /// Makes a purchase with RevenueCat and returns its result. This gets called when
  /// someone tries to purchase a product on one of your paywalls.
  func purchase(product: SuperwallKit.StoreProduct) async -> PurchaseResult {
    do {
      guard let sk2Product = product.sk2Product else {
        throw PurchasingError.sk2ProductNotFound
      }
      let storeProduct = RevenueCat.StoreProduct(sk2Product: sk2Product)
      let revenueCatResult = try await Purchases.shared.purchase(product: storeProduct)
      if revenueCatResult.userCancelled {
        return .cancelled
      } else {
        return .purchased
      }
    } catch let error as ErrorCode {
      if error == .paymentPendingError {
        return .pending
      } else {
        return .failed(error)
      }
    } catch {
      return .failed(error)
    }
  }

  // MARK: Handle Restores
  /// Makes a restore with RevenueCat and returns `.restored`, unless an error is thrown.
  /// This gets called when someone tries to restore purchases on one of your paywalls.
  func restorePurchases() async -> RestorationResult {
    do {
      _ = try await Purchases.shared.restorePurchases()
      return .restored
    } catch let error {
      return .failed(error)
    }
  }
}
```

2. Configure Superwall

Initialize an instance of RCPurchaseController and pass it in to Superwall.configure(apiKey:purchaseController):

```swift
let purchaseController = RCPurchaseController()

Superwall.configure(
  apiKey: "MY_API_KEY",
  purchaseController: purchaseController
)
```

3. Then sync the subscription status

```swift
func syncSubscriptionStatus() {
    assert(Purchases.isConfigured, "You must configure RevenueCat before calling this method.")
    Task {

      for await customerInfo in Purchases.shared.customerInfoStream {
        // Gets called whenever new CustomerInfo is available
        let superwallEntitlements = customerInfo.entitlements.activeInCurrentEnvironment.keys.map {
          Entitlement(id: $0)
        }
        await MainActor.run { [superwallEntitlements] in
          Superwall.shared.subscriptionStatus = .active(Set(superwallEntitlements))
        }
      }
    }
  }
```

## Using Observer Mode

If you wish to make purchases outside of Superwall’s SDK and paywalls, you can use observer mode to report purchases that will appear in the Superwall dashboard, such as transactions.

This is useful if you are using Superwall solely for revenue tracking, and you’re making purchases using frameworks like StoreKit or Google Play Billing Library directly. Observer mode will also properly link user identifiers to transactions. To enable observer mode, set it using SuperwallOptions when configuring the SDK:

```swift
let options = SuperwallOptions()
options.shouldObservePurchases = true
Superwall.configure(apiKey: "your_api_key", options: options)
```

There are a few things to keep in mind when using observer mode:

On iOS, if you’re using StoreKit 2, then Superwall solely reports transaction completions. If you’re using StoreKit 1, then Superwall will report transaction starts, abandons, and completions.
When using observer mode, you can’t make purchases using our SDK — such as Superwall.shared.purchase(aProduct).

## Direct Purchasing APIs

If you wish to purchase products directly on iOS and Android, use our SDK’s purchase methods.
If you’re using Superwall for revenue tracking, but want a hand with making purchases in your implementation, you can use our purchase methods:

```swift
// For StoreKit 1
private func purchase(_ product: SKProduct) async throws -> PurchaseResult {
  return await Superwall.shared.purchase(product)
}

// For StoreKit 2
private func purchase(_ product: StoreKit.Product) async throws -> PurchaseResult {
  return await Superwall.shared.purchase(product)
}

// Superwall's `StoreProduct`
private func purchase(_ product: StoreProduct) async throws -> PurchaseResult {
  return await Superwall.shared.purchase(product)
}
```

For iOS, the purchase() method supports StoreKit 1, 2 and Superwall’s abstraction over a product, StoreProduct. You can fetch the products you’ve added to Superwall via the products(for:) method. Similarly, in Android, you can fetch a product using a product identifier — and the first base plan will be selected:

```swift
private func fetchProducts(for identifiers: Set<String>) async -> Set<StoreProduct> {
    return await Superwall.shared.products(for: identifiers)
}
```

If you already have your own product fetching code, simply pass the product representation to these methods. For example, in StoreKit 1 — an SKProduct instance, in StoreKit 2, Product, etc. Each purchase() implementation returns a PurchaseResult, which informs you of the transaction’s resolution:

.cancelled: The purchase was cancelled.
.purchased: The product was purchased.
.pending: The purchase is pending/deferred and requires action from the developer.
.failed(Error): The purchase failed for a reason other than the user cancelling or the payment pending.
