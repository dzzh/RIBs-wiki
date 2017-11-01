# Deep Linking

> Note: If you haven't completed [tutorial 3](Android-Tutorial-3) yet, we encourage you to do so before jumping into this tutorial.

## Goals
The goals of this code lab are to learn the following:
* Understand basics behind RIB workflows
* Learn how to create actionable item interfaces, implement their methods, and create workflows to launch specifics flows via deeplinks.

## Overview
This codelab will focus on adding a new workflow to launch the app and automatically set the player names and launch a new game.

To save time, the AndroidManifest.xml file for the codelab has been updated to handle any URL with the `rib-tutorial` scheme. In addition, the workflow libraries have already been integrated into the tutorial 4 starter code.


...

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
