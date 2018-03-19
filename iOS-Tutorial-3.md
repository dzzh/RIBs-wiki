# RIBs Dependency Injection and Communication

> Note: If you haven't completed [tutorial 2](iOS-Tutorial-2) yet, we encourage you to do so before jumping into this tutorial.

Welcome to the RIBs tutorials, which have been designed to give you a hands-on walkthrough through the core concepts of RIBs. As part of the tutorials, you'll be building a simple TicTacToe game using the RIBs architecture and associated tooling.

For tutorial 3, we'll use the source code [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial3) as a starting point. Follow the [README](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial3/README.md) to install and open the project before reading any further.

In the previous tutorial we have built a simple TicTacToe game that consists out of five RIBs. Their structure is shown on a diagram below.

<p align="center">
<img src="https://github.com/dzzh/RIBs-wiki/blob/wip/assets/ribs-tutorials-102-1.png" width="580" alt="Project structure"/>
</p>

In this tutorial, we will not create the new RIBs but will modify the RIBs we already have in the project.

## Goals

There are several things that can be added to the start screen of the TicTacToe game. First of all, we want to know which players participate in the game, so we will display their names there. Secondly, if the players decide to play several games in a row, we will track the game score and also show it at the start screen.

The main goals of this tutorial are to explain the following concepts:
- Passing a dynamic dependency into a child RIB via the Builder’s `build` method.
- Passing static dependencies using the Dependency Injection tree (DI tree).
  - Extension based dependency conformance in Swift.
- Rx stream lifecycle management using the RIB lifecycle.

## Dynamic dependencies

In [tutorial 1](iOS-Tutorial-1), we have built a login form for the game and forwarded the player names from the `LoggedOut` RIB up to the `Root` RIB. In [tutorial 2](iOS-Tutorial-2), however, we didn't use these data on the newly built screens and didn't forward it to the new RIBs. In this tutorial, we will forward the player names down the RIB tree to `OffGame` and `TicTacToe` RIBs.

We start with passing the player names as dynamic dependencies from the `Root` RIB to the `LoggedIn` RIB via the `LoggedInBuilder`’s build method.

For this, we'll update the `LoggedInBuildable` protocol to include two player names as dynamic dependencies in addition to the existing `listener` dependency:

```swift
protocol LoggedInBuildable: Buildable {
    func build(withListener listener: LoggedInListener, 
               player1Name: String, 
               player2Name: String) -> LoggedInRouting
}
 ```

Then we'll update the implementation of the `LoggedInBuilder`'s `build` method:

```swift
func build(withListener listener: LoggedInListener, player1Name: String, player2Name: String) -> LoggedInRouting {
    let component = LoggedInComponent(dependency: dependency,
                                      player1Name: player1Name,
                                      player2Name: player2Name)
```

Finally, we'll update the `LoggedInComponent` initializer to put the player names onto the DI tree. We'll store them as constants in the `LoggedInComponent`:

```swift
let player1Name: String
let player2Name: String

init(dependency: LoggedInDependency, player1Name: String, player2Name: String) {
    self.player1Name = player1Name
    self.player2Name = player2Name
    super.init(dependency: dependency)
}
```

This effectively transforms the player names from dynamic dependencies provided by the `LoggedIn`’s parent to the static dependencies available to any of the `LoggedIn`’s children. 

Next, we'll update the `RootRouter` class to pass in the player names to the `LoggedInBuildable`’s `build` method:

```swift
func routeToLoggedIn(withPlayer1Name player1Name: String, player2Name: String) {
    // Detach logged out.
    if let loggedOut = self.loggedOut {
        detachChild(loggedOut)
        viewController.dismiss(viewController: loggedOut.viewControllable)
        self.loggedOut = nil
    }

    let loggedIn = loggedInBuilder.build(withListener: interactor, player1Name: player1Name, player2Name: player2Name)
    attachChild(loggedIn)
}
```

After all these changes, the player names that were entered by the user and handled by the `LoggedOut` RIB became available to the `LoggedIn` RIB and all its children.

### Dynamic dependencies vs static dependencies

As you can see, we decided to inject the player names into the `LoggedIn` RIB dynamically when building the RIB. Instead, we could have configured the `LoggedIn` RIB to resolve these dependencies statically by passing them down the RIBs tree. However, in this case we would have to make them optional, because the player names cannot be initialized when the `Root` RIB is created (for the sake of this exercise let's assume that initializing them with the default values is not an option here).

If we decided to deal with the optional values, we would have introduced additional complexity into the RIB code. The `LoggedIn` RIB and its children would have to handle `nil` values that could come instead of proper player names, and doing so would clearly be beyond their responsibilities. Instead, we should resolve the player names (or handle a `nil` value) as soon as possible after the names are entered by the user and remove this burden from the other parts of the app. Properly scoped dependencies allow us to make invariant assumptions, thus eliminating any unreasonable or unstable code.

## RIB's Dependencies and Components

We haven't discussed the dependencies and components in RIBs tutorials yet, so it makes sense to make a small detour and explain what they are before we proceed.

In RIBs terms, a Dependency is a protocol that lists the dependencies that a RIB needs from its parent to be properly instantiated. A Component is an implementation of the Dependency protocol. In addition to providing the parent dependencies to the RIB's Builder, the Component is also responsible for owning the dependencies that the RIB creates for itself and its children.

Usually, when a parent RIB instantiates a child RIB, it injects its own component into the child's Builder as a constructor dependency. Each injected Component decides what dependencies to expose to the children on its own. 

The dependencies contained in the Component usually either hold some state that has to be passed down the DI tree, or are costly to construct and are shared between the RIBs for performance reasons.

## Passing the player names to the `OffGame` scope using the DI tree

Now that we have resolved the player names and made sure they are valid when being injected into the `LoggedIn` RIB, we can safely pass them down the DI tree to the `OffGame` RIB to display them alongside the "Start Game" button.

For this, we'll declare the player names as dependencies in the `OffGameDependency` protocol:

```swift
protocol OffGameDependency: Dependency {
    var player1Name: String { get }
    var player2Name: String { get }
}
```

These static dependencies will have to be passed to the `OffGame` RIB during its initialization by the parent RIB.

As the next step, we'll make these dependencies available for the `OffGame`’s own scope using its component class defined in `OffGameComponent`.

```swift
final class OffGameComponent: Component<OffGameDependency> {
    fileprivate var player1Name: String {
        return dependency.player1Name
    }

    fileprivate var player2Name: String {
        return dependency.player2Name
    }
}
```

Notice that these properties are marked as `fileprivate`. This means they are only accessible within the `OffGameBuilder.swift` file and therefore are not exposed to the child scopes. We didn’t use the `fileprivate` access control in `LoggedInComponent` because we wanted to provide these values to the `OffGame` child scope.

Since we’ve already added the player names to the `LoggedInComponent` in the previous steps, there’s nothing else we need to do to make the `OffGame`’s parent scope - the `LoggedIn` scope - to satisfy the new dependencies we have just added.

Next, we'll pass these dependencies into the `OffGameViewController` via constructor injection to display them. We could also pass the dependencies into the `OffGameInteractor` first and let the interactor invoke the methods of its `OffGamePresentable` to display this information, but since showing the player names does not require any additional processing, we can directly pass them to the view controller to display. We'll use `player1Name` and `player2Name` constants in the `OffGameViewController` to store the values passed in through the constructor.

Let's update the `OffGameBuilder` to inject the dependencies into the view controller.

```swift
final class OffGameBuilder: Builder<OffGameDependency>, OffGameBuildable {
    override init(dependency: OffGameDependency) {
        super.init(dependency: dependency)
    }

    func build(withListener listener: OffGameListener) -> OffGameRouting {
        let component = OffGameComponent(dependency: dependency)
        let viewController = OffGameViewController(player1Name: component.player1Name,
                                                   player2Name: component.player2Name)
        let interactor = OffGameInteractor(presenter: viewController)
        interactor.listener = listener
        return OffGameRouter(interactor: interactor, viewController: viewController)
    }
}
```

And modify the `OffGameViewController` to store the player names received during initialization.

```swift
...

private let player1Name: String
private let player2Name: String

init(player1Name: String, player2Name: String) {
    self.player1Name = player1Name
    self.player2Name = player2Name
    super.init(nibName: nil, bundle: nil)
}

...
```

Finally, we will have to update the UI of the `OffGame` RIB's view controller to display the player names on screen. To save time, you may use the provided code [here](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial3-rib-di-and-communication/source/source1.swift?raw=true). 

Now you should be able to launch the app and see the player names at start screen after logging in.

## Track scores using a ReactiveX stream

At the moment, the application doesn't track the score after the games. When a game is over, the user simply gets redirected to the starting screen. We will improve the application to update the scores and display them at the starting screen. To do so, we will create and observe a reactive stream. 

Reactive programming techniques are used widely in RIBs architecture. One of the most common usages for them is to facilitate the communication between the RIBs. When a child RIB needs to receive dynamic data from its parent, it is a common practice to wrap the data into an observable stream on the producer side and subscribe to this stream at the consumer side. This tutorial assumes that the reader is familiar with the main concepts of reactive programming and observable streams. If you want to learn more about them, refer to [ReactiveX documentation](http://reactivex.io/documentation/observable.html).

In our case, the game score should be updated by the `TicTacToe` RIB as this RIB controls the state of the current game. This score should be read by the `OffGame` RIB as it will be displayed at the screen owned by this RIB. `TicTacToe` and `OffGame` don't know about each other and can't exchange data directly. However, both of them have the same parent - the `LoggedIn` RIB. We will have to implement the score stream in this RIB to give access to the stream to both its children.

Create a new Swift file named `ScoreStream` in `LoggedIn` group and add it to `TicTacToe` target. To save time, use the provided implementation of the stream available [here](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial3-rib-di-and-communication/source/source2.swift?raw=true).  

Notice that we have declared two versions of the score stream protocol, a read-only version named `ScoreStream`, and a mutable version named `MutableScoreStream`. We’ll explain the differences between the two below.

Create a shared `ScoreStream` instance in `LoggedInComponent`.  
```swift
var mutableScoreStream: MutableScoreStream {
    return shared { ScoreStreamImpl() }
}
```

A shared instance means a singleton created for the given scope (in our case, the scope includes the `LoggedIn` RIB and all its children). The streams are typically scoped singletons, as with most stateful objects. The majority of other dependencies should however be stateless, and therefore not shared.  

Notice that `mutableScoreStream` property was created not with `fileprivate`, but with (implicit) `internal` access modifier. We need to expose this property outside the file because it has to be accessible by the `LoggedIn`'s children. When this requirement doesn't hold, it's preferable to encapsulate the streams within the declaring file. 

Furthermore, only those dependencies that are directly used in the RIB should be placed in the base implementation of the component, with the exception being stored properties that are injected from dynamic dependencies, such as the player names. In this case, because `LoggedIn` RIB will directly use the `mutableScoreStream` in the `LoggedInInteractor` class, it is appropriate for us to place the stream in the base implementation. Otherwise, we would have placed the dependency in the extension, e.g. `LoggedInComponent+OffGame`. 

Now, let's pass the `mutableScoreStream` into the `LoggedInInteractor` so that it could update the scores later. We’ll also need to update the `LoggedInBuilder` to make the project compile.
 
```swift
...

private let mutableScoreStream: MutableScoreStream

init(mutableScoreStream: MutableScoreStream) {
    self.mutableScoreStream = mutableScoreStream
}

...
```

```swift
func build(withListener listener: LoggedInListener, player1Name: String, player2Name: String) -> LoggedInRouting {
    let component = LoggedInComponent(dependency: dependency,
                                      player1Name: player1Name,
                                      player2Name: player2Name)
    let interactor = LoggedInInteractor(mutableScoreStream: component.mutableScoreStream)

...
```
 
## Passing a read-only `ScoreStream` down to `OffGame` scope for displaying

Now we want to pass a read-only version of the `ScoreStream` down to the `OffGame` RIB so that it could display (but not update) the player scores after the game is over. Declare the read-only `ScoreStream` as a dependency of the `OffGame` RIB's scope in `OffGameDependency` protocol. 
 
```swift
protocol OffGameDependency: Dependency {
    var player1Name: String { get }
    var player2Name: String { get }
    var scoreStream: ScoreStream { get }
}
```

Then we'll provide the dependency to the current scope in `OffGameComponent`:
 
```swift
fileprivate var scoreStream: ScoreStream {
    return dependency.scoreStream
}
```

The `OffGame`'s builder should be modified to inject the stream into the `OffGameInteractor` for later use. 

```swift
func build(withListener listener: OffGameListener) -> OffGameRouting {
    let component = OffGameComponent(dependency: dependency)
    let viewController = OffGameViewController(player1Name: component.player1Name,
                                               player2Name: component.player2Name)
    let interactor = OffGameInteractor(presenter: viewController,
                                       scoreStream: component.scoreStream)
```

Finally, we should update the `OffGameInteractor` constructor to receive the score stream and store it in a private constant:

```swift
...

private let scoreStream: ScoreStream

init(presenter: OffGamePresentable,
     scoreStream: ScoreStream) {
    self.scoreStream = scoreStream
    super.init(presenter: presenter)
    presenter.listener = self
}

...
```

Notice that when we define the score stream variable in the `OffGame`'s component, we do so with `fileprivate` access modifier, in contrast to the definition in `LoggedIn` RIB. This is because we do not intend to expose this dependency to the `OffGame`’s children, which it doesn’t have anyways at the moment.

Because the read-only score stream is only needed by the `OffGame` scope and will not be used by the `LoggedIn` RIB, we will place this dependency in the `LoggedInComponent+OffGame` extension. A stub implementation of this file is already provided.

You can either work with the provided extension stub or delete the file and re-create it using the Component Extension Xcode template. Please read the TODO comments in the file to get a better understanding on how to work with component extensions.

Add the score stream as a dependency to the component extension.

```swift
extension LoggedInComponent: OffGameDependency {
    var scoreStream: ScoreStream {
        return mutableScoreStream
    }
}
```

Because our `MutableScoreStream` protocol extends the read-only version, we can expose it to the children expecting a read-only stream implementation. 

## Display the scores by subscribing to the score stream

Now, the `OffGame` RIB needs to subscribe to the score stream. After being notified about a new `Score` value emitted by the stream, `OffGamePresentable` will forward this value to the view controller for displaying it on the screen. By using a reactive subscription, we get rid of the stored state and automatically update the UI reflecting the changes in data.

Let’s update the `OffGamePresentable` protocol so we can set the score value. Remember, this is the protocol we use to communicate from the interactor to its view.  

```swift
protocol OffGamePresentable: Presentable {
    weak var listener: OffGamePresentableListener? { get set }
    func set(score: Score)
}
```

We create a subscription in the `OffGameInteractor` class and invoke the `OffGamePresentable` to set the new score when the stream emits a value.
  
```swift
private func updateScore() {
    scoreStream.score
        .subscribe(
            onNext: { (score: Score) in
                self.presenter.set(score: score)
            }
        )
        .disposeOnDeactivate(interactor: self)
}
```

Here we use the `disposeOnDeactivate` extension to handle our Rx subscription’s lifecycle. As the name suggests, the subscription is automatically disposed when the given interactor, in this case, the `OffGameInteractor`, is deactivated. We should almost always create Rx subscriptions in our interactor or worker classes to take advantage of these Rx lifecycle management utilities.

We then invoke the `updateScore` method in `OffGameInteractor`’s `didBecomeActive` lifecycle method. This allows us to create a new subscription whenever the `OffGameInteractor` is activated, which ties nicely with the use of `disposeOnDeactivate`. 
 
```swift
override func didBecomeActive() {
    super.didBecomeActive()

    updateScore()
}
```

Finally, we have to implement the UI to display the scores. To save time, you can use the `OffGameViewController` implementation provided [here](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial3-rib-di-and-communication/source/source3.swift?raw=true).

If you build and start the app now, you will see that it shows the score at the starting screen. We have added a reactive subscription that notifies the `OffGame` RIB each time the score changes and updated the view controller to respond to the change. However, we don't yet have the code to actually update the score after the game is over. Let's implement it.

## Updating the score stream when a game is over

After a game is over, the `TicTacToe` RIB invokes its listener to call up to `LoggedInInteractor`. This is where we should update our score stream.

Update `TicTacToe`’s listener to share the information about the game winner. 
 
```swift
protocol TicTacToeListener: class {
    func gameDidEnd(withWinner winner: PlayerType?)
}
```

Now we'll update the `TicTacToeInteractor` implementation to pass the winner to the listener we have just updated.  

There are different ways to do this. We can either store the winner in a local variable in `TicTacToeInteractor`, or we can let the `TicTacToeViewController` pass the winner back to the interactor in the `closeGame` method after the user closes the alert. Technically speaking, both ways are correct and appropriate. Let’s explore the advantages and drawbacks of both solutions.  

With the local variable stored in `TicTacToeInteractor`, the advantage is that we encapsulate all the necessary data within the interactor. The downside is that we have to maintain local, mutable state. This is somewhat mitigated by the fact that our RIBs are well scoped. The local states of each RIB are well encapsulated and limited. When we create a new `TicTacToe` RIB after launching a new game, the previous one is deallocated with all its local variables.  

If we take an approach with passing the data back from the view controller, we will avoid storing the local mutable state in the interactor, but will have to rely on the view controller in dealing with the business logic.  

To get the best of both worlds, we can use Swift closures. When the interactor informs the view controller that the game is over, it can provide a completion handler that will be invoked after the view controller updates its state. This will encapsulate the winner within the `TicTacToeInteractor` and won't lead to storing extra state. Also, this will render `closeGame` method in the view controller's listener unnecessary.

Here's how we can use a completion handler to update the score.

In `TicTacToePresentableListener`, remove the declaration of `closeGame` method.
 
```swift
protocol TicTacToePresentableListener: class {
    func placeCurrentPlayerMark(atRow row: Int, col: Int)
}
```

In `TicTacToeViewController`, modify the `announce` method to receive the completion handler as an argument and invoke it after the user dismisses an alert. The `announce` method is called by the interactor after the interactor decides that one of the players has won the game. 

```swift
func announce(winner: PlayerType?, withCompletionHandler handler: @escaping () -> ()) {
    let winnerString: String = {
        if let winner = winner {
            switch winner {
            case .player1:
                return "Red won!"
            case .player2:
                return "Blue won!"
            }
        } else {
            return "It's a draw!"
        }
    }()
    let alert = UIAlertController(title: winnerString, message: nil, preferredStyle: .alert)
    let closeAction = UIAlertAction(title: "Close Game", style: UIAlertActionStyle.default) { _ in
        handler()
    }
    alert.addAction(closeAction)
    present(alert, animated: true, completion: nil)
}
```

In `TicTacToePresentable` protocol, update the declaration of `announce` method to include the completion handler argument.

```swift
protocol TicTacToePresentable: Presentable {
    ...

    func announce(winner: PlayerType?, withCompletionHandler handler: @escaping () -> ())
}
```

In `TicTacToeInteractor`, update the call to `announce` method of the presenter with a completion handler.

```swift
func placeCurrentPlayerMark(atRow row: Int, col: Int) {
    guard board[row][col] == nil else {
        return
    }

    let currentPlayer = getAndFlipCurrentPlayer()
    board[row][col] = currentPlayer
    presenter.setCell(atRow: row, col: col, withPlayerType: currentPlayer)

    if let winner = checkWinner() {
        presenter.announce(winner: winner) {
            self.listener?.gameDidEnd(withWinner: winner)
        }
    }
}
```

Finally, in `LoggedInInteractor` update the implementation of `gameDidEnd` method to update the score stream when there's a new winner. 
  
```swift
func gameDidEnd(withWinner winner: PlayerType?) {
    if let winner = winner {
        mutableScoreStream.updateScore(withWinner: winner)
    }
    router?.routeToOffGame()
}
```

Now the score stream is fully functional. If you build and launch an app, you will see that each time a player wins the game, the score at the start screen is correctly updated.

## Bonus exercises

There are two more things we recommend you to do to improve the app and better understand the communication between the RIBs.

The first thing would be improving an alert shown after the game is over. At the moment, instead of showing a name of the winning player we use hardcoded names "Red" and "Blue". You could pass down the player names from the `LoggedIn` scope to the `TicTacToe` scope and display them in the alert instead.

Another nice improvement would be dealing with the draws. At the moment, the game gets stuck when it ends in a draw with all game fields marked. You can update the game logic, the user interface, and score calculation to handle this case.

## Tutorial complete

Congratulations! You have completed tutorial 3. The completed source code for this tutorial can be found [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial3-completed).

Now onwards to [tutorial 4](iOS-Tutorial-4).