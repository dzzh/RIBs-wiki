# Composing RIBs

> Note: If you haven't completed [tutorial 1](iOS-Tutorial-1) yet, we encourage you to do so before jumping into this tutorial.

Welcome to the RIBs tutorials, which have ben designed to give you a hands-on walkthrough through the core concepts of RIBs. As part of the tutorials, you'll be building a simple TicTacToe game using the RIBs architecture and associated tooling.

For this tutorial, we'll use the source code [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial2) as a starting point. Follow the [README](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial2/README.md) to install and open the project before reading any further.

## Goals

The main goals of this exercise to understand the following concepts:

- Having a child RIB calling up to its parent RIB.
- Attaching/detaching a child RIB when the parent interactor decides to do so.
- Creating a view-less RIB.
  - Cleaning up view modifications when a viewless RIB is detached.
- Attaching a child RIB when the parent RIB first loads up.
  - Understanding the lifecycle of a RIB.
- Unit testing a RIB.

## Calling up to a parent RIB

We want to inform the Root RIB from LoggedOut RIB when the players have logged in. For this we need to implement the following code.

First, update the `LoggedOutListener` (in the LoggedOutInteractor.swift file) to add a method that allows LoggedOut RIB to inform Root RIB that players has logged in.
```swift
protocol LoggedOutListener: class {
    func didLogin(withPlayer1Name player1Name: String, player2Name: String)
}
```

This forces the any parent RIB of the LoggedOut RIB to implement the `didLogin` function and makes sure that the compiler enforces the contract between the parent and its children.

Add a login method implementation to the `LoggedOutInteractor` and have it perform the business logic of handling nil player names, as well as calling to LoggedOutListener to inform its parent (the Root RIB in our project) that the players have login. 
```swift
// MARK: - LoggedOutPresentableListener

func login(withPlayer1Name player1Name: String?, player2Name: String?) {
    let player1NameWithDefault = playerName(player1Name, withDefaultName: "Player 1")
    let player2NameWithDefault = playerName(player2Name, withDefaultName: "Player 2")

    listener?.didLogin(withPlayer1Name: player1NameWithDefault, player2Name: player2NameWithDefault)
}

private func playerName(_ name: String?, withDefaultName defaultName: String) -> String {
    if let name = name {
        return name.isEmpty ? defaultName : name
    } else {
        return defaultName
    }
}
```

## Attaching a viewless LoggedIn RIB and detach LoggedOut RIB when users sign in

Delete DELETE\_ME.swift (in the LoggedIn group), it was only required to stub out classes you're about to implement.

Update `RootRouting` in RootInteractor.swift to add a method to route to LoggedIn RIB.
```swift
protocol RootRouting: ViewableRouting {
    func routeToLoggedIn(withPlayer1Name player1Name: String, player2Name: String)
}
```

This establishes the contract between the `RootInteractor` and its router, the `RootRouter`.

Invoke `RootRouting` in `RootInteractor` to route to the LoggedIn RIB, by implementing the `LoggedOutListener` protocol.
 
```swift
// MARK: - LoggedOutListener

func didLogin(withPlayer1Name player1Name: String, player2Name: String) {
    router?.routeToLoggedIn(withPlayer1Name: player1Name, player2Name: player2Name)
}
```

This will make the Root RIB route to the LoggedIn RIB whenever the LoggedOut RIB says that users have logged in.

Next, create a LoggedIn RIB using Xcode templates as a viewless RIB. Uncheck the "Owns corresponding view" box create the RIB in the LoggedIn group. There are already some files in that group, but don't worry about them. Also make sure that the newly created files are added to the TicTacToe target.

The RootRouter now needs to be able to build the newly created LoggedIn RIB. We make this possible by passing in the LoggedInBuildable protocol into the RootRouter (in the RootRouter.swift file) via constructor injection. Modify the constructor of the RootRouter to look like this:

```swift
init(interactor: RootInteractable,
     viewController: RootViewControllable,
     loggedOutBuilder: LoggedOutBuildable,
     loggedInBuilder: LoggedInBuildable) {
    self.loggedOutBuilder = loggedOutBuilder
    self.loggedInBuilder = loggedInBuilder
    super.init(interactor: interactor, viewController: viewController)
    interactor.router = self
}
```

You'll also need to add a private loggedInBuilder instance constant for the RootRouter:

```swift
    // MARK: - Private

    private let loggedInBuilder: LoggedInBuildable

    ...
```

Then, update the RootBuilder to instantiate a LoggedInBuilder concrete class and inject it into the RootRouter. Modify the build function of the RootBuilder (yes, in the RootBuilder.swift file) like so:

```swift
func build() -> LaunchRouting {
    let viewController = RootViewController()
    let component = RootComponent(dependency: dependency,
                                  rootViewController: viewController)
    let interactor = RootInteractor(presenter: viewController)

    let loggedOutBuilder = LoggedOutBuilder(dependency: component)
    let loggedInBuilder = LoggedInBuilder(dependency: component)
    return RootRouter(interactor: interactor,
                      viewController: viewController,
                      loggedOutBuilder: loggedOutBuilder,
                      loggedInBuilder: loggedInBuilder)
}
```

If you look at the code we just modified, we pass in RootComponent as the dependency for the LoggedInBuilder using constructor injection. Don't worry about why we do this rignt now, we'll this when we get to [tutorial 3](../tutorial3).

`RootRouter` depends on `LoggedInBuildable`, instead of the concrete `LoggedInBuilder` class. This allows us to pass in a test mock for the `LoggedInBuildable` when unit testing the `RootRouter`. This is a constraint of Swift, where swizzling based mocking is not possible. At the same time, this also follows the protocol-based programming principle, ensuring RootRouter and LoggedInBuilder are not tightly coupled.

Now, lets implement the `routeToLoggedIn` method in the `RootRouter` (by now I'm sure you already know what file `RootRouter` is implemented in). A good place to add it is just before the `// MARK: - Private` section.
 
```swift
// MARK: - RootRouting

func routeToLoggedIn(withPlayer1Name player1Name: String, player2Name: String) {
    // Detach logged out.
    if let loggedOut = self.loggedOut {
        detachChild(loggedOut)
        viewController.dismiss(viewController: loggedOut.viewControllable)
        self.loggedOut = nil
    }

    let loggedIn = loggedInBuilder.build(withListener: interactor)
    attachChild(loggedIn)
}
```

When routing to the LoggedIn RIB, we first detach the `LoggedOutRouter` and dismiss its view. In order to be able to do so, we need to add a new method in the `RootViewControllable` protocol.

Modify the protocol look like this:

```swift
protocol RootViewControllable: ViewControllable {
    func present(viewController: ViewControllable)
    func dismiss(viewController: ViewControllable)
}
```

Once we add the `dismiss` method to the protocol, we are then required by the compiler to provide an implementation in the `RootViewController`. Just add it under the `present` method. 

```swift
func dismiss(viewController: ViewControllable) {
    if presentedViewController === viewController.uiviewController {
        dismiss(animated: true, completion: nil)
    }
}
```

Going back to the `RootRouter`'s `routeToLoggedIn`, it can now correctly dismiss the loggedOut RIB. The last two lines of the `routeToLoggedIn` method build the LoggedInRouter, and attach it as a child. Routers always attach the routers of their children.

Notice we don't need to call a `RootViewControllable` to show the LoggedIn RIB, as the `LoggedIn` RIB doesn't have a view. If you want to see what presenting a RIB with a view looks like, check out the `routeToLoggedOut` method.

## Pass in LoggedInViewControllable instead of creating it

Since the LoggedIn RIB does not own its own view and yet still needs to be able to show the views of it's child RIBs, the LoggedIn RIB needs to use the view of its ancestor. In our case, the parent Root RIB to provide the view.

Update `RootViewController` to conform to `LoggedInViewControllable`, by adding the following snippet to the end of the file:

```swift
// MARK: LoggedInViewControllable

extension RootViewController: LoggedInViewControllable {

}
```

Now we need to dependency inject the LoggedInViewControllable protocol. We'll not walk you throught this right now, as this is the focus area for [tutorial3](../tutorial3-rib-di-and-communication). For now, just override the content of LoggedInBuilder.swift with [this code](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/source/source1.swift?raw=true).

Now the LoggedIn RIB can show and hide its child RIBs views by invoking methods on the `LoggedInViewControllable`.

## Attaching the OffGame RIB when the LoggedIn RIB loads

Create a new RIB called OffGame, which is supposed to display a "Start Game" button. This is our splash screen for the game that is displayed when we haven't started the game yet. 

Follow the same instructions as in our [previous tutorial](../tutorial1) to create a RIB with a view. We'd suggest creating a new group for it called "OffGame".

Once you've created the RIB, implement the UI. Feel free to use the provided [OffGameViewController](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/source/source2.swift?raw=true) implementation to save time.

Then, we'll change the constructor of the `LoggedInRouter` to declare a dependency on a `OffGameBuildable` instance. Let's modify the constructor like so: 

```swift
init(interactor: LoggedInInteractable,
     viewController: LoggedInViewControllable,
     offGameBuilder: OffGameBuildable) {
    self.viewController = viewController
    self.offGameBuilder = offGameBuilder
    super.init(interactor: interactor)
    interactor.router = self
}
```

We'll also have to declare a two new private constant to hold the viewController and offGameBuilder:
```swift
// MARK: - Private

private let viewController: LoggedInViewControllable

private let offGameBuilder: OffGameBuildable
```

Then we'll update the `LoggedInBuilder` to instantiate a `OffGameBuilder` concrete class and inject it into the `LoggedInRouter` instance. Modify the build function like so:

```swift
func build(withListener listener: LoggedInListener) -> LoggedInRouting {
    let component = LoggedInComponent(dependency: dependency)
    let interactor = LoggedInInteractor()
    interactor.listener = listener

    let offGameBuilder = OffGameBuilder(dependency: component)
    return LoggedInRouter(interactor: interactor,
                          viewController: component.loggedInViewController,
                          offGameBuilder: offGameBuilder)
}
```

Then, we'll implement a `attachOffGame` private method in the `LoggedInRouter`class to build and attach the OffGame RIB and present its view controller. Add the following to the end of your class implementation.

```swift
// MARK: - Private

private var currentChild: ViewableRouting?

private func attachOffGame() {
    let offGame = offGameBuilder.build(withListener: interactor)
    self.currentChild = offGame
    attachChild(offGame)
    viewController.present(viewController: offGame.viewControllable)
}
```

RIBs is unforgivable when it comes to conforming to listener interfaces as they are protocol based. We're using protocols instead of some other implicit listeners, so that the compiler will give you errors when some parent isn't consuming all the events of its children instead of failing at runtime. Now that we pass the `LoggedInInteractable` as a listener to the `OffGameBuilder`'s `build` method, the `LoggedInInteractable` needs to conform to the `OffGameListener` protocol. Let's add this conformance to the `LoggedInInteractable` by adding the following code to the LoggedInInteractor.swift file:

```swift
extension LoggedInInteractor: OffGameListener {
}
```

Then we'll invoke the `attachOffGame` method whenever the `LoggedInRouter` loads. We'll do this by overriding the  `didLoad` method of the Router. Add the following code somewhere above your private implementations in the `LoggedInRouter` class.

```swift
override func didLoad() {
    super.didLoad()
    attachOffGame()
}
```

## Cleaning up attached views when the LoggedIn RIB is detached

Because the LoggedIn RIB doesn't own its own view, but rather uses a protocol to modify the view hierarchy one of its parents, the Root RIB has no way to automatically remove the view modifications the LoggedIn RIB may have performed. Fortunately, the Xcode templates we used to generate the viewless LoggedIn RIB already provides a hook for us to clean up the view modifications when LoggedIn is detached.

Add a present and dismiss method declaration to the LoggedInViewControllable protocol:

```swift
protocol LoggedInViewControllable: ViewControllable {
    func present(viewController: ViewControllable)
    func dismiss(viewController: ViewControllable)
}
```

Similar to other protocol declarations, this declares that LoggedIn RIB **needs** the functionality of dismissing a ViewControllable.

Then we'll update the cleanupViews method of the `LoggedInRouter` to dismiss the views the currentChild's view controller:

```swift
func cleanupViews() {
    if let currentChild = currentChild {
        viewController.dismiss(viewController: currentChild.viewControllable)
    }
}
```

This method is invoked from the `LoggedInInteractor` when it resigns its active state.

## Attaching the TicTacToe RIB and detach the OffGame RIB on button tap

This step is very similar to attaching LoggedIn RIB and detaching LoggedOut RIB when the "Login" button is tapped. To save time, the TicTacToe RIB is already included in your project. 

In order to route to TicTacToe, you should implement `routeToTicTacToe` in the `LoggedInRouter` class and wire up the button tap event from the `OffGameViewController` to the `OffGameInteractor` and finally to the `LoggedInInteractor`.

You should be able to do this without any help from us, right?

## Attaching the OffGame RIB and detaching the TicTacToe RIB when we have a winner

We want to get back to the OffGame RIB when the game is over. This is very similar to the other listener based routing we've already exercised. The provided TicTacToe RIB already has a listener set up. We just need to implement it in the `LoggedInInteractor`.

Declare the `routeToOffGame`method in the `LoggedInRouting` protocol. 

```swift
protocol LoggedInRouting: Routing {
    func routeToTicTacToe()
    func routeToOffGame()
}
```

Implement the `gameDidEnd` method in the `LoggedInInteractor` class:
 
```swift
// MARK: - TicTacToeListener

func gameDidEnd() {
    router?.routeToOffGame()
}
```

Then, implement the routeToOffGame in the `LoggedInRouter` class.

```swift
func routeToOffGame() {
    detachCurrentChild()
    attachOffGame()
}
```
Also add the private helper method somewhere in your private section:

```swift
private func detachCurrentChild() {
    if let currentChild = currentChild {
        detachChild(currentChild)
        viewController.dismiss(viewController: currentChild.viewControllable)
    }
}
```

## Unit testing

Finally, let's write some test for our app. Let's unit test our `RootRouter` The same principles can be applied to unit testing other parts of a RIB, and there's even a tooling template that will create all the unit tests for a RIB for you.

Create a swift file in the TicTacToeTests/Root group and call it RootRouterTests. Add it to the TicTacToeTest bundle.

Let’s write a test that verifies when we invoke `routeToLoggedIn`, the `RootRouter` invokes the `LoggedInBuildable` protocol and attaches the returned Router. Feel free to refer to the written test: [RootRouterTests.swift](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/source/source3.swift?raw=true).


## Tutorial completed
You completed the second tutorial Now onwards to [tutorial 3](iOS-Tutorial-3).

