# Uphold SDK for iOS [![Build Status](https://travis-ci.org/uphold/uphold-sdk-ios.svg?branch=master)](https://travis-ci.org/uphold/uphold-sdk-ios) [![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage) [![CocoaPods](https://img.shields.io/cocoapods/v/UpholdSdk.svg)](https://cocoapods.org/pods/UpholdSdk)

Uphold is a next generation platform that allows anyone to transfer and exchange value for free, instantly and securely.

The Uphold SDK for iOS provides an easy way for developers to integrate iOS applications with the [Uphold API](https://uphold.com/en/developer/api).

## Requirements

    * Xcode 8
    * Swift 3
    * Carthage or CocoaPods

## Installation

### Using [CocoaPods](https://cocoapods.org)

1. Add to your `Podfile`.

    ```
    platform :ios, '10.0'
    use_frameworks!

    # To use Uphold's production environment.
    pod 'UpholdSdk/Production'

    # To use Uphold's sandbox environment:
    # pod 'UpholdSdk/Sandbox'
    ```

2. Run `pod install`.

### Using [Carthage](https://github.com/Carthage/Carthage)

1. Add to your `Cartfile`.

    ```
    github "uphold/uphold-sdk-ios" ~> 0.17.0
    ```

2. Run `carthage update --platform iOS` specifying the build configuration to use Uphold's different environments.

    ```
    # To use Uphold's production environment.
    carthage update --platform iOS --configuration ProductionRelease

    # To use Uphold's sandbox environment:
    # carthage update --platform iOS --configuration SandboxRelease
    ```

## Basic usage

In order to learn more about the Uphold API, please visit the [developer website](https://uphold.com/en/developer).

To use the SDK you must first register an Application and obtain a unique `CLIENT_ID` and `CLIENT_SECRET` combination. We recommend your first app be [registered in the Sandbox environment](https://sandbox.uphold.com/dashboard/profile/applications/developer/new), so you can safely play around during development.

From the application page in your account you can get the `client id`, `client secret` , configure the `redirect URI` and the desired `scopes`.

### Authenticate User

In order to allow users to be re-directed back to the application after the authorization process, you’ll need to associate your custom `scheme` with your app by adding the following keys into the [`Info.plist`](https://github.com/uphold/uphold-sdk-ios/blob/master/SampleApplication/Info.plist) file:

* CFBundleURLTypes - The list of URLs types to be handled by the application.
    * CFBundleURLSchemes - The custom application schemes.

For instance, our demo application has the following configuration:

```xml
<key>CFBundleURLTypes</key>
    <array>
        <dict>
            <key>CFBundleURLSchemes</key>
            <array>
                <string>uphold-demo</string>
            </array>
        </dict>
    </array>
```

We start the authentication process by instantiating the UpholdClient and then calling the `beginAuthorization` method:

```swift
/// LoginViewController.swift

let upholdClient = UpholdClient()
let authorizationViewController = upholdClient.beginAuthorization(self, clientId: CLIENT_ID, scopes: scopes, state: state)
```

In the `AppDelegate` class you'll need to implement the method `application(application: UIApplication, openURL url: NSURL, options: [String : AnyObject]) -> Bool` that is called when the user completes the authorization process.

```swift
/// AppDelegate.swift

func application(application: UIApplication, openURL url: NSURL, options: [String : AnyObject]) -> Bool {
    loginViewController.completeAuthorization(url)

    return true
}
```

To complete the authorization process you'll need to call the `completeAuthorization` method from the `upholdClient` and get the user bearer token from the authentication response.

```swift
/// LoginViewController.swift

upholdClient.completeAuthorization(authorizationViewController, clientId: CLIENT_ID, clientSecret: CLIENT_SECRET, grantType: "authorization_code", state: state, uri: url).then { (response: AuthenticationResponse) -> () in
    // Get the user bearer token from the authenticationResponse.
}
```

To get the current user information, just instantiate the Uphold client with the user bearer token and then call the `getUser()` function:

```swift
let upholdClient = UpholdClient(bearertoken: bearerToken)

upholdClient.getUser().then { (user: User) -> () in
    /// The user information is available at the user object.
}
```

Note: Don't forget to add keychain sharing capabilities in your application's target inside the `Capabilities` tab.

### Get user cards with chaining

```swift
let upholdClient = UpholdClient(bearerToken: bearerToken)

upholdClient.getUser().then { (user: User) -> Promise<[Card]> in
    return user.getCards()
.then { (cards: [Card]) -> () in
    /// Do something with the list of cards.         
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.
}
```

### Get user cards

```swift
user.getCards().then { (cards: [Card]) -> () in
    /// Do something with the list of cards.        
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.            
}
```

### Create a new card for the user

```swift
let cardRequest: CardRequest = CardRequest(currency: "foo", label: "BTC", settings: CardSettings(position: 1, starred: true))

// Or just create a card without specifying the card settings.
// let cardRequest: CardRequest = CardRequest(currency: "foo", label: "BTC")

user.createCard(cardRequest).then { (card: Card) -> () in
    /// Do something with the card.        
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.            
}
```

### Create a new address for a card

```swift
let addressRequest: AddressRequest = AddressRequest(network: "bitcoin")

card.createAddress(addressRequest).then { (address: Address) -> () in
    /// Do something with the address of the card.        
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.            
}
```

### Get ticker

```swift
/// Instantiate the client. In this case, we don't need an
/// AUTHORIZATION_TOKEN because the Ticker endpoint is public.
let upholdClient = UpholdClient()

/// Get tickers.
upholdClient.getTickers().then { (rateList: [Rate]) -> () in
    /// Do something with the rates list.
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.
}
```

Or you could get a ticker for a specific currency:

```swift
/// Get tickers for BTC.
upholdClient.getTickersByCurrency("BTC").then { (rateList: [Rate]) -> () in
    /// Do something with the rates list.
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.
}
```

### Create and commit a new transaction

```swift
let transactionDenominationRequest = TransactionDenominationRequest(amount: "1.0", currency: "BTC")

/// A transaction to a destination (card id, crypto address, email, phone number or username).
let transactionTransferRequest = TransactionTransferRequest(denomination: transactionDenominationRequest, destination: "foo@bar.com")

card.createTransaction(transactionTransferRequest).then { (transaction: Transaction) -> () in
    /// Commit the transaction.
    transaction.commit(TransactionCommitRequest("Commit message"))
}.error({ (error: ErrorType) -> Void in
    /// Do something with the error.            
})

/// A transaction to a destination (card id, crypto address, email, phone number or username) with reference.
let transactionTransferRequest = TransactionTransferRequest(denomination: transactionDenominationRequest, destination: "foo@bar.com", reference: "123456")

card.createTransaction(transactionTransferRequest).then { (transaction: Transaction) -> () in
    /// Commit the transaction.
    transaction.commit(TransactionCommitRequest("Commit message"))
}.error({ (error: ErrorType) -> Void in
    /// Do something with the error.            
})

/// A deposit from an ACH or SEPA account.
let transactionDepositRequest = TransactionDepositRequest(denomination: transactionDenominationRequest, origin: "accountId")

card.createTransaction(transactionDepositRequest).then { (transaction: Transaction) -> () in
    /// Commit the transaction.
    transaction.commit(TransactionCommitRequest("Commit message"))
}.error({ (error: ErrorType) -> Void in
    /// Do something with the error.
})

/// A deposit from a credit card.
let transactionCardDepositRequest = TransactionCardDepositRequest(denomination: transactionDenominationRequest, origin: "creditCardId", securityCode: "1234")

card.createTransaction(transactionCardDepositRequest).then { (transaction: Transaction) -> () in
    /// Commit the transaction.
    transaction.commit(TransactionCommitRequest("Commit message"))
}.error({ (error: ErrorType) -> Void in
    /// Do something with the error.
})
```

If you want to commit the transaction on the creation process, call the `createTransaction` method with the first parameter set to `true`.

```swift
card.createTransaction(true, transactionRequest: transactionRequest)
```

### Get all public transactions

```swift
/// Instantiate the client. In this case, we don't need an
/// AUTHORIZATION_TOKEN because the Ticker endpoint is public.
let upholdClient = UpholdClient()
let paginator: Paginator<Transaction> = client.getReserve().getTransactions()

/// Get the list of transactions.
paginator.elements.then { (transactions: [Transaction]) -> () in
    /// Do something with the list of transactions.            
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.         
}

/// Get the next page of transactions.
paginator.getNext().then { (transactions: [Transaction]) -> () in
    /// Do something with the list of transactions.         
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.         
}
```

Or you could get a specific public transaction:

```swift
/// Get one public transaction.
upholdClient.getReserve().getTransactionById("a97bb994-6e24-4a89-b653-e0a6d0bcf634").then { (transaction: Transaction) -> () in
    /// Do something with the list of transactions.     
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.      
}
```

### Get reserve status

```swift
/// Instantiate the client. In this case, we don't need an
/// AUTHORIZATION_TOKEN because the Ticker endpoint is public.
let upholdClient = UpholdClient()

/// Get the reserve summary of all the obligations and assets within it.
client.getReserve().getStatistics().then { (reserveStatistics: [ReserveStatistics]) -> () in
    /// Do something with the reserve statistics.    
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.     
}
```

### Pagination
Some endpoints will return a paginator. Here are some examples on how to handle it:

```swift
/// Get public transactions paginator.
let paginator: Paginator<Transaction> = client.getReserve().getTransactions()

/// Get the first page of transactions.
paginator.elements.then { (transactions: [Transaction]) -> () in
    /// Do something with the list of transactions.            
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.         
}

/// Check if the paginator has a valid next page.
paginator.hasNext().then { (hasNext: Bool) -> () in
    /// Do something with the hasNext.     
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.         
}

/// Get the number of paginator elements.
paginator.count().then { (count: Int) -> () in
    /// Do something with the count.    
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.     
}

/// Get the next page.
paginator.getNext().then { (transactions: [Transaction]) -> () in
    /// Do something with the list of transactions.         
}.error { (error: ErrorType) -> Void in
    /// Do something with the error.         
}
```

## Uphold SDK sample

Check the [sample application](https://github.com/uphold/uphold-sdk-ios/tree/master/SampleApplication) to explore an application using the Uphold iOS SDK.

#### Building

To build the sample application you need the [Xcode](https://developer.apple.com/xcode/download/). Steps to build:

1. Clone the repository.
2. Get the project dependencies:

 ```
 carthage bootstrap --platform iOS
 ```

3. Open the sample project `SampleApplication.xcodeproj`.
4. Add keychain sharing capabilities.
5. Build and run the app from inside Xcode.

The sample application is configured to use the [sandbox environment](https://sandbox.uphold.com), make sure you use a sandbox account to perform the login.

## Contributing & Development

#### Contributing

Have you found a bug or want to suggest something? Please search the [issues](https://github.com/uphold/uphold-sdk-ios/issues) first and, if it is new, go ahead and [submit it](https://github.com/uphold/uphold-sdk-ios/issues/new).

#### Develop

It will be awesome if you can help us evolve `uphold-sdk-ios`. Want to help?

1. [Fork it](https://github.com/uphold/uphold-sdk-ios).
2. Hack away.
3. Run the tests.
5. Create a [Pull Request](https://github.com/uphold/uphold-sdk-ios/compare).
