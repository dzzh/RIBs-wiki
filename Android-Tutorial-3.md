# RIB Dependency Injection and Communication

> Note: If you haven't completed [tutorial 2](Android-Tutorial-2) yet, we encourage you to do so before jumping into this tutorial.

Welcome to the RIBs tutorials, which have ben designed to give you a hands-on walkthrough through the core concepts of RIBs. As part of the tutorials, you'll be building a simple TicTacToe game using the RIBs architecture and associated tooling.

For tutorial 3, we'll start off where [tutorial 2](Android-Tutorial-2) ended. You can either continue with the project you completed in [tutorial 2](Android-Tutorial-2), or use the source code [here](https://github.com/uber/RIBs/tree/master/android/tutorials/tutorial3). Follow the [README](https://github.com/uber/RIBs/tree/master/android/tutorials/tutorial3/README.md) to install and open the project before reading any further.

## Goals
The goals of this code lab are to learn the following:
* Pass dependencies from a parent RIB to a child RIB.
* Pass data from a parent RIB to a child RIB via a stream.
* Understand how to properly unsubscribe from streams.
* Reinforce unit testing concepts.

## Overview
This codelab will focus on adding two new features to the TicTacToe app:
* Adding the ability to propagate the name entered by the player when logging in (currently the game just displays FixedName).
* Add the concept of scores to track state across multiple games.

To do this, player names will be passed into the build method of the LoggedInBuilder and the names will be available as a dependency for LoggedIn and its children.

To keep track of scores, a new score stream will be added at the LoggedIn scope. Games will signal game state changes via a listener pattern.

When done this tutorial the application will contain the following new behavior:
* Actual player names will be displayed instead of dummy data.
* When a player wins a game of tic tac toe, the app will route back to the OffGame screen.
* The OffGame screen will now display a win count for each player.


## Propagating Name Data
You’ll be writing this code lab inside the tutorial3 module.

### Step 1: Update the LoggedInBuilder to take player name dependencies
The LoggedInBuilder should receive this player names as dynamic dependencies in its build method. These dependencies are passed in via the build method because they don’t exist in the parent component and are ephemeral objects that only exist for a short period of time. 

Changes to LoggedInBuilder:
```java
public LoggedInRouter build(String playerOne, String playerTwo) {
  LoggedInInteractor interactor = new LoggedInInteractor();
  Component component = DaggerLoggedInBuilder_Component.builder()
      .parentComponent(getDependency())
      .interactor(interactor)
      .build();

  return component.loggedinRouter();
}
```

In addition to passing in the player names, we must add an entry to the Dagger component builder for the LoggedIn RIB. This allows the LoggedIn RIB and any children of the LoggedIn RIB to inject the player names.

New methods added to the module in the LoggedInBuilder’s component builder:
```java
@BindsInstance
Builder playerOne(@Named("player_one") UserName playerOne);

@BindsInstance
Builder playerTwo(@Named("player_two") UserName playerTwo);
```

NOTE:
These dependencies are provided using the new BindsInstance Dagger API. It’s similar to using @Provides, but allows us to not have to pass the player names into the module via its constructor.

Now that Dagger knows about the new dependencies, we must go back and update the build method again to bind the variables:

```java
@NonNull
public LoggedInRouter build(String playerOne, String playerTwo) {
  LoggedInInteractor interactor = new LoggedInInteractor();
  Component component = DaggerLoggedInBuilder_Component.builder()
      .parentComponent(getDependency())
      .interactor(interactor)
      .playerOne(playerOne)
      .playerTwo(playerTwo)
      .build();

  return component.loggedinRouter();
}
```

With these changes, the player names are now available as a dependency in the LoggedIn RIB. However, the app won’t compile until we provide the names to the build method.

### Step 2: Pass the player names into the LoggedInBuilder
Next, we must pass the player names into the LoggedInBuilder’s build method. To do this, the attachLoggedIn method in RootRouter must be updated to receive player names as parameters:
```java
void attachLoggedIn(String playerOne, String playerTwo) {
  // No need to attach views in any way.
  attachChild(loggedInBuilder.build(playerOne, playerTwo));
}
```

Next, we’ll pass the player names to the router method when the user hits the login button. To do this, we’ll update the call to attachLoggedIn in RootInteractor:
```java
class LoggedInListener implements LoggedOutInteractor.Listener {
  @Override
  public void requestLogin(String playerOne, String playerTwo) {
    getRouter().detachLoggedOut();
    getRouter().attachLoggedIn(playerOne, playerTwo);
  }
}
```

Now that all the DI plumbing is done, the app should compile again.

### Step 3: Display the player names in the OffGame RIB
Now that the name dependency is wired up, we can use the data in the LoggedIn RIB and its children. Let’s use the name data to display the names on the screen when in the OffGame RIB.

Since the OffGame RIB is a child of the LoggedIn RIB, it can declare that it needs the player name dependencies in its ParentComponent:
```java
public interface ParentComponent extends OffGameOptionalExtension.ParentComponent {
  @Named("player_one") String playerOne();
  @Named("player_two") String playerTwo();
  OffGameInteractor.Listener offGameListener();
}
```

Next, we’ll update the OffGamePresenter interface to add a new method to pass in player names:
```java
interface OffGamePresenter {
  void setPlayerNames(String playerOne, String playerTwo);
  Observable<Irrelevant> startGameRequest();
}
```

Next, implement the new presenter method in OffGameView. 

Lastly, we’ll want to inject the names into the OffGameInteractor

```java
@Inject @Named("player_one") String playerOne;
@Inject @Named("player_two") String playerTwo;
```

And call the new presenter method in didBecomeActive:

```java
@Override
protected void didBecomeActive(@Nullable Bundle savedInstanceState) {
   super.didBecomeActive(savedInstanceState);
   presenter.setPlayerNames(playerOne, playerTwo);
   ...
}
```

Now, running the app should show the correct player names when in the off game state:


## Keeping Track Of Scores
Next, we’ll add a new ScoreStream class, this will be owned by the LoggedIn RIB and will be used to emit a map of UserName objects to scores. The OffGame RIB can then observe this stream  and display a win count for each player.

### Step 1: Creating the score stream
To get started, we’ll make a new class in the logged_in package called MutableScoreStream. This class will hold the current scores, and have methods to update the stored value:
```java
class MutableScoreStream implements ScoreStream {

  private final BehaviorRelay<ImmutableMap<String, Integer>> scoresRelay = BehaviorRelay.create();

  MutableScoreStream(String playerOne, String playerTwo) {
    scoresRelay.accept(ImmutableMap.of(playerOne, 0, playerTwo, 0));
  }

  void addVictory(String userName) {
    ImmutableMap<String, Integer> currentScores = scoresRelay.getValue();

    ImmutableMap.Builder<String, Integer> newScoreMapBuilder = new ImmutableMap.Builder<>();
    for (Map.Entry<String, Integer> entry : currentScores.entrySet()) {
      if (entry.getKey().equals(userName)) {
        newScoreMapBuilder.put(entry.getKey(), entry.getValue() + 1);
      } else {
        newScoreMapBuilder.put(entry.getKey(), entry.getValue());
      }
    }

    scoresRelay.accept(newScoreMapBuilder.build());
  }
}
```

A few things worth noting here:
Each time the score is updated, a new copy of the score map is emitted (as opposed to mutating the existing map and emitting it again).
This class is intentionally package-private, since only the LoggedIn RIB should know about it.

Now that we have our MutableScoreStream, we’ll want to create an Immutable version that child RIBs can observe. We’ll call this just ScoreStream:
```java
public interface ScoreStream {
   Observable<ImmutableMap<String, Integer>> scores();
}
```

Next, we’ll circle back to the MutableScoreStream and have it implement ScoreStream:
```java
class MutableScoreStream implements ScoreStream {

  private final BehaviorRelay<ImmutableMap<String, Integer>> scoresRelay = BehaviorRelay.create();

  MutableScoreStream(String playerOne, String playerTwo) {
    scoresRelay.accept(ImmutableMap.of(playerOne, 0, playerTwo, 0));
  }

  void addVictory(String userName) {
    ImmutableMap<String, Integer> currentScores = scoresRelay.getValue();

    ImmutableMap.Builder<String, Integer> newScoreMapBuilder = new ImmutableMap.Builder<>();
    for (Map.Entry<String, Integer> entry : currentScores.entrySet()) {
      if (entry.getKey().equals(userName)) {
        newScoreMapBuilder.put(entry.getKey(), entry.getValue() + 1);
      } else {
        newScoreMapBuilder.put(entry.getKey(), entry.getValue());
      }
    }

    scoresRelay.accept(newScoreMapBuilder.build());
  }

  @Override
  public Observable<ImmutableMap<String, Integer>> scores() {
    return scoresRelay.hide();
  }
}
```

### Step 2: Providing the score stream in the LoggedIn scope
Now that we have all the required classes and interfaces, we’ll need to provide them. Since the MutableScoreStream is owned by the LoggedIn RIB, a new provider needs to be added to the LoggedInBuilder:
```java
@LoggedInScope
@LoggedInInternal
@Provides
static MutableScoreStream mutableScoreStream(
    @Named("player_one") String playerOne,
    @Named("player_two") String playerTwo) {
  return new MutableScoreStream(playerOne, playerTwo);
}
```

It’s worth pointing out the @LoggedInInternal qualifier - this a Dagger qualifier that is generated for free with every RIB when using the IntelliJ template. It’s a package-private qualifier to prevent child RIBs from listing the MutableScoreStream in their ParentComponent.

Also worth noting, using @LoggedInScope ensures that the score stream is a singleton for the logged in RIB and all of its children.

Now that the mutable class is on the dependency graph, we can also add a provider for the immutable ScoreStream interface (which child RIBs are allowed to use) in the module in the LoggedInBuilder:
```java
@LoggedInScope
@Binds
abstract ScoreStream scoreStream(@LoggedInInternal MutableScoreStream mutableScoreStream);
```

If you haven’t seen @Binds before, this is just shorthand for creating an @Provides method that takes the MutableScoreStream and returns a ScoreStream.
Step 3: Subscribing to the score stream in the OffGame scope
Now that the score stream is wired up, we can subscribe to it in the OffGame RIB.

First we must add a dependency to the OffGame RIB’s parent component:
```java
public interface ParentComponent {

  @Named("player_one") String playerOne();

  @Named("player_two") String playerTwo();

  OffGameInteractor.Listener listener();

  ScoreStream scoreStream();
}
```

Now we can inject the ScoreStream into the OffGameInteractor, but before we do that, we need to add a new presenter API to pass the scores to the views:
```java
interface OffGamePresenter {
  void setPlayerNames(String playerOne, String playerTwo);

  void setScores(Integer playerOneScore, Integer playerTwoScore);

  Observable<Object> startGameRequest();
}
```

Next, implement the new API in the OffGameView to update the displayed scores (there are two UTextView fields in OffGame to hold the scores - playerOneScore and playerTwoScore).

Now that the presenter and view have been updated, we can subscribe to the stream in the OffGameInteractor. First we must inject the stream into the OffGameInteractor:
```java
@RibInteractor
public class OffGameInteractor
    extends Interactor<OffGameInteractor.OffGamePresenter, OffGameRouter> {

  @Inject @Named("player_one") String playerOne;
  @Inject @Named("player_two") String playerTwo;
  @Inject Listener listener;
  @Inject OffGamePresenter presenter;
  @Inject ScoreStream scoreStream;

  ...
}
```

Now we can subscribe to the score stream in OffGameInteractor’s didBecomeActive lifecycle method and pass new scores to the presenter when they arrive:
```java
scoreStream.scores()
    .subscribe(new Consumer<ImmutableMap<String,Integer>>() {
      @Override
      public void accept(ImmutableMap<String, Integer> scores)
          throws Exception {
        Integer playerOneScore = scores.get(playerOne);
        Integer playerTwoScore = scores.get(playerTwo);
        presenter.setScores(playerOneScore, playerTwoScore);
      }
    });
```

Now, when we run the app, it’ll display the current score for each player instead of dummy text:
<img src="https://github.com/uber/RIBs/blob/master/android/tutorials/tutorial3-rib-di-and-communication-starter-code/tutorial_assets/off_game.png?raw=true" width="600">

Let’s take a look at our Rx subscription to the ScoreStream again - what happens if we want to detach and garbage collect the OffGame RIB? Currently, it will cause a memory leak because it’s subscribed to the ScoreStream which is scoped to the LoggedInScope. To fix this, we update our code to use AutoDispose to automatically unsubscribe when OffGame is detached:
```java
scoreStream.scores()
    .to(new ObservableScoper<ImmutableMap<String, Integer>>(this))
    .subscribe(new Consumer<ImmutableMap<String,Integer>>() {
      @Override
      public void accept(ImmutableMap<String, Integer> scores)
          throws Exception {
        Integer playerOneScore = scores.get(playerOne);
        Integer playerTwoScore = scores.get(playerTwo);
        presenter.setScores(playerOneScore, playerTwoScore);
      }
    });
```

### Step 4: Updating the ScoreStream when a player wins
Now that the score stream is all wired up and OffGame is displaying its data, it would be useful to actually update it.

Since the TicTacToe RIB is a child of the LoggedIn RIB, we’ll want to use the listener pattern here.

More specifically we’ll want:
A new listener defined in the TicTacToe RIB. This listener should have a method to signal that the game has ended and a specific player has won.
The LoggedIn interactor should implement this listener, when called it should update the MutableScoreStream and route the user back to the OffGame RIB so they can view their stats. (when injecting the MutableScoreStream into the LoggedInInteractor, be sure to also use the @LoggedInInternal qualifier, otherwise Dagger won’t be able to find the dependency).

Hopefully this is a review of concepts that have been used in past steps, so there are no example code snippets here. However, if you need help or are stuck, don’t hesitate to reach out.

## (Bonus) Update unit tests
If you reach this point, it means the app now saves names and keeps track of scores. However, because we have added quite a bit of new dependencies, the unit tests will not compile.

Now is a great time to update the unit tests to build along with adding some new test cases.

Some example ideas include:
* Ensuring OffGame correctly sets the scores when the score stream emits.
* Ensuring the TicTacToe RIB correctly calls its listener when a game ends.
* Ensuring the OffGameView properly formats score data.

## Tutorial complete

Congratulations! You completed tutorial 3. The completed source for this tutorial can be found [here].(https://github.com/uber/RIBs/tree/master/android/tutorials/tutorial3-completed)

Now onwards to [tutorial 4](Android-Tutorial-4).