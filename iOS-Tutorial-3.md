# RIBs Dependency Injection and Communication

> Note: If you haven't completed [tutorial 2](iOS-Tutorial-2) yet, we encourage you to do so before jumping into this tutorial.

Welcome to the RIBs tutorials, which have ben designed to give you a hands-on walkthrough through the core concepts of RIBs. As part of the tutorials, you'll be building a simple TicTacToe game using the RIBs architecture and associated tooling.

For tutorial 3, we'll use the source code [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial3) as a starting point. Follow the [README](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial3/README.md) to install and open the project before reading any further.

## Goals

The main goals of this exercise to understand the following concepts:
- Passing a dynamic dependency via the Builder’s build method.
- Passing static dependencies using the Dependency Injection tree.
  - Extension based dependency conformance in Swift.
- Rx stream lifecycle management using the RIB lifecycle.

## Dynamic dependencies

We'd want to pass our player names as dynamic dependencies from the Root RIB to the LoggedIn RIB via the `LoggedInBuilder`’s build method.

For this, we'll update the `LoggedInBuildable` protocol to include two player names as dynamic dependencies, in addition to the existing `listener` dynamic dependency:

```swift
protocol LoggedInBuildable: Buildable {
    func build(withListener listener: LoggedInListener, player1Name: String, player2Name: String) -> LoggedInRouting
}
 ```

Then we'll update the `LoggedInBuilder`'s  `build` method:

```swift
func build(withListener listener: LoggedInListener, player1Name: String, player2Name: String) -> LoggedInRouting {
    let component = LoggedInComponent(dependency: dependency,
                                      player1Name: player1Name,
                                      player2Name: player2Name)
```

Finally, we'll update the `LoggedInComponent` initializer to put the player names onto DI tree. We'll store them as constants of the `LoggedInComponent`:

```swift
let player1Name: String
let player2Name: String

init(dependency: LoggedInDependency, player1Name: String, player2Name: String) {
    self.player1Name = player1Name
    self.player2Name = player2Name
    super.init(dependency: dependency)
}
```

This effectively transforms the player names from dynamic dependencies taken in from `LoggedIn`’s parent to static dependencies for any of LoggedIn’s children. 

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

### Dynamic dependencies vs static dependencies

Let’s examine why we use dynamic dependencies here, instead of just passing down the player names via the normal DI tree. If we give the static approach a try, we'd find that we wouldn't be able to make the player names invariants. They would have to be optionals. This is because the player names aren't actually available at the time Root scope is created. 

The implication of using static optional user names would be that is our application could become unstable. When we write our code that uses the player names, what would we do if the player names for some reason are `nil`? Crash the app? There probably isn't any reasonable handling for this at all. Properly scoped dependencies allow us to make invariant assumptions, thus eliminating any unreasonable or unstable code.

## Passing down player names from the LoggedIn scope to the OffGame scope using the DI tree

Next we'll want to use the player names in our app and display them in the OffGame RIB. For this, we'll add player names dependencies in the `OffGameDependency` protocol in the OffGameBuilder.swift file:

```swift
protocol OffGameDependency: Dependency {
    var player1Name: String { get }
    var player2Name: String { get }
}
```
This declares that the OffGame RIB is dependent on both player names and cannot be instantiated without receiving them from the DI graph.

Next, we'll provide these dependencies to the OffGame’s own scope using its OffGameComponent.
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

Notice these properties are marked as fileprivate. This means they are only accessible within the OffGameBuilder.swift file, and therefore are not exposed to child scopes. We didn’t use the fileprivate access control in `LoggedInComponent`, because we wanted to provide these values to the OffGame child scope.

Since we’ve already added the player names to the `LoggedInComponent` in the previous steps, there’s nothing further we need to do to make OffGame’s parent scope - the LoggedIn scope - satisfy these new dependencies we just added.

Next we'll pass these dependencies into the `OffGameViewController` via constructor injection to and display them. We could also pass these into the `OffGameInteractor` first and let the interactor invoke functions on its `OffGamePresentable` to display them, but since this data does not require any business logic or formatting, we can directly pass these to the view controller to display. We'll use `player1Name` and `player2Name` constants in the `OffGameViewController` to store the values passed in through the constructor: 
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

```swift
private let player1Name: String
private let player2Name: String

init(player1Name: String, player2Name: String) {
    self.player1Name = player1Name
    self.player2Name = player2Name
    super.init(nibName: nil, bundle: nil)
}
```

Finally we'll build some UI to display them using a `UILabel`. To save time, you may use the provided code [here](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial3-rib-di-and-communication/source/source1.swift?raw=true).

## Track scores using a Rx stream

In order to determine which is the appropriate scope for our score stream, we should consider where this stream would be used. We need it in the OffGame scope to display the scores. We also need to update the score when a game is won, when TicTacToe invokes its TicTacToeListener up to LoggedIn scope. Therefore, the lowest scope that encompasses all the access needs is LoggedIn.

Create a score Rx stream class/file in the LoggedIn folder.  

To save time, here’s the implementation of the [ScoreStream](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial3-rib-di-and-communication/source/source2.swift?raw=true) class.  

Notice there are two versions of the score stream protocol, a read-only version named ScoreStream, and mutable version named MutableScoreStream. We’ll go over how they are used below.

Create a shared ScoreStream instance at the LoggedIn scope in LoggedInComponent.  
```swift
var mutableScoreStream: MutableScoreStream {
    return shared { ScoreStreamImpl() }
}
```
A shared instance, means singleton per scope. This allows us to keep a single score stream for LoggedIn RIB and all of its children. Streams are typically scoped singletons, as with most stateful objects. Most other dependencies, however, should be stateless, therefore, not shared.  

Notice how the property is not fileprivate, but rather internal. This is because we know we would need to expose it to children scopes. Otherwise all properties in the base FooComponent class should be fileprivate. Further more, only dependencies that are directly used in the RIB should be placed in the base implementation, with the exception being stored properties that are injected from dynamic dependencies, such as the player names. In this case, because LoggedIn RIB directly uses the mutableScoreStream in LoggedInInteractor, it is appropriate for us to place the stream in the base implementation. Otherwise we would have placed the dependency in the extension LoggedInComponent+OffGame.  

Pass mutableScoreStream into LoggedInInteractor so we can update it later. We’ll also need to update the LoggedInBuilder to make things compile.  
```swift
func build(withListener listener: LoggedInListener, player1Name: String, player2Name: String) -> LoggedInRouting {
    let component = LoggedInComponent(dependency: dependency,
                                      player1Name: player1Name,
                                      player2Name: player2Name)
    let interactor = LoggedInInteractor(mutableScoreStream: component.mutableScoreStream)
```
```swift
private let mutableScoreStream: MutableScoreStream

init(mutableScoreStream: MutableScoreStream) {
    self.mutableScoreStream = mutableScoreStream
}
```
 
## Pass read-only ScoreStream down to OffGame scope for display

Declare the read-only ScoreStream as a dependency of the OffGame scope, in the OffGameDependency protocol.  
```swift
protocol OffGameDependency: Dependency {
    var player1Name: String { get }
    var player2Name: String { get }
    var scoreStream: ScoreStream { get }
}
```

Provide the stream dependency to the current scope in OffGameComponent, OffGame and inject it into the interactor for later use.  
```swift
fileprivate var scoreStream: ScoreStream {
    return dependency.scoreStream
}
```
```swift
func build(withListener listener: OffGameListener) -> OffGameRouting {
    let component = OffGameComponent(dependency: dependency)
    let viewController = OffGameViewController(player1Name: component.player1Name,
                                               player2Name: component.player2Name)
    let interactor = OffGameInteractor(presenter: viewController,
                                       scoreStream: component.scoreStream)
```
```swift
private let scoreStream: ScoreStream

init(presenter: OffGamePresentable,
     scoreStream: ScoreStream) {
    self.scoreStream = scoreStream
    super.init(presenter: presenter)
    presenter.listener = self
}
```

Notice this stream is provided as fileprivate, in contrast to the LoggedIn version, where it was internal. This is because we do not intend to expose this down to OffGame’s children, which at the moment, it doesn’t have any anyways.

Because the read-only score stream is only needed by the OffGame scope, and not the LoggedIn scope RIB, we place this dependency in the LoggedInComponent+OffGame extension. The starting point already has a stub implementation. Feel free to create it from scratch using the Component Extension Xcode template. Please spend some time read through the TODO documentation in the file. It should provide insights into what these are for.  
```swift
extension LoggedInComponent: OffGameDependency {
    var scoreStream: ScoreStream {
        return mutableScoreStream
    }
}
```
Because our MutableScoreStream protocol extends from the read-only version, we can just directly expose that. As mentioned before, this is why we marked the mutableScoreStream dependency in LoggedInComponent class as internal, instead of fileprivate. We need to expose it down to children scopes.

## Display scores by subscribing to the score stream

Whenever the score stream emits a new Score value, we should invoke OffGamePresentable, our OffGameViewController, to show the new score. This type of reactive programming is extremely powerful in the sense that there’s no stored states to maintain, and our UI just automatically updates as the underlying data changes.

Let’s update the OffGamePresentable protocol so we can set the score value. Remember, this is the protocol we use to communicate from an interactor to its view.  
```swift
protocol OffGamePresentable: Presentable {
    weak var listener: OffGamePresentableListener? { get set }
    func set(score: Score)
}
```

We create a subscription in OffGameInteractor and invoke the OffGamePresentable to set to the new score when the stream emits a value.  
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

Here we use the disposeOnDeactivate extension to handle our Rx subscription’s lifecycle. As the name suggests, the subscription is automatically disposed when the given interactor, in this case self the OffGameInteractor, is deactivated. We should almost always create Rx subscriptions in our interactor or worker classes to take advantage of these Rx lifecycle management utilities.

We then invoke the updateScore method in OffGameInteractor’s didBecomeActive lifecycle method. This allows us to create a new subscription whenever the OffGameInteractor is activated, which ties nicely with the use of disposeOnDeactivate.  
```swift
override func didBecomeActive() {
    super.didBecomeActive()

    updateScore()
}
```

Finally we can implement the UI to display the scores. To save time, we can use the implementation provided [here](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial3-rib-di-and-communication/source/source3.swift?raw=true).

## Update score stream when a game is won

When a game is won, TicTacToe invokes its listener to call up to LoggedInInteractor. This is where we should update our score stream.

Update TicTacToe’s listener to provide context on which player had just won.  
```swift
protocol TicTacToeListener: class {
    func gameDidEnd(withWinner winner: PlayerType?)
}
```

Update the TicTacToeInteractor implementation to pass the winner to the listener we just updated.  

There are a couple of ways to do this. We can either store the winner as a local state in the TicTacToeInteractor, or we can let the TicTacToeViewController pass the winner back to the interactor when the view controller invokes closeGame method when the user closes the alert. Technically speaking, both ways are correct and appropriate. Let’s explore the advantages and drawbacks of both solutions.  

With the local state stored in TicTacToeInteractor approach, the advantage is that we encapsulate all the necessary data within the interactor. The downside is that we have to maintain local, mutable state. Having said that, this is somewhat mitigated by the fact that RIBs are well scoped. The local states are well encapsulated and limited. When we create a new TicTacToe RIB when we launch a new game, the previous one is deallocated with all local states erased.  

With the passing back from view controller approach, we can avoid the local mutable state, but we end up relying on the view controller to pass back business data.  

To get the best of both worlds, we can take advantage of Swift closures. When we invoke the TicTacToePresentable, the view controller, to announce the winner, we can pass an opaque closure to be invoked when the announcement is completed. This encapsulates the winner within the TicTacToeInteractor, without storing any local states. This also removes the need to have the closeGame method in our TicTacToeViewControllerListener protocol.  
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
```swift
func announce(winner: PlayerType?, withCompletionHandler handler: @escaping () -> ()) {
    let winnerString: String = {
        if let winner = winner {
            switch winner {
            case .player1:
                return "\(player1Name) Won!"
            case .player2:
                return "\(player2Name) Won!"
            }
        } else {
            return "It's a Tie"
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

We’ve already injected in the MutableScoreStream before, so we can just directly use it in LoggedInInteractor.  
```swift
func gameDidEnd(withWinner winner: PlayerType?) {
    if let winner = winner {
        mutableScoreStream.updateScore(withWinner: winner)
    }
    router?.routeToOffGame()
}
```

## Bonus

So far when the game ends, we have a fixed alert that shows either “Player 1” or “Player 2” has won. Let’s pass down the player names from LoggedIn to TicTacToe scope and display the actual names in the alert instead.

Our current TicTacToe will get stuck if the game is a tie. Let’s update the game logic as well as the necessary display and score keeping logic to handle the tie case.

## Tutorial complete

Congratulations! You completed tutorial 3. The completed source for this tutorial can be found [here].(https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial3-completed)

Now onwards to [tutorial 4](iOS-Tutorial-4).