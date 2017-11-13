# Deep Linking Workflows

> Note: If you haven't completed [tutorial 3](iOS-Tutorial-3) yet, we encourage you to do so before jumping into this tutorial.

Welcome to the RIBs tutorials, which have been designed to give you a hands-on walkthrough through the core concepts of RIBs. As part of the tutorials, you'll be building a simple tic-tac-toe game using the RIBs architecture and associated tooling.

For tutorial 4, we'll start source code that can be found [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial4). Follow the [README](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial4/README.md) to install and open the project before reading any further.

## Goals

The goals of this tutorial are to learn the following:
* Understand basics of RIB workflows
* Learn how to create actionable item interfaces, implement their methods, and create workflows to launch specifics flows via deeplinks.

In the end, you should be able to open the app from Safari, by opening to the URL `ribs-training://launchGame?gameId=ticTacToe`, which should start a game with an identifier of `gameId`.

## Implementing the URL handler

In order for the application to handle a custom URL scheme, we should add the following lines in the `Info.plist`:

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.uber.TickTackToe</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>ribs-training</string>
        </array>
    </dict>
</array>
```

We'll add a new protocol called `UrlHandler` in the `AppDelegate.swift` file:

```swift
protocol UrlHandler: class {
    func handle(_ url: URL)
}
```

And we'll add an instance variable in the `AppDelegate` class:

```swift
private var urlHandler: UrlHandler?
```

We'll make sure that the application delegate passes an url to the `urlHandler`:

```swift
public func application(_ application: UIApplication, open url: URL, sourceApplication: String?, annotation: Any) -> Bool {
    urlHandler?.handle(url)
    return true
}
```

The `RootInteractor` is going to be our `UrlHandler`. We need to make `RootInteractor` conform to `RootActionableItem` and `UrlHandler` protocols:

```swift
final class RootInteractor: PresentableInteractor<RootPresentable>, 
    RootInteractable, 
    RootPresentableListener, 
    RootActionableItem, 
    UrlHandler
```

To be able to start handling an URL, we'll pass it to the `LaunchGameWorkflow`, and subscribe to the workflow. We can do this as `LaunchGameWorkflow`'s `ActionableItem` is `RootActionableItem`, and `RootInteractor` conforms to this protocol.

```swift
// MARK: - UrlHandler

func handle(_ url: URL) {
    let launchGameWorkflow = LaunchGameWorkflow(url: url)
    launchGameWorkflow
        .subscribe(self)
        .disposeOnDeactivate(interactor: self)
}
```

Let's change the `RootBuilder`, so that it returns `UrlHandler` together with `RootRouting` instance:

```swift
protocol RootBuildable: Buildable {
    func build() -> (launchRouter: LaunchRouting, urlHandler: UrlHandler)
}
```

```swift
func build() -> (launchRouter: LaunchRouting, urlHandler: UrlHandler) {
    let viewController = RootViewController()
    let component = RootComponent(dependency: dependency,
                                  rootViewController: viewController)
    let interactor = RootInteractor(presenter: viewController)

    let loggedOutBuilder = LoggedOutBuilder(dependency: component)
    let loggedInBuilder = LoggedInBuilder(dependency: component)
    let router = RootRouter(interactor: interactor,
                            viewController: viewController,
                            loggedOutBuilder: loggedOutBuilder,
                            loggedInBuilder: loggedInBuilder)

    return (router, interactor)
}
```

And set the `urlHandler` in the `AppDelegate`:

```swift
public func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
    let window = UIWindow(frame: UIScreen.main.bounds)
    self.window = window

    let result = RootBuilder(dependency: AppComponent()).build()
    launchRouter = result.launchRouter
    urlHandler = result.urlHandler
    launchRouter?.launchFromWindow(window)

    return true
}
```

## Implementing the workflow

We'll implement the workflow in the `Promo` project group. We're assuming the promotion feature is the one that provides this workflow.

Let's declare the `Root` scope actionable item we need in order to launch a game. For the `Root` scope, we need to wait until the players log in. In order to do that, we simply modify the `RootActionableItem` protocol:

```swift
public protocol RootActionableItem: class {
    func waitForLogin() -> Observable<(LoggedInActionableItem, ())>
}
```

The return type is `Observable<(NextActionableItemType, NextValueType)>`, which allows us to chain another step for the next actionable item with a new value. In the case of our application, once we are logged in, we are routed to the LoggedIn RIB. Which means that `NextActionableItemType` is `LoggedInActionableItem` which we'll define in the next step. We don't need any values to process our workflow, so our `NextValueType` is just `Void`.

Once we get to the LoggedIn RIB, we'll need to launch a game with the identifier provided by the URL. Let's define the `LoggedInActionableItem` in a new file. 

```swift
import RxSwift

public protocol LoggedInActionableItem: class {
    func launchGame(with id: String?) -> Observable<(LoggedInActionableItem, ())>
}
```

Next, we'll create our workflow in the `Promo` project group. We’ll create a new file called `LaunchGameWorkflow.swift`, and remove the `Stub.swift` file, since that is no longer needed. All workflows inherit from the `Workflow` base class. Because we start at the `Root` scope, the initial actionable item type should be the `RootActionableItem`. 

```swift
import RIBs
import RxSwift

public class LaunchGameWorkflow: Workflow<RootActionableItem> {
    public init(url: URL) {
        super.init()

        let gameId = parseGameId(from: url)

        self
            .onStep { (rootItem: RootActionableItem) -> Observable<(LoggedInActionableItem, ())> in
                rootItem.waitForLogin()
            }
            .onStep { (loggedInItem: LoggedInActionableItem, _) -> Observable<(LoggedInActionableItem, ())> in
                loggedInItem.launchGame(with: gameId)
            }
            .commit()
    }

    private func parseGameId(from url: URL) -> String? {
        let components = URLComponents(string: url.absoluteString)
        let items = components?.queryItems ?? []
        for item in items {
            if item.name == "gameId" {
                return item.value
            }
        }

        return nil
    }
}
```

## Integrating the waitForLogin step at the Root scope

The `RootInteractor` already conforms to the `RootActionableItem` protocol. Now, we just need to make the `RootInteractor` compile by providing the required implementations.

First, we'll implement the protocol's `waitForLogin` method in the `RootInteractor`. Each scope's interactor is always the actionable item for that scope.

Because the "wait for login" action is asynchronous, it's best we use Rx for the implementation. We first declare a `ReplaySubject` constant that holds the `LoggedInActionableItem` in the `RootInteractor`:

```swift
private let loggedInActionableItemSubject = ReplaySubject<LoggedInActionableItem>.create(bufferSize: 1)
```

We use a `ReplaySubject` because once we are logged in, we don't really want to wait for the "next" login, but simply replay the "existing" login. 

Next, we return this subject as an `Observable` in our `waitForLogin` method. As soon as we have a `LoggedInActionableItem` emitted from the `Observable`, our workflow's step of waiting for the user to login is completed. Therefore, we can move onto the next step with the LoggedInActionableItem as our actionable item type. 

```swift
// MARK: - RootActionableItem

func waitForLogin() -> Observable<(LoggedInActionableItem, ())> {
    return loggedInActionableItemSubject
        .map { (loggedInItem: LoggedInActionableItem) -> (LoggedInActionableItem, ()) in
            (loggedInItem, ())
        }
}
```

Finally, when we route to logged in, we'll emit the `LoggedInActionableItem` onto the `ReplaySubject`. We do this by modifying the `didLogin` method in the `RootInteractor`. 

```swift
// MARK: - LoggedOutListener

func didLogin(withPlayer1Name player1Name: String, player2Name: String) {
    let loggedInActionableItem = router?.routeToLoggedIn(withPlayer1Name: player1Name, player2Name: player2Name)
    if let loggedInActionableItem = loggedInActionableItem {
        loggedInActionableItemSubject.onNext(loggedInActionableItem)
    }
}
```

As the `didLogin` method's new implementation suggests, we need to update the `RootRouting` protocol's `routeToLoggedIn` method to return the `LoggedInActionableItem` instance, which will be the `LoggedInInteractor`. 

```swift
protocol RootRouting: ViewableRouting {
    func routeToLoggedIn(withPlayer1Name player1Name: String, player2Name: String) -> LoggedInActionableItem
}
```

Now, let's update the `RootRouter` implementation because the `RootRouting` protocol has been modified. We need to return the `LoggedInActionableItem`, which is the `LoggedInInteractor`. 

```swift
func routeToLoggedIn(withPlayer1Name player1Name: String, player2Name: String) -> LoggedInActionableItem {
    // Detach logged out.
    if let loggedOut = self.loggedOut {
        detachChild(loggedOut)
        viewController.replaceModal(viewController: nil)
        self.loggedOut = nil
    }

    let loggedIn = loggedInBuilder.build(withListener: interactor, player1Name: player1Name, player2Name: player2Name)
    attachChild(loggedIn.router)
    return loggedIn.actionableItem
}
```

We'll update the `LoggedInBuildable` protocol accordingly, so that it returns a tuple of the `LoggedInRouting` and `LoggedInActionableItem` instances.

```swift
protocol LoggedInBuildable: Buildable {
    func build(withListener listener: LoggedInListener, player1Name: String, player2Name: String) -> (router: LoggedInRouting, actionableItem: LoggedInActionableItem)
}
```
 
And because the `LoggedInBuildable` protocol has changed, we need to update the `LoggedInBuilder` implementation to conform to the changes. We need to return the interactor as well. As mentioned before, the interactor of a scope is the actionable item for that scope. 

```swift
func build(withListener listener: LoggedInListener, player1Name: String, player2Name: String) -> (router: LoggedInRouting, actionableItem: LoggedInActionableItem) {
    let component = LoggedInComponent(dependency: dependency,
                                      player1Name: player1Name,
                                      player2Name: player2Name)
    let interactor = LoggedInInteractor(games: component.games)
    interactor.listener = listener

    let offGameBuilder = OffGameBuilder(dependency: component)
    let router = LoggedInRouter(interactor: interactor,
                          viewController: component.loggedInViewController,
                          offGameBuilder: offGameBuilder)
    return (router, interactor)
}
```

## Integrating the launchGame step in the LoggedIn scope

Let's update the `LoggedInInteractor` to conform to the `LoggedInActionableItem` protocol that we declared earlier. Recall that each scope's interactor should always conform to the actionable item protocol for that scope.  

```swift
final class LoggedInInteractor: Interactor, LoggedInInteractable, LoggedInActionableItem
```

We then provide an implementation for the `LoggedInInteractor` to conform to the `LoggedInActionableItem` protocol. 

```swift
// MARK: - LoggedInActionableItem

func launchGame(with id: String?) -> Observable<(LoggedInActionableItem, ())> {
    let game: Game? = games.first { game in
        return game.id.lowercased() == id?.lowercased() 
    }

    if let game = game {
        router?.routeToGame(with: game.builder)
    }

    return Observable.just((self, ()))
}
```

## Run the workflow

In order to test our implementation, we need to run the app once, then open Safari on the same device.
Type in `ribs-training://launchGame?gameId=ticTacToe` as the address. We can also try changing the `gameId` parameter to `randomWin` to launch the `RandomWin` game.

Notice that the workflow initially waits for players to login. Once players login, the workflow continues and launches the tic-tac-toe game immediately.

## Tutorial complete

Congratulations! You completed tutorial 4. The completed source for this tutorial can be found [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial4-completed).