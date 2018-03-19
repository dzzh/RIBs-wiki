# Deeplinking and Workflows

> Note: If you haven't completed [tutorial 3](iOS-Tutorial-3) yet, we encourage you to do so before jumping into this tutorial.

Welcome to the RIBs tutorials, which have been designed to give you a hands-on walkthrough through the core concepts of RIBs. As part of the tutorials, you'll be building a simple tic-tac-toe game using the RIBs architecture and associated tooling.

In tutorial 4 we'll start with the source code that can be found [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial4). Follow the [README](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial4/README.md) to install and open the project before reading any further.

## Goals

In tutorials 1-3 we have built a tic-tac-toe game consisting of five RIBs. In this tutorial, we will demonstrate how to add deeplinking support into our app and start a new game by opening a URL in Safari.

After completing this exercise, we expect you to understand the basics of RIB workflows and actionable items as well as to learn how to launch a workflow via a deeplink. You will be able to start the app by opening `ribs-training://launchGame?gameId=ticTacToe` URL from Safari. Opening this link will not only launch the app, but also start a new game bypassing the starting screen.

## Implementing a URL handler

[Custom URL schemes support](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6-SW10), or deeplinking, is an iOS mechanism allowing for inter-app communication via custom URLs. An app that registers itself as a handler of a particular URL scheme is launched after the user opens an URL with a matching scheme from another app. The opened app gets access to the contents of the received URL and is thus able to transition itself to the state described in the URL.

To register TicTacToe app as a handler of a custom URL scheme `ribs-training://`, we should add the following lines in the `Info.plist`.

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.uber.TicTacToe</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>ribs-training</string>
        </array>
    </dict>
</array>
```

To handle the custom URL sent to the app by the system, we need to make some adjustments to the app delegate.

We start with adding a new protocol called `UrlHandler` to `AppDelegate.swift`. This protocol will later be implemented by a concrete class that will contain app-specific URL handling logic.

```swift
protocol UrlHandler: class {
    func handle(_ url: URL)
}
```

Let's store a reference to the URL handler inside the app delegate, so we could ask it to handle a deeplink URL after we receive it. Add a new instance variable to the `AppDelegate` class.

```swift
private var urlHandler: UrlHandler?
```

Now, let's implement an `AppDelegate`'s method triggered when a deeplink is sent to the app. In this method, we forward the URL to the URL handler.

```swift
public func application(_ application: UIApplication, open url: URL, sourceApplication: String?, annotation: Any) -> Bool {
    urlHandler?.handle(url)
    return true
}
```

The custom URLs have to be handled by the `Root` RIB as it resides at the top of the RIBs hierarchy. Handling the URLs at the root allows us to configure the app the way we want after receiving a deeplink, because the `Root` RIB can build all its children and put any of them on screen (either directly or indirectly while going down the RIB tree). We will make the `RootInteractor` our URL handler. Let's start with requiring the root interactor to conform to `UrlHandler` and `RootActionableItem` protocols. `RootActionableItem` is just an empty protocol that will be explained in the next section.

```swift
final class RootInteractor: PresentableInteractor<RootPresentable>, 
    RootInteractable, 
    RootPresentableListener, 
    RootActionableItem, 
    UrlHandler
```

To handle an URL inside the app, we will use a RIBs mechanism named _a workflow_. We will explain the workflows in greater detail in the next section, for now let's just copy the code below to the `RootInteractor` class.

```swift
// MARK: - UrlHandler

func handle(_ url: URL) {
    let launchGameWorkflow = LaunchGameWorkflow(url: url)
    launchGameWorkflow
        .subscribe(self)
        .disposeOnDeactivate(interactor: self)
}
```

We already have a workflow stub in `Promo` group, you will have to replace it with a proper implementation later on.

Next, let's modify the `RootBuilder` so that it would return `UrlHandler` together with `RootRouting` instance.

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

With all these changes, we can now return to the app delegate and initialize the `urlHandler` property that was created earlier.

```swift
public func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
    let window = UIWindow(frame: UIScreen.main.bounds)
    self.window = window

    let result = RootBuilder(dependency: AppComponent()).build()
    launchRouter = result.launchRouter
    urlHandler = result.urlHandler
    launchRouter?.launch(from: window)

    return true
}
```

We have added deeplinking support to our app. After receiving a deeplink with `ribs-training://` scheme, the app will launch a workflow defined in `RootInteractor`. However, the workflow is just a stub, so nothing will change after the app opens.

## Workflows and actionable items

In RIBs terms, a workflow is a sequence of steps that compose a certain operation. This operation can be executed on the RIBs tree, meaning that we can go up and down the tree as the operation progresses. Usually, a workflow starts at the root of the tree and navigates down via a certain path until reaching the point where it could transition the app to the expected state. The base workflow is implemented as a generic class parameterized with an actionable item. App-specific workflows are expected to extend the base one.

The workflows are backed with reactive streams and expose the API similar to what can be found in ReactiveX's observable. To launch a workflow, it is necessary to subscribe to it and add the returned disposable to a dispose bag, exactly as it happens with a regular observable. After being launched, the workflow will start to asynchronously execute its steps one-by-one until no more steps will be left.

A workflow step can be defined as a pair of an actionable item and its associated value. The actionable item contains the logic that has to be executed during the step, the value serves as an argument used to pass the state between different steps to help them make the decisions as the workflow progresses.

It is a responsibility of a RIB's interactor to serve as an actionable item for a workflow step and encapsulate the logic necessary to execute the step and navigate the RIBs tree. As you remember, at one of the previous snippets we made `RootInteractor` conform to `RootActionableItem` protocol. This basically means that we want `RootInteractor` to serve as an actionable item for the `Root` RIB.

The `LaunchGameWorkflow` that we created earlier is parameterized with `RootActionableItem` type. This means that this workflow will use the code written in a class conforming to `RootActionableItem` protocol (in our case, this class is `RootInteractor`) to configure and execute its first step. The code invoked in `RootActionableItem` will eventually have to provide the actionable item and its associated value for the second step. For the first step, the associated value is assumed to be void.

## Implementing the workflow

As we dicsussed previously, after receiving a deeplink the app should be able to start a new game with a given identifier. However, if the players are not logged in, the app should first wait until they do log in and only after that redirect the players onto the game field.

This can be modelled with a workflow consisting out of two steps. During the first step we will check if the players are logged in and wait until they do if necessary. This will be done at the `Root` RIB level. After we decide that the players are ready, the first step will transfer the control to the second step (by emitting a value via an observable stream). 

The second step will be implemented by the `LoggedIn` RIB. This RIB is a direct child of the `Root` RIB, it knows how to start a new game. Its router interface declares `routeToGame` method that has to be triggered to navigate to the game field. The second step will trigger this method. This will conclude the work needed to be done by the workflow.

Let us declare an interface allowing us to wait until the players log in. Proceed to `RootActionableItem` protocol and add a signature of a new method.

```swift
public protocol RootActionableItem: class {
    func waitForLogin() -> Observable<(LoggedInActionableItem, ())>
}
```

As you see from the method's signature, it returns an observable that emits a tuple `(NextActionableItemType, NextValueType)`. This tuple allows us to build a workflow step that will be executed after the current step completes. Until the observable emits the first value, the workflow will be blocked. This reactive pattern allows the workflow to be asynchronous, it shouldn't necessarily complete all its steps immediately after the launch.

You can also notice that `NextActionableItemType` in our case is `LoggedInActionableItem`. This is a name of an actionable item that we need to add to the `LoggedIn` RIB for the second step. `NextValueType` is void as there's no need to forward any extra state from the `Root` RIB down the workflow chain.

Now, let us define `LoggedInActionableItem` protocol in a new Swift file inside `ActionableItems` group.

```swift
import RxSwift

public protocol LoggedInActionableItem: class {
    func launchGame(with id: String?) -> Observable<(LoggedInActionableItem, ())>
}
```

This protocol describes the second and last step of the workflow we are building. As it doesn't need to trigger another step upon completion, we will return `LoggedInActionableItem` itself as the next actionable item type. This is needed to comply with the workflow's type constraints. As in the previous step, there's no need to forward any additional data down the chain, so we use a void object as the step's value.

After having declared the steps that need to be executed by the workflow, we can finally build the workflow itself. Let's imagine that the deeplink for this workflow was created to support one of our promotion campaigns and we want to have the workflow implementation close to the other promotion-related code. Go to the `Promo` group, delete a stub workflow file from there and add a new Swift file named `LaunchGameWorkflow.swift`. Copy the code from the snippet below into the new file.

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

From this code snippet you can see that we configure the workflow directly inside its initializer. Each of the two workflow steps is configured with a Swift closure that accepts the actionable item and the value as parameters (or only the actionable item for the first step) and returns an observable emitting a tuple in form of `(NextActionableItemType, NextValueType)`. 

Under the hood, the workflow executes the closure and subscribes to the returned observable. During each step, the worflow waits until the observable emits the first value and then switches to the next step. 

## Integrating the `waitForLogin` step at the `Root` scope

The `RootInteractor` already conforms to the `RootActionableItem` protocol. Now, we just need to make the `RootInteractor` compile by providing the required implementations.

First, we'll implement already declared `waitForLogin` method in the `RootInteractor`. It's a responsibility of an interactor to serve as the actionable item for a RIB. 

Waiting for login is an asynchronous operation. To implement it, we will use a reactive subject. First, let's declare a `ReplaySubject` constant that holds the `LoggedInActionableItem` in the `RootInteractor`.

```swift
private let loggedInActionableItemSubject = ReplaySubject<LoggedInActionableItem>.create(bufferSize: 1)
```

Next, we need to return this subject as an `Observable` in our `waitForLogin` method. As soon as we have a `LoggedInActionableItem` emitted from the `Observable`, our workflow's step of waiting for the user to log in is completed. Therefore, we can move onto the next step with the `LoggedInActionableItem` as our actionable item type. 

```swift
// MARK: - RootActionableItem

func waitForLogin() -> Observable<(LoggedInActionableItem, ())> {
    return loggedInActionableItemSubject
        .map { (loggedInItem: LoggedInActionableItem) -> (LoggedInActionableItem, ()) in
            (loggedInItem, ())
        }
}
```

Finally, when we route from the `Root` RIB to the `LoggedIn` RIB, we'll emit the `LoggedInActionableItem` onto the `ReplaySubject`. We do this by modifying the `didLogin` method in the `RootInteractor`. 

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

With these changes, we have implemented the first step of the workflow. After the launch, the workflow will wait until the users log in and then will switch to the second step that has to be implemented in the `LoggedIn` RIB to start the game.

## Integrating the `launchGame` step in the `LoggedIn` scope

Let's update the `LoggedInInteractor` to conform to the `LoggedInActionableItem` protocol that we declared earlier. Recall that each scope's interactor should always conform to the actionable item protocol for that scope.  

```swift
final class LoggedInInteractor: Interactor, LoggedInInteractable, LoggedInActionableItem
```

You can use a provided implementation of the `launchGame` method required by the `LoggedInActionableItem` protocol. This method implements the logic for the second step of the workflow.

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

As you can see from the implementation of this method, we request the `LoggedIn`'s router to navigate to the game field and then return back an observable to conform to the expected type constraints. As this step is the last in the workflow, the returned actionable type won't really be used by the workflow.

## Run the workflow

After we implemented both workflow steps, we can test the application to make sure the deeplinking mechanism and the workflow work as expected.

Build and run the application, then close it and open Safari browser on your phone. Type in `ribs-training://launchGame?gameId=ticTacToe` URL and then tap "Go". Safari will ask you to open our TicTacToe app. After loggiing into the app, you will see not the start screen, but the game field, because this was configured when executing the workflow that we implemented.

You can also try to use `randomWin` as a game identifier instead of `ticTacToe`. In this case, you will be forwarded to a different game screen.

If you log into the game and switch to Safari without killing the app, then after typing in the URL you will be forwarded directly to the game field, the workflow won't wait until the players log in because it will immediately see that they are already logged in.

## Tutorial completed

Congratulations! You have completed tutorial 4. The completed source for this tutorial can be found [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial4-completed).