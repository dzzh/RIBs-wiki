# Composing RIBs

> Note: If you haven't completed [tutorial 1](iOS-Tutorial-1) yet, we encourage you to do so before jumping into this tutorial.

Welcome to the RIBs tutorials which have been designed to give you a hands-on walkthrough through the core concepts of RIBs. As part of the tutorials, you'll be building a simple TicTacToe game using the RIBs architecture and associated tooling.

For this tutorial, we'll use the source code [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial2) as a starting point. Follow the [README](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial2/README.md) to install and open the project before reading any further.

## Goals

In the previous tutorial we have built an app that contains a login form powered by `LoggedOut` RIB. In this exercise, we will proceed from there and extend the application to show the game field after the user logs in. In the end of this tutorial, we will briefly explain how to unit test the RIBs.

The main goals of this exercise are to understand the following concepts:

- Having a child RIB communicate with its parent RIB.
- Attaching/detaching a child RIB when the parent interactor decides to do so.
- Creating a view-less RIB.
  - Cleaning up view modifications when a viewless RIB is detached.
- Attaching a child RIB when the parent RIB first loads up.
  - Understanding the lifecycle of a RIB.
- Unit testing a RIB.

## Project structure

After completing the previous tutorial, we ended up with an application consisting out of two RIBs, namely `Root` and `LoggedOut`. In this exercise, we will implement three additional RIBs, namely `LoggedIn`, `OffGame` and `TicTacToe`. By the end of this tutorial, our application will have the following RIBs hierarchy.

<p align="center">
<img src="https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/project-structure.png" width="580" alt="Project structure"/>
</p>

Here, `LoggedIn` RIB is viewless. Its only purpose is to switch between `TicTacToe` and `OffGame` RIBs. All the other RIBs include the own view controllers and can display views on screen.

`OffGame` RIB will allow the players to start a new game and will contain an interface with a "Start Game" button. `TicTacToe` RIB will display the game field and will allow the players to make their moves. 

## Communicating with a parent RIB

After the user types in the player names and taps the "Login" button, he has to be forwarded to the "Start game" view. To support this, our active `LoggedOut` RIB will have to inform the `Root` RIB about the login action. After that, the root router will switch control from `LoggedOut` RIB to `LoggedIn` RIB. In its turn, the viewless `LoggedIn` RIB will load `OffGame` RIB and present its view controller on screen.

As the `Root` RIB is a parent of the `LoggedOut` RIB, its router is configured to be a listener of `LoggedOut`'s interactor. We have to forward the login event from the`LoggedOut` RIB to the `Root` RIB via this listener interface.

First, update the `LoggedOutListener` to add a method that allows the `LoggedOut` RIB to inform the `Root` RIB that the players have logged in.
```swift
protocol LoggedOutListener: class {
    func didLogin(withPlayer1Name player1Name: String, player2Name: String)
}
```

This forces any parent RIB of the `LoggedOut` RIB to implement the `didLogin` function and makes sure that the compiler enforces the contract between the parent and its children.

Change the implementation of `login` function inside the `LoggedOutInteractor` to add a newly declared listener call. 

```swift
func login(withPlayer1Name player1Name: String?, player2Name: String?) {
    let player1NameWithDefault = playerName(player1Name, withDefaultName: "Player 1")
    let player2NameWithDefault = playerName(player2Name, withDefaultName: "Player 2")
    listener?.didLogin(withPlayer1Name: player1NameWithDefault, player2Name: player2NameWithDefault)
}
```

With these changes, the listener of `LoggedOut` RIB will be notified after the user taps "Login" button in the RIB's view controller.

## Routing to `LoggedIn` RIB

As you can see from the diagram above, after the users log in, the `Root` RIB has to switch from the `LoggedOut` RIB to `LoggedIn` RIB. Let's write the routing code to support this.

Update `RootRouting` protocol to add a method to route to the `LoggedIn` RIB.

```swift
protocol RootRouting: ViewableRouting {
    func routeToLoggedIn(withPlayer1Name player1Name: String, player2Name: String)
}
```

This establishes the contract between the `RootInteractor` and its router, the `RootRouter`.

Invoke `RootRouting` in `RootInteractor` to route to the `LoggedIn` RIB by implementing the `LoggedOutListener` protocol. Being a parent of `LoggedOut` RIB, the `Root` RIB has to implement its listener interface. 
 
```swift
// MARK: - LoggedOutListener

func didLogin(withPlayer1Name player1Name: String, player2Name: String) {
    router?.routeToLoggedIn(withPlayer1Name: player1Name, player2Name: player2Name)
}
```

This will make the `Root` RIB to route to the `LoggedIn` RIB whenever the users log in. However, we don't yet have `LoggedIn` RIB implemented and cannot switch to it from the `Root` RIB. Let's implement the missing RIB.

Delete `DELETE\_ME.swift` file in the `LoggedIn` group, it was only required to stub the classes you're about to implement.

Next, create a `LoggedIn` RIB using Xcode templates as a viewless RIB. Uncheck the "Owns corresponding view" box and create the RIB in the `LoggedIn` group. Make sure that the newly created files are added to the TicTacToe target.

## Attaching a viewless `LoggedIn` RIB and detaching `LoggedOut` RIB when the users log in

To attach the newly created RIB, the root router has to be able to build it. We make this possible by passing the `LoggedInBuildable` protocol into the `RootRouter` via constructor injection. Modify the constructor of the `RootRouter` to look like this:

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

You'll also need to add a private `loggedInBuilder` constant for the `RootRouter`:

```swift
    // MARK: - Private

    private let loggedInBuilder: LoggedInBuildable

    ...
```

Then, update the `RootBuilder` to instantiate a `LoggedInBuilder` concrete class and inject it into the `RootRouter`. Modify the build function of the `RootBuilder` like so:

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

If you look at the code we just modified, we pass in `RootComponent` as a dependency for the `LoggedInBuilder` using constructor injection. Don't worry about why we do this right now, we'll cover this when we get to [tutorial 3](../tutorial3).

`RootRouter` depends on `LoggedInBuildable` protocol instead of the concrete `LoggedInBuilder` class. This allows us to pass a test mock for the `LoggedInBuildable` when unit-testing the `RootRouter`. This is a constraint of Swift, where swizzling-based mocking is not possible. At the same time, this also follows the protocol-based programming principle, ensuring `RootRouter` and `LoggedInBuilder` are not tightly coupled.

We have created all the boilerplate code for the `LoggedIn` RIB and made it possible for the `Root` RIB to instantiate it. Now, we can implement the `routeToLoggedIn` method in the `RootRouter`. 

A good place to add it is just before the `// MARK: - Private` section.
 
```swift
// MARK: - RootRouting

func routeToLoggedIn(withPlayer1Name player1Name: String, player2Name: String) {
    // Detach LoggedOut RIB.
    if let loggedOut = self.loggedOut {
        detachChild(loggedOut)
        viewController.dismiss(viewController: loggedOut.viewControllable)
        self.loggedOut = nil
    }

    let loggedIn = loggedInBuilder.build(withListener: interactor)
    attachChild(loggedIn)
}
```

As you can see from the code snippet above, to switch control to the child RIB the parent RIB has to detach an existing child, create a new child RIB and attach it instead of the detached one. In RIBs architecture, parent routers always attach the routers of their children. 

It is also a responsibility of the parent RIB to maintain consistency between RIB and view hierarchies. If a child RIB has a view controller, then the parent RIB should dismiss or present the child view controller when the child RIB is being detached or attached. Check the implementation of `routeToLoggedOut` method to understand how to attach a RIB that owns a view controller.

To be able to receive events from the newly created `LoggedIn` RIB, the `Root` RIB configures its interactor as the listener of the `LoggedIn` RIB. This happens when the `Root` RIB builds the child RIB in the code above. However, at this point the `Root` RIB doesn't yet implement the protocol allowing it to respond to `LoggedIn` RIB's requests.

RIBs are unforgiving when it comes to conforming to listener interfaces as they are protocol-based. We use protocols instead of some other implicit observation methods so that the compiler will return errors when any parent isn't consuming all the events of its children instead of failing silently at runtime. Now that we pass the `RootInteractable` as a listener to the `LoggedInBuilder`'s `build` method, the `RootInteractable` needs to conform to the `LoggedInListener` protocol. Let's add this conformance to the `RootInteractable`:

```swift
protocol RootInteractable: Interactable, LoggedOutListener, LoggedInListener {
    weak var router: RootRouting? { get set }
    weak var listener: RootListener? { get set }
}
```

To be able to detach the `LoggedOut` RIB and dismiss its view, we need to add a new `dismiss` method to the `RootViewControllable` protocol.

Modify the protocol to look like this:

```swift
protocol RootViewControllable: ViewControllable {
    func present(viewController: ViewControllable)
    func dismiss(viewController: ViewControllable)
}
```

Once we add the `dismiss` method to the protocol, we will have to implement it in the `RootViewController`. Just add it under the `present` method. 

```swift
func dismiss(viewController: ViewControllable) {
    if presentedViewController === viewController.uiviewController {
        dismiss(animated: true, completion: nil)
    }
}
```

Now, the `RootRouter` is able to correctly detach the `LoggedOut` RIB and dismiss its view controller when routing to `LoggedIn` RIB using `routeToLoggedIn` method that we had implemented earlier.

## Pass in `LoggedInViewControllable` instead of creating it

Since the `LoggedIn` RIB does not have its own view but still needs to be able to show the views of its child RIBs, the `LoggedIn` RIB needs to access the view of its ancestor. In our case, this view has to be provided by the `Root` RIB, a parent of the `LoggedIn` RIB. 

Update `RootViewController` to conform to `LoggedInViewControllable` by adding an extension to the end of the file:

```swift
// MARK: LoggedInViewControllable

extension RootViewController: LoggedInViewControllable {
}
```

We need to inject the `LoggedInViewControllable` instance into `LoggedIn` RIB. We'll not walk you through this right now, as this will be covered in [tutorial 3](../tutorial3-rib-di-and-communication). For now, just override the content of `LoggedInBuilder.swift` with [this code](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/source/source1.swift?raw=true).

Now the `LoggedIn` RIB can show and hide its child RIBs views by invoking methods on the `LoggedInViewControllable` implemented by the `Root` RIB.

## Attaching the `OffGame` RIB when the `LoggedIn` RIB loads

As mentioned previously, `LoggedIn` RIB is viewless, it can only switch between its child RIBs. Let's create the first of these child RIBs, a RIB called `OffGame` that will display a "Start Game" button and handle the taps on it. 

Follow the same instructions as in our [previous tutorial](../tutorial1) to create a RIB with a view. We'd suggest creating a new group for it called "OffGame".

Once you've created the RIB, implement its UI in `OffGameViewController` class. To save time, you can use [the provided implementation](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/source/source2.swift?raw=true).

Now, let's connect the newly created `OffGame` RIB with its parent `LoggedIn`. The `LoggedIn` RIB should be able to build the `OffGame` RIB and attach it as a child.

Change the constructor of the `LoggedInRouter` to declare a dependency on a `OffGameBuildable` instance. To do so, modify its constructor as suggested below: 

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

We'll also have to declare a new private constant to hold a reference to the `offGameBuilder`:
```swift
// MARK: - Private

...

private let offGameBuilder: OffGameBuildable
```

Now, update the `LoggedInBuilder` to instantiate a `OffGameBuilder` concrete class and inject it into the `LoggedInRouter` instance. Modify the `build` function like so:

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

To fulfill the dependency contract of the `OffGameBuilder`, we'll modify the `LoggedInComponent` class to conform to `OffGameComponent` (RIB dependencies and components are covered in greater detail in [tutorial 3](../tutorial3)):

```swift
final class LoggedInComponent: Component<LoggedInDependency>, OffGameDependency {
    
    fileprivate var loggedInViewController: LoggedInViewControllable {
        return dependency.loggedInViewController
    }
}
```

We want to show the start screen powered by the `OffGame` RIB immediately after the users log in. This means that the `LoggedIn` RIB will have to attach the `OffGame` RIB as soon as it loads. Let's override `didLoad` method of the `LoggedInRouter` to load `OffGame` RIB.

```swift
override func didLoad() {
    super.didLoad()
    attachOffGame()
}
```

`attachOffGame` will be a private method in the `LoggedInRouter` class used to build and attach the `OffGame` RIB and present its view controller. Add the implementation of this method to the end of `LoggedInRouter` class.

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

To instantiate `OffGameBuilder` inside `attachOffGame` method, we have to inject a `LoggedInInteractable` instance into it. This interactor will serve as the `OffGame`'s listener interface allowing the parent to receive and interpret events coming from the child RIB.

To receive `OffGame` RIB events, `LoggedInInteractable` has to conform to `OffGameListener` protocol. Let's add the protocol conformance to it.

```swift
protocol LoggedInInteractable: Interactable, OffGameListener {
    weak var router: LoggedInRouting? { get set }
    weak var listener: LoggedInListener? { get set }
}
```

Now, `LoggedIn` RIB will attach `OffGame` RIB after loading and will be able to listen to the events originating in this RIB.

## Cleaning up the attached views when the `LoggedIn` RIB is detached

Because the `LoggedIn` RIB doesn't have an own view but rather modifies the view hierarchy of its parent, the `Root` RIB has no way to automatically remove the view modifications the `LoggedIn` RIB may have performed. Fortunately, the Xcode template we used to generate the viewless `LoggedIn` RIB already provides a hook for us to clean up the view modifications when the `LoggedIn` RIB is detached.

Declare `present` and `dismiss` methods in `LoggedInViewControllable` protocol:

```swift
protocol LoggedInViewControllable: ViewControllable {
    func present(viewController: ViewControllable)
    func dismiss(viewController: ViewControllable)
}
```

Similar to other protocol declarations, this declares that the `LoggedIn` RIB **needs** the functionality of dismissing a `ViewControllable`.

Then we'll update the `cleanupViews` method of the `LoggedInRouter` to dismiss the  view controller of the current child RIB:

```swift
func cleanupViews() {
    if let currentChild = currentChild {
        viewController.dismiss(viewController: currentChild.viewControllable)
    }
}
```

`cleanupViews` method will be invoked by `LoggedInInteractor` when the parent RIB decides to detach the `LoggedIn` RIB. By dismissing the presented view controller in `cleanupViews`, we guarantee that the `LoggedIn` RIB won't leave its views in the view hierarchy of the parent RIB after being detached.

## Switching to `TicTacToe` RIB on tapping "Start Game" button

As we discussed earlier in this tutorial, the `LoggedIn` RIB should allow the users to switch between `OffGame` and `TicTacToe` RIBs, with the former RIB being responsible for showing "Start Game" screen and the latter drawing the game field and handling the moves made by the players. So far, we have only implemented the `OffGame` RIB and made sure that it gets control from the `LoggedIn` RIB after the users log in. Now, we need to implement the `TicTacToe` RIB and switch to it after the user taps "Start Game" button at `OffGame` RIB.

This step is very similar to attaching the `LoggedIn` RIB and detaching the `LoggedOut` RIB when the "Login" button is tapped. To save time, the `TicTacToe` RIB is already implemented and included into the project. 

In order to route to `TicTacToe`, you should implement `routeToTicTacToe` method in the `LoggedInRouter` class and wire up the button tap event from the `OffGameViewController` to the `OffGameInteractor` and finally to the `LoggedInInteractor`.

You should be able to do this without any help from us, right? After implementing the code, run the app, log in and tap "Start Game" button to make sure that the `TicTacToe` RIB loads and shows the game field.

When working on this exercise, we recommend you to name the new `OffGameListener`'s method as `startTicTacToe` as this method is already stubbed for the unit tests. Otherwise, you'll see compilation errors later on when building the unit tests target.

## Attaching the `OffGame` RIB and detaching the `TicTacToe` RIB when we have a winner

After the game is over, we want to switch from the `TicTacToe` RIB back to the `OffGame` RIB. To do so, we will use the same listener-based routing pattern we've already exercised. The provided `TicTacToe` RIB already has a listener set up. We just need to implement it in the `LoggedInInteractor` to allow the `LoggedIn` RIB to respond to `TicTacToe` events.

Declare the `routeToOffGame` method in the `LoggedInRouting` protocol. 

```swift
protocol LoggedInRouting: Routing {
    func routeToTicTacToe()
    func routeToOffGame()
    func cleanupViews()
}
```

Implement the `gameDidEnd` method in the `LoggedInInteractor` class:
 
```swift
// MARK: - TicTacToeListener

func gameDidEnd() {
    router?.routeToOffGame()
}
```

Then, implement the `routeToOffGame` in the `LoggedInRouter` class.

```swift
func routeToOffGame() {
    detachCurrentChild()
    attachOffGame()
}
```

Add the private helper method somewhere in your private section:

```swift
private func detachCurrentChild() {
    if let currentChild = currentChild {
        detachChild(currentChild)
        viewController.dismiss(viewController: currentChild.viewControllable)
    }
}
```

Now, the app will switch from the game screen to start screen after one of the players wins the game.

## Unit testing

Finally, we will demonstrate how to write unit tests for our app. Let's test our `RootRouter` class. The same principles can be applied for unit testing the other parts of a RIB, and there's even a tooling template that will create all the unit tests for the RIB for you.

Create a new swift file in `TicTacToeTests/Root` group and call it `RootRouterTests`. Add it to the `TicTacToeTest` target.

Let’s write a test that verifies the behavior of `routeToLoggedIn` method. When this method is called, the `RootRouter` should invoke the `build` method of `LoggedInBuildable` protocol and attach the returned router. We have already prepared an implementation of this test that is available [here](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial2-composing-ribs/source/source3.swift?raw=true), copy the code into `RootRouterTests` and make sure the test compiles and passes.

Let's explore the structure of the test that we just added.

As we're testing the `RootRouter`, we need to instantiate it. The router has a number of protocol-based dependencies that are instantiated with the mocks. All the mocks needed for this exercise are already provided in `TicTacToeMocks.swift` file. When writing the unit tests for the other RIBs, you'll have to create the mocks for them yourself.

When calling `routeToLoggedIn`, the implementation of our root router should call the `build` method of the `LoggedIn` RIB to instantiate its router. We don't want to copy the builder logic into our mocks, so instead we pass in the closure that returns us a router mock implementing the expected `LoggedInRouting` interface. This closure is configured before running the test. 

Working with handler closures is a common development pattern that we use heavily during the unit testing. Another such pattern is counting the number of method invocations. For example, from the implementation of the `routeToLoggedIn` method that we're testing we know that it should invoke the `build` method of `LoggedInBuildable` exactly once, so we check the call cound of the respective mock before and after calling the method under the test.


## Tutorial completed
If you didn't make any mistakes while following the instructions, you should be able to build and launch the project. If this is not the case and you believe that there are errors in this tutorial, please open an issue to help us fix them. 

Congratulations with completing the second tutorial! Now onwards to [tutorial 3](iOS-Tutorial-3).
