# Deep Linking Workflows

> Note: If you haven't completed [tutorial 3](Android-Tutorial-3) yet, we encourage you to do so before jumping into this tutorial.

## Goals
The goals of this code lab are to learn the following:
* Understand basics behind RIB workflows
* Learn how to create actionable item interfaces, implement their methods, and create workflows to launch specifics flows via deeplinks.

## Background

The RIB deeplinking method is designed with the following goals in mind:

* Define every step in a deeplink's workflow in a single place despite acting over many places in the code base
* Should *not* be broken silently when app refactorings are made. If you introduce a gap in a workflow's chain it will no longer compile
* Should be tolerant of entering the app in a state other than the intended app state. For example: capable of waiting for the user to login or begin a trip.

Complex deeplinking behaviors can be built like the following example:

```java
@Override
protected Step<Step.NoValue, ? extends ActionableItem> getSteps(
    final RootActionableItem rootActionableItem,
    final ScheduledRidesDeepLink scheduledRidesDeepLink) {
  return rootActionableItem
      .waitUntilSignedIn()
      .onStep((noValue, rootSignedInActionableItem) -> rootSignedInActionableItem.goToLoggedIn())
      .onStep((value, loggedInActionableItem) -> loggedInActionableItem.goToRide())
      .onStep((noValue, rideActionableItem) -> rideActionableItem.isOnTrip())
      .onStep((isOnTrip, rideActionableItem) -> {
            if (isOnTrip) {
              // Terminate the flow
              return Step.fromOptional(Single.just(Optional.absent()));
            } else {
              return rideActionableItem.waitForRequest();
            }
          })
      .onStep((noValue, addressEntryActionableItem) -> addressEntryActionableItem.waitForAddressEntry())
      .onStep((noValue, addressEntryActionableItem) -> addressEntryActionableItem.showScheduledRideDatePicker(
              scheduledRidesDeepLink.getSource()))
      .onStep((noValue, addressEntryActionableItem) -> addressEntryActionableItem.getAddressEntryComponent())
      .onStep((component, addressEntryActionableItem) -> showMessageAboutBeingAwesome(component));
}
```

## Overview
This codelab will focus on adding a new workflow to launch the app and automatically set the player names and launch a new game.

To save time, the AndroidManifest.xml file for the codelab has been updated to handle any URL with the `rib-tutorial` scheme. In addition, the workflow libraries have already been integrated into the tutorial 4 starter code.

The actions that need to happen for the workflow look like this:

<img src="https://github.com/uber/RIBs/blob/assets/tutorial_assets/android/tutorial4_rib.png?raw=true" width="600">

## Goals


## Creating ActionableItem Interfaces and Implementing Them

Before we dive into coding this, I’d like to highlight the `Step` class. The `Step` type is what all actions must return (so Workflows can chain them together). Each step encapsulates a piece of work to be performed, it also has two generics - the first generic is an optional return type to be passed into the next step (if you don’t have a return value - you can use Step.NoValue), the second generic is the next actionable item interface that the user will be placed in when the step is finished doing its work. 

For the `RootActionableItem`, we’ll be creating a single action to check if the user is logged in, it will return a step with no return type (because future steps don’t need anything here) and the next actionable item will be the `LoggedInActionableItem`.

With that being said, let’s add this new method to the `RootActionableItem`:

```java
@Override
public Step<Step.NoValue, LoggedInActionableItem> logIn(final UserName playerOne, final UserName playerTwo) {
   return Step.from(
           Single.defer(new Callable<SingleSource<? extends Step.Data<Step.NoValue, LoggedInActionableItem>>>() {
               @Override
               public SingleSource<Step.Data<Step.NoValue, LoggedInActionableItem>> call() {
                   getRouter().detachLoggedOut();
                   LoggedInActionableItem loggedInActionableItem
                           = getRouter().attachLoggedIn(playerOne, playerTwo);
                   return Single.just(Step.Data.toActionableItem(loggedInActionableItem));
               }
           }));
}
```

> This tutorial's code would be a lot nicer if you enable lambda's in your project. But still usable without lambdas.

All this does is attach the logged in router, and then return it as the next actionable item. There are a few subtle things worth noting here though:
* All of the code that performs changes is wrapped within the Rx single. If you were to place code outside of this block, it would execute at the time the workflow is created, as opposed to when it is this actions turn to perform its work.
* `attachLoggedIn()` has been updated to return the `LoggedInActionableItem` (which is really just the LoggedInInteractor behind an interface). We use the actionable item interface to have a clear list of actions that can be performed instead of just exposing the entire public API for the interactor.

## Creating the Workflow
### Step 1: Create the workflow

Now that we have a single action defined, let’s create a workflow to test it out.  In your root package, create a new class called LaunchGameWorkflow and insert this boilerplate:

```java

public class LaunchGameWorkflow extends RootWorkflow<Step.NoValue, LaunchGameWorkflow.LaunchGameDeepLinkModel> {

    public LaunchGameWorkflow(@NonNull Intent deepLinkIntent) {
        super(deepLinkIntent);
    }

    @NonNull
    @Override
    protected Step<Step.NoValue, ? extends ActionableItem> getSteps(
            @NonNull RootActionableItem rootActionableItem,
            @NonNull LaunchGameDeepLinkModel launchGameDeepLinkModel) {
        return null;
    }

    @NonNull
    @Override
    protected LaunchGameDeepLinkModel parseDeepLinkIntent(@NonNull Intent deepLinkIntent) {
        return null;
    }

    @Validated(factory = TrainingSessionsValidationFactory.class)
    public static class LaunchGameDeepLinkModel implements RootWorkflowModel {
      // Unimplemented class
    }
}
```

This is obviously not fully implemented, but let’s look over a few details here before filling it out. First, let’s take a look at the generics - the first generic is the return type for the entire Workflow, in this case there is no return type. The second generic is the POJO model class that is used to hold information about the deep link.

To create the pojo model, parseDeeplinkIntent will be implemented to convert an intent to a LaunchGameDeepLinkModel. In addition, the deeplink plugin will run this model through RAVE to ensure the data is valid.

Next, the deep link model will be passed to getSteps() to create a workflow with the model and the initial root actionable item.

First, let’s implement the the LaunchGameDeepLinkModel class and parseDeepLinkIntent method to properly pull out paramters into a model.

Your LaunchGameDeepLinkModel should look like the following:

```java
@Validated(factory = TrainingSessionsValidationFactory.class)
static class LaunchGameDeepLinkModel implements RootWorkflowModel {
   private final String playerOneName;
   private final String playerTwoName;
   private final String gameName;

   public LaunchGameDeepLinkModel(String playerOneName, String playerTwoName, String gameName) {
       this.playerOneName = playerOneName;
       this.playerTwoName = playerTwoName;
       this.gameName = gameName;
   }

   @NonNull
   public String getPlayerOneName() {
       return playerOneName;
   }

   @NonNull
   public String getPlayerTwoName() {
       return playerTwoName;
   }

   @NonNull
   public String getGameName() {
       return gameName;
   }
}
```

Next we’ll implement parseDeepLink to turn the intent into a LaunchGameDeepLinkModel instance:

```java
@NonNull
@Override
protected LaunchGameDeepLinkModel parseDeepLinkIntent(@NonNull Intent deepLinkIntent) {
  Uri uri = deepLinkIntent.getData();
  String playerOne = uri.getQueryParameter("playerOne");
  String playerTwo = uri.getQueryParameter("playerTwo");
  String gameName = uri.getQueryParameter("gameKey");

  return new LaunchGameDeepLinkModel(playerOne, playerTwo, gameName);
}
```

In the event one of these parameters are missing, this model will fail [RAVE](https://github.com/uber-common/rave) validation and a workflow won’t be launched.

Next, let’s implement getSteps() to return a single step to log the user in:

```java
@NonNull
@Override
protected Step<Step.NoValue, ? extends ActionableItem> getSteps(
       @NonNull RootActionableItem rootActionableItem,
       @NonNull LaunchGameDeepLinkModel launchGameDeepLinkModel) {
   return rootActionableItem.logIn(
           UserName.create(launchGameDeepLinkModel.getPlayerOneName()), UserName.create(launchGameDeepLinkModel.getPlayerTwoName()));
}
```

### Step 2: Integrate the Workflow

Now that we a workflow let’s integrate it. Open up `WorkflowFactory` and add your workflow to the list.

### Step 3: Test it out
Now that all this wiring is done let’s test it out. You can launch the app using the example URL via adb:
`adb shell am start -a "android.intent.action.VIEW -d rib-tutorials://launchGame?playerOne=PlayerOne\&playerTwo=PlayerTwo\&gameKey=TicTacToe"`

You should see the app launch in a logged in state.

## Routing the User to the Specific Game

Now that most of the plumbing is done, let’s add the remaining steps.

### Step 1: Add a new action to route the user to their selected game.

First, let’s add a new method to the LoggedInActionableItem interface:
```java
Step<Step.NoValue, LoggedInActionableItem> launchGame(String gameKey);
```

Implementing this method in LoggedInInteractor is left as an exercise to the reader. Here are some tips:
* A list of games is now available in LoggedInInteractor.
* Be sure to perform work in a deferred Single.
* If a game is not available, just `Log.e()` for now

### Step 2: Add the new action to the workflow

Now that we have the new action, the workflow needs to be updated to use it:

```java
@NonNull
@Override
protected Step<Step.NoValue, ? extends ActionableItem> getSteps(
       @NonNull RootActionableItem rootActionableItem,
       @NonNull final LaunchGameDeepLinkModel launchGameDeepLinkModel) {
   return rootActionableItem.logIn(
           UserName.create(launchGameDeepLinkModel.getPlayerOneName()),
           UserName.create(launchGameDeepLinkModel.getPlayerTwoName())
   ).onStep(
           new BiFunction<Step.NoValue, LoggedInActionableItem, Step<Step.NoValue, LoggedInActionableItem>>() {
               @Override
               public Step<Step.NoValue, LoggedInActionableItem> apply(
                       Step.NoValue noValue, LoggedInActionableItem loggedInActionableItem) {
                   return loggedInActionableItem.launchGame(launchGameDeepLinkModel.getGameName());
               }
           });
}
```

Note the usage of the onStep method here - this allows you to chain steps together.

### Step 3: Test it out
Let’s test it again with the example link:
`adb shell am start -a "android.intent.action.VIEW -d rib-tutorials://launchGame?playerOne=PlayerOne\&playerTwo=PlayerTwo\&gameKey=TicTacToe"`

Now, the app should launch into TicTacToe with the correct player names.


