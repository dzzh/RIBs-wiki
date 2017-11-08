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

Because LoggedIn RIB does not own its own view, yet it still needs to be able to show child RIBs views, we need one of its ancestors, in this case the parent Root RIB to provide the view.

Update RootViewController to conform to LoggedInViewControllable. 
```swift
// MARK: LoggedInViewControllable

extension RootViewController: LoggedInViewControllable {

}
```

Dependency inject the LoggedInViewControllable protocol. Don't worry about what this means or how it's done for now. This is the focus area for [tutorial3](../tutorial3-rib-di-and-communication). We'll revisit this portion in that tutorial. For now, override LoggedInBuilder.swift with [this code](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/source/source1.swift?raw=true).

Now LoggedIn RIB can show/hide its child RIBs views by invoking methods on LoggedInViewControllable.

## Attach OffGame RIB on LoggedIn didLoad

Create an OffGame RIB that displays a "Start Game" button. This is the same as creating the LoggedOut RIB in the [previous tutorial](../tutorial1-create-a-rib). Feel free to use the provided [OffGameViewController](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/source/source2.swift?raw=true) implementation to save time. Use the Xcode RIB template to create a RIB that owns its own view. Then paste in the provided [OffGameViewController](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/source/source2.swift?raw=true).

Pass in the OffGameBuildable protocol into LoggedInRouter via constructor injection. This is the same as how we just passed LoggedInBuildable into RootRouter. 
```swift
init(interactor: LoggedInInteractable,
     viewController: LoggedInViewControllable,
     offGameBuilder: OffGameBuildable) {
    self.offGameBuilder = offGameBuilder
    super.init(interactor: interactor, viewController: viewController)
    interactor.router = self
}
```

Update the LoggedInBuilder to instantiate OffGameBuilder concrete class and inject into LoggedInRouter. This is the same as how we just instantiated LoggedInBuilder. 
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

For now just pass RootComponent as the dependency for LoggedInBuilder. We'll cover what this means in the [tutorial3](../tutorial3-rib-di-and-communication).

Implement attachOffGame method in LoggedInRouter to build and attach OffGame RIB and present its view controller. 
```swift
private var currentChild: ViewableRouting?

private func attachOffGame() {
    let offGame = offGameBuilder.build(withListener: interactor)
    self.currentChild = offGame
    attachChild(offGame)
    viewController.present(viewController: offGame.viewControllable)
}
```

Invoke attachOffGame in didLoad method of LoggedInRouter. 
```swift
override func didLoad() {
    super.didLoad()
    attachOffGame()
}
```

## Cleanup LoggedIn attached views when LoggedIn is detached

Because LoggedIn RIB doesn't own its own view, but rather uses a protocol to modify the view hierarchy one of its ancestors, in this case Root, provided, when Root detaches LoggedIn, Root has no way to directly remove the view modifications LoggedIn may have performed. Fortunately, the Xcode templates we used to generate the view-less LoggedIn RIB already provides a hook for us to clean up the view modifications when LoggedIn is detached/deactivated.

Add a dismiss method to the LoggedInViewControllable protocol. 
```swift
protocol LoggedInViewControllable: ViewControllable {
    func present(viewController: ViewControllable)
    func dismiss(viewController: ViewControllable)
}
```
Similar to other protocol declarations, this declares that LoggedIn RIB **needs** the functionality of dismissing a ViewControllable.

Dismiss the currentChild's view controller in cleanupViews method. 
```swift
func cleanupViews() {
    if let currentChild = currentChild {
        viewController.dismiss(viewController: currentChild.viewControllable)
    }
}
```

This method is invoked from the LoggedInInteractor when it resigns active.

## Attach TicTacToe RIB and detach OffGame RIB on button tap

This step is very similar to attaching LoggedIn RIB and detaching LoggedOut RIB when the "Login" button it tapped. To save time, the TicTacToe RIB is provided already. In order to route to TicTacToe, we should implement routeToTicTacToe in LoggedInRouter and wire up the button tap event from OffGameViewController to OffGameInteractor and then finally to LoggedInInteractor.

## Attach OffGame RIB and detach TicTacToe RIB when we have a winner

This is very similar to the other listener based routing we've already exercised. If you used the TicTacToe RIB provided above, a listener is already setup. We just need to implement it in the LoggedInInteractor.

Declare routeToOffGame in LoggedInRouting protocol. 
```swift
protocol LoggedInRouting: Routing {
    func cleanupViews()
    func routeToTicTacToe()
    func routeToOffGame()
}
```

Implement gameDidEnd method in LoggedInInteractor. 
```swift
// MARK: - TicTacToeListener

func gameDidEnd() {
    router?.routeToOffGame()
}
```

Implement routeToOffGame in LoggedInRouter.
```swift
func routeToOffGame() {
    detachCurrentChild()
    attachOffGame()
}
```
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

