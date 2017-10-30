# Composing RIBs

> Note: If you haven't completed [tutorial 1](tutorial-1-ios) yet, we encourage you to do so before jumping into this tutorial.

Welcome to the RIBs tutorials, which have ben designed to give you a hands-on walkthrough through the core concepts of RIBs. As part of the tutorials, you'll be building a simple TicTacToe game using the RIBs architecture and associated tooling.

For tutorial 2, we'll start off where tutorial 1 ended. You can either continue with the project you completed in tutorial 1, or use the source code [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial2). Follow the [README](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial2/README.md) to install and open the project before reading any further.

## Goals

The main goals of this exercise to understand the following concepts:

- Child RIB calling up to parent RIB.
- Attaching/detaching a child RIB when the parent decides so.
- Creating a view-less RIB.
  - Cleanup view modifications when view-less RIB is detached.
- Attaching a child RIB when the parent RIB first loads up.
  - Lifecycle of a RIB.
- Unit testing.

## Inform Root RIB from LoggedOut RIB that the players have logged in

Update LoggedOutListener to add a method that allows LoggedOut RIB to inform Root RIB that players did login.
```swift
protocol LoggedOutListener: class {
    func didLogin(withPlayer1Name player1Name: String, player2Name: String)
}
```

Update LoggedOutInteractor's implementation of login method to perform the business logic of handling nil player names, as well as calling to LoggedOutListener to inform Root RIB that players did login. 
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

## Attach view-less LoggedIn RIB and detach LoggedOut RIB on button tap

Delete DELETE\_ME.swift, it was only required to stub out classes you're about to implement.

Update RootRouting in RootInteractor.swift to add a method to route to LoggedIn RIB.
```swift
protocol RootRouting: ViewableRouting {
    func routeToLoggedIn(withPlayer1Name player1Name: String, player2Name: String)
}
```

Invoke RootRouting in RootInteractor to route to LoggedIn, as the LoggedOutListener implementation. 
```swift
func didLogin(withPlayer1Name player1Name: String, player2Name: String) {
    router?.routeToLoggedIn(withPlayer1Name: player1Name, player2Name: player2Name)
}
```
Create LoggedIn RIB using Xcode templates as a view-less RIB. Uncheck the "Owns corresponding view" box.

Pass LoggedInBuildable protocol into RootRouter via constructor injection.
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

Update the RootBuilder to instantiate LoggedInBuilder concrete class and inject into RootRouter. 
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
For now just pass RootComponent as the dependency for LoggedInBuilder. We'll cover what this means in [tutorial3](../tutorial3-rib-di-and-communication).

RootRouter only uses and depends on LoggedInBuildable, instead of the concrete LoggedInBuilder class. This allows us to mock the LoggedInBuildable when unit testing RootRouter. This is a constraint by Swift, where swizzling based mocking is not possible. At the same time, this also follows the protocol-based programming principle, ensuring RootRouter and LoggedInBuilder are not tightly coupled.

Implement the routeToLoggedIn method in RootRouter. 
```swift
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

We need to first detach the LoggedOutRouter and dismiss its view. This means we need to add a new method in RootViewControllable, which is the protocol for  our current scope, Root. 
```swift
protocol RootViewControllable: ViewControllable {
    func present(viewController: ViewControllable)
    func dismiss(viewController: ViewControllable)
}
```

Once we add the dismiss method. We are then required to provide an implementation in RootViewController. 
```swift
func dismiss(viewController: ViewControllable) {
    if presentedViewController === viewController.uiviewController {
        dismiss(animated: true, completion: nil)
    }
}
```
Then we can go back to RootRouter routeToLoggedIn method and build the LoggedInRouter.

And finally attach the LoggedInRouter.

Notice we don't need to call our RootViewControllable to show the LoggedIn RIB, since LoggedIn RIB is view-less. We do need to show the LoggedOut in the routeToLoggedOut method.

## Unit test RootRouter

Now that Root is complete, let’s unit test its router. The same process can be applied to unit test other parts of a RIB.

Create a swift file in TicTacToeTests/Root and call it RootRouterTests and put it in the TicTacToeTest test bundle.

Let’s write a test that verifies when we invoke routeToLoggedIn, the RootRouter invokes the LoggedInBuildable protocol and attaches the returned Router. Feel free to refer to the written test: [RootRouterTests.swift](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/source/source3.swift?raw=true).

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

## Solution

The completed solution can be found [here](../tutorial3-rib-di-and-communication).