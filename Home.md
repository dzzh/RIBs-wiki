<img src="https://github.com/uber/ribs/blob/assets/rib_horizontal_black.png" width="450" height="196" alt="RIBs"/>
<p></p>
 

This wiki provides an overview of how RIBs are designed. If you want to understand RIBs in detail, work through the tutorials.
What are RIBs For?
RIBs is Uber’s cross-platform architecture framework. This framework is designed for large mobile applications that contain many nested states.

When designing this framework for Uber, we emphasized the following principles:
* **Encourage Cross-Platform Collaboration:** Most of the complex parts of our apps are similar on both iOS and Android. RIBs present similar development patterns for Android and iOS. By using RIBs, engineers across both iOS and Android platforms can share a single, co-designed architecture for their features.
* **Minimize Global States and Decisions:** Global state changes cause unpredictable behavior and can make it impossible for engineers to know the full impact of their changes. RIBs encourage encapsulating states within a deep hierarchy of well-isolated individual RIBs, thus avoiding global state issues.
* **Testability and Isolation:** Classes must be easy to unit test and reason about in isolation. Individual RIB classes have distinct responsibilities (i.e.,: routing, business, view logic, creation). Plus, most RIB logic is decoupled from child RIB logic. This makes RIB classes easy to test and reason about independently.
* **Tooling for Developer Productivity:** Adopting non-trivial architecture patterns does not scale beyond small applications without robust tooling. RIBs come with IDE tooling around code generation, static analysis and runtime integrations--all of which improve developer productivity for large teams and small.
* **Open-Closed Principle:** Whenever possible, it should be possible to add features without modifying existing code. This can be seen in a few places when using RIBs. For example: you can attach/build a complex child RIB that requires uses of dependencies from its parent without almost no changes to the parent.
* **Structured around Business Logic:** The app’s business logic structure should not need to strictly mirror the structure of the UI. For example: to facilitate animations and view performance the view hierarchy may want to be shallower than the RIB hierarchy. Or, a single feature RIB may control the appearance of three views that appear different places in the UI.
* **Explicit Contracts:** Requirements should be declared with compile-time safe contracts. A class should not compile if its class dependencies and ordering dependencies are not satisfied. We use ReactiveX to represent ordering dependencies, type safe DI systems to represent class dependencies and many DI scopes to encourage the creation of data invariants.

## Parts of a RIB

If you’re familiar with VIPER then the class breakdown of RIBs will look familiar. RIBs are commonly made up of the following classes (one of each):

<p align="center">
<img src="https://github.com/uber/ribs/blob/assets/documentation/ribs.png" width="800" alt="RIBs"/>
</p>

### Interactors
The class that performs business logic. This is where you perform Rx subscriptions, make state altering decisions, decide where/what data to store and decide what other RIBs should be attached as children. 

All operations performed by an Interactor must be confined to its lifecycle. We’ve built tooling to ensure business logic is only executed when the Interactor is active. This prevents scenarios where Interactors are deactivated, but subscriptions still fire and cause unwanted updates to business or UI states.


### Router
Routers listen to Interactors, and translate their outputs into attaching and detaching child RIBs. Routers exist for three simple reasons:
* They act as Humble Objects that make it easier to test complex Interactor logic without needing to think-about/mock child Interactors.
* By creating an additional layer between each Interactor and child Interactor, Routers make synchronous communication between Interactors a tiny bit harder, thereby encouraging adoption of reactive communication instead of direct coupling between RIBs.
* By separating out the simple and repetitive routing logic from the complex business logic inside interactors, the business logic inside interactors becomes easier to parse and understand. 


### Builder
The Builder’s responsibility is to instantiate all the RIB’s constituent classes plus the Builder for each RIB’s children. 

Separating the class creation logic in the builder adds support for mockability on iOS and DI independence. Only the Builder is aware of the DI system used in your project. As a result of this, much of your app can be written in a DI tool independent way. Your project can use different DI tools per RIB.

### Presenter
Presenters are stateless classes that translate business models into view models and vice versa. This type of class can be useful to facilitate testing view-model transformations. However, often this translation is so trivial that it doesn’t warrant the creation of a Presenter class. In this case you can omit the creation of a Presenter class. You do this by inlining the model translation into the view/view-controller or Interactor. Or doing the transformation implicitly.

### View(Controller)
Views build and update the UI. This includes instantiating and laying out UI components, handling user interaction, filling UI components with data, and animations. Views are designed to be as “dumb” as possible. They just display information. In general, they don’t contain any code that needs to be unit tested.

## State Management
Application state is largely managed and represented by which RIBs are currently attached in the RIB tree. For example, as the user progresses through different states in a simplified ride sharing app the app attaches and detaches the following RIBs (see gif below).

<p align="center">
<img src="https://github.com/uber/ribs/blob/assets/documentation/state.gif" alt="State"/>
</p>

RIBs only make state decisions within their scope. For example, the LoggedIn RIB only makes state decisions for transitioning between states like Request, and OnTrip. It doesn’t make any decisions about how to behave once we’re on the OnTrip screen. 

Not all state can be stored by the addition/removal of RIBs. For example, when an user’s profile settings change no RIB is attached or detached. Typically, we store this state inside streams of immutable models that re-emit when details change. For example, the user’s name may be stored in a ProfileDataStream that lives inside the LoggedIn scope. Only network responses have write access to this stream. We pass an interface that provides read access to these streams down the DI graph.

There is nothing in RIBs that forces a single source of truth for RIB state. This is in contrast to what some opinionated frameworks, like React, already provide out of the box.  Within the context of each RIB, you can choose to adopt patterns that promote unidirectional data flow. Or you can allow business state and view state to temporarily diverge in order to take advantage of efficient platform animation frameworks.


## Communication Between RIBs
When an Interactor makes a business logic decision it may need to inform another RIB of events, like completion, and send data. The RIB framework does not include a single way to pass data between RIBs. Nonetheless, it is built to facilitate some common patterns.

Typically, if communication is downward to a child RIB we pass this information as emissions into Rx streams. Or, the data may be included as a parameter to a child RIB’s build() method, in which case this parameter becomes an invariant for the lifetime of the child.

<p align="center">
<img src="https://github.com/uber/ribs/blob/assets/documentation/stream.png" width="450" alt="RIBs"/><br/>Example of downwards communication via Rx. Lines denote RIB hierarchy.
</p>

If communication is going up the RIB tree to a parent RIB’s Interactor, then the communication is done via a listener interface since the parent can outlive the child. The parent RIB, or some object on its DI graph, implements the listener interface and places it on its DI graph so that its children RIBs can invoke it. Using this pattern to pass data upwards instead of having parents directly subscribe to rx streams from their children has a few benefits. It prevents memory leaks, it allows parents to be written, tested and maintained without knowledge of which children are attached,  and it reduces the amount of ceremony needed to attach/detach a child RIB. No Rx streams or listeners need to be unregistered/re-registered when attaching a child RIB this way.

<p align="center">
<img src="https://github.com/uber/ribs/blob/assets/documentation/listener.png" width="250" alt="RIBs"/><br/>
Example of upwards communication with a listener interface. Lines denote RIB hierarchy.
</p>


## RIB Tooling
In order to ensure smooth adoption of the RIB architecture across four apps (per platform) we’ve invested in tooling to make (1) RIBs easier to use (2) take advantage of the invariants created by adopting RIBs. Some of this tooling has been open sourced and will be discussed in tutorials. The most interesting pieces of RIB related tooling that we’re open sourcing now, or in the future, are:
* IDE plugins for creating new RIBs
* Static analysis that prevents common RIB memory leaks
* RIB integration with runtime leak detection
* Code generation to make testing easier
* (Android) RxJava static analysis that ensures RIBs don’t mutate views off the main thread
* (Android) NullAway, NPE static analysis that made NPEs a thing of the past for us


## Where to Go From Here
We highly encourage you to run through each of the tutorials on the platform that you're developing for. You can find them in the side pane of this wiki.

For platform specific questions, refer to the [iOS specific questions](ios-specific-questions) and [Android specific questions](android-specific-questions) pages.

