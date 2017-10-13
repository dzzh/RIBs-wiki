Edits are currently being made in https://docs.google.com/document/d/1yw-WBzWqbMFdW-zsEqbLBt3ijKiaQdFkWR1gQjKvxmo/edit?ts=59e09a3f#heading=h.v5csdjewrc11 

This wiki provides an overview of how RIBs are designed. If you want to understand RIBs in detail, work through the tutorials.

## What are RIBs For?
RIBs are Uber’s cross-platform architecture framework. This framework is designed for large mobile applications that contain lots of volatile server driven state. 

When designing this framework, we emphasized the following principles:
* **Encourage Cross Platform Collaboration** Most of the complex parts of our apps are similar on both iOS and Android. RIBs present similar development patterns for Android and iOS. Android engineers should be able to code review iOS engineers code and vice-versa when not doing UI work.
* **Minimize Global States and Decisions** Global state changes cause unpredictable behavior and can make it impossible for engineers to know the full impact of their changes.
* **Structured around Business Logic** The app’s business logic structure should not need to strictly mirror the structure of the UI. For example: to facilitate animations and view performance the view hierarchy may want to be shallower than the router hierarchy. Or, a single feature RIB may control the appearance of three non-continquous views.
* **Explicit Contracts Requirements** should be declared with compile safe contracts. A class should not compile if it’s class dependencies and ordering dependencies are not satisfied. We use ReactiveX to represent ordering dependencies, type safe DI systems to represent class dependencies and many DI scopes to encourage the creation of data invariants.
* **Separating Logic Types** Merging business and view logic into single classes make systems harder to understand, modify and test.
* **Easy Decoupling** It must be trivial to extract features into their own compilation units for the sake of code reuse, build performance or enforcing encapsulation of responsibilities.
* **Open-Closed Principle** Whenever possible, it should be possible to add features without modifying existing code. This can be seen in a few places when using RIBs. For ex: you can attach/build a complex child RIB that requires a uses of dependencies from its parent without almost no changes to the parent.

## Parts of a RIB

If you’re familiar with VIPER then the class breakdown of RIBs will look familiar. RIBs are commonly made up of the following classes (one of each):
* Router
* Interactor
* Builder
* Presenter (optional)
* View (optional)

#### Interactors
The class that performs most business logic. This is where you perform Rx subscriptions, make state altering decisions, decide where/what data to store and decide what other RIBs should be attached as children. 
All operations performed by an Interactor must be confined to its lifecycle. We’ve built tooling to ensure business logic is only executed when the Interactor is active. This prevents scenarios where Interactors are deactivated, but subscriptions still fire and cause unwanted updates to business or UI states.

#### Routers
Router’s listen to Interactors, and translate their outputs into attaching and detaching child RIBs. Routers exist for two simple reasons:
* They act as Humble Objects that make it easier to test complex Interactor logic without needing to think-about/mock child Interactors
* By creating an additional layer between each Interactor and child Interactor, Routers make synchronous communication between Interactors a tiny bit harder, thereby encouraging adoption of Rx. Whether you consider this a benefit will depend on the nature of your app

#### Builder
The Builder’s responsibility is to instantiate all the RIB’s constituent classes plus the Builder for each RIB’s children. 

The Builder class needs to exist to support mockability on iOS and DI independence. Only the Builder is aware of the DI system used in your project. As a result of this, much of your app can be written in a DI tool independent way. Your project can use different DI tools per RIB.

#### Presenter
Presenter’s are stateless classes that translate business models into view models and vice versa. This type of class can be useful to facilitate testing view-model transformations. However, often this translation is so trivial that it doesn’t warrant the creation of a Presenter class. In this case you can omit the creation of a Presenter class. You do this by inlining the model translation into the Interactor of Presenter. Or doing the transformation implicitly.

#### View(Controller)
Views build and update the UI. This includes instantiating and laying out UI components, handling user interaction, filling UI components with data, and animations. Views are designed to be as “dumb” as possible. They just display information. In general, they don’t contain any code that needs to be unit tested.

## State Management
Application state is largely managed and represented by which RIBs are currently attached in the RIB tree. RIBs only make state decisions within their scope. For example, the LoggedIn RIB only makes state decisions for transitioning between states like Home, Confirmation, and Dispatching. It doesn’t make any decisions about how to behave once we’re on the Home screen. 

**This might be a good spot to plop http://eng.uber.com/wp-content/uploads/2017/08/image7.gif **

Naturally, not all state can be stored by the additional/removal of RIBs. For example, when an user’s name changes no RIB is attached or detached. Typically, we store this state inside streams of immutable models. For example, the user’s name may be stored in a ProfileDataStream that lives inside the LoggedIn scope. Only network responses have write access to this stream. We pass an interface that provides read access to these streams down the DI graph.

Unlike highly opinionated frameworks like React, there is nothing in RIBs that pushes a single source of truth for RIB state. Within the context of each RIB, you can choose to adopt patterns that promote unidirectional data flow. Or you can allow business state and view state to temporarily diverge in order to take advantage of efficient platform animation frameworks.

## Communication Between RIBs
When an Interactor makes a business logic decision it may need to inform another RIB of events, like completion, and send data. The RIB framework does not include a single way to pass data between RIBs. Nonetheless, it is built to facilitate some common patterns.

Typically, if communication is downward to a child RIB we pass this information as emissions into Rx streams. Or, the data may be included as a parameter to a child RIB’s build() method, in which case this parameter becomes an invariant for the lifetime of the child.

If communication is going up the RIB tree to a parent RIB’s Interactor, then the communication is done via a listener interface since the parent can outlive the child. The parent RIB, or some object on its DI graph, implements the listener interface and places it on its DI graph so that its children RIBs can invoke it. Using this pattern to pass data upwards instead of having parents directly subscribe to rx streams from their children has a few benefits. It prevents memory leaks, it allows parents to be written, tested and maintained without knowledge of which children are attached,  and it reduces the amount of ceremony needed to attach/detach a child RIB. No Rx streams or listeners need to be unregistered/re-registered when attaching a child RIB this way.

## RIB Tooling
In order to ensure smooth adoption of the RIB architecture across four apps (per platform) we’ve invested in tooling to make (1) RIBs easier to use (2) take advantage of the invariants created by adopting RIBs. Some of this tooling has been open sourced and will be discussed in tutorials. The most interesting pieces of tooling that we’re open sourcing now, or in the future, are:
* IDE plugins for creating new RIBs
* Static analysis that prevents common RIB memory leaks
* RIB integration with runtime leak detection
* RxJava static analysis that ensures RIBs don’t mutate views off the main thread
* NullAway, NPE static analysis that made NPEs a thing of the past for us