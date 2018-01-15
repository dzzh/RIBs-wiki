<p align="center"><img src="https://github.com/uber/ribs/blob/assets/rib_horizontal_black.png" width="450" height="196" alt="RIBs"/></p>
<p> </p>
 

_This wiki provides an overview of how RIBs are designed. If you want to understand RIBs in detail, work through the tutorials._

# What are RIBs For?
RIBs is Uber’s cross-platform architecture framework. This framework is designed for large mobile applications that contain many nested states.

When designing this framework for Uber, we were adhering to the following principles:
* **Encourage Cross-Platform Collaboration:** Most of the complex parts of our apps are similar on both iOS and Android. RIBs present similar development patterns for Android and iOS. By using RIBs, engineers across both iOS and Android platforms can share a single, co-designed architecture for their features.
* **Minimize Global States and Decisions:** Global state changes cause unpredictable behavior and can make it impossible for engineers to know the full impact of their changes. RIBs encourage encapsulating states within a deep hierarchy of well-isolated individual RIBs, thus avoiding global state issues.
* **Testability and Isolation:** Classes must be easy to unit test and reason about in isolation. Individual RIB classes have distinct responsibilities (i.e. routing, business logic, view logic, creation of other RIB classes). In addition to that, most parts of the parent RIB logic is decoupled from its child RIB logic. This makes RIB classes easy to test and reason about independently.
* **Tooling for Developer Productivity:** Adopting non-trivial architectural patterns does not scale beyond small applications without robust tooling. RIBs come with IDE tooling around code generation, static analysis and runtime integrations — all of which improve developer productivity for large and small teams.
* **Open-Closed Principle:** Whenever possible, it should be able to add new features without modifying existing code. This can be seen in a few places when using RIBs. For example, you can attach or build a complex child RIB that requires dependencies from its parent with almost no changes to the parent RIB.
* **Structured around Business Logic:** The app’s business logic structure should not need to strictly mirror the structure of the UI. For example, to facilitate animations and view performance, the view hierarchy may want to be shallower than the RIB hierarchy. Or, a single feature RIB may control the appearance of three views that appear at different places in the UI.
* **Explicit Contracts:** Requirements should be declared with compile-time safe contracts. A class should not compile if its class dependencies and ordering dependencies are not satisfied. We use ReactiveX to represent ordering dependencies, type safe dependency injection (DI) systems to represent class dependencies and many DI scopes to encourage the creation of data invariants.

## Parts of a RIB
If you have previously worked with the [VIPER](https://mutualmobile.com/posts/meet-viper-fast-agile-non-lethal-ios-architecture-framework) architecture, then the class breakdown of a RIBs will look familiar to you. RIBs are usually composed of the following elements, with every element implemented in its own class:

<p align="center">
<img src="https://github.com/uber/ribs/blob/assets/documentation/ribs.png" width="800" alt="RIBs"/>
</p>


### Interactor
An Interactor contains business logic. This is where you perform Rx subscriptions, make state-altering decisions, decide where to store what data, and decide what other RIBs should be attached as children. 

All operations performed by the Interactor must be confined to its lifecycle. We have built tooling to ensure that business logic is only executed when the Interactor is active. This prevents scenarios where Interactors are deactivated, but subscriptions still fire and cause unwanted updates to the business logic or the UI state.


### Router
A Router listens to the Interactor and translates its outputs into attaching and detaching child RIBs. Routers exist for three simple reasons:

* Routers act as Humble Objects that make it easier to test complex Interactor logic without a need to to mock child Interactors or otherwise care about their existence.
* Routers create an additional abstraction layer between a parent Interactor and its child Interactors. This makes synchronous communication between Interactors a tiny bit harder and encourages adoption of reactive communication instead of direct coupling between the RIBs.
* Routers contain simple and repetitive routing logic that would otherwise be implemented by the Interactors. Factoring out this boilerplate code helps to keep the Interactors small and more focused on the core business logic provided by the RIB. 


### Builder
The Builder’s responsibility is to instantiate all the RIB’s constituent classes as well as the Builders for each of the RIB’s children. 

Separating the class creation logic in the Builder adds support for mockability on iOS and makes the rest of the RIB code indifferent to the details of DI implementation. The Builder is the only part of the RIB that should be made aware of the DI system used in the project. By implementing a different Builder, it is possible to reuse the rest of the RIB code in a project using a different DI mechanism.


### Presenter
Presenters are stateless classes that translate business models into view models and vice versa. They can be used to facilitate testing of view model transformations. However, often this translation is so trivial that it doesn’t warrant the creation of a dedicated Presenter class. If the Presenter is omitted, translating the view models becomes a responsibility of a View(Controller) or an Interactor. 


### View(Controller)
Views build and update the UI. This includes instantiating and laying out UI components, handling user interaction, filling UI components with data, and animations. Views are designed to be as “dumb” as possible. They just display information. In general, they do not contain any code that needs to be unit tested.


## State Management
Application state is largely managed and represented by the RIBs that are currently attached to the RIB tree. For example, as the user progresses through different states in a simplified ride sharing app, the app attaches and detaches the following RIBs (see GIF below).

<p align="center">
<img src="https://github.com/uber/ribs/blob/assets/documentation/state.gif" alt="State"/><br/>Example of state transitions in which lines denote RIB hierarchy.
</p>

RIBs only make state decisions within their scope. For example, the `LoggedIn` RIB only makes state decisions for transitioning between states like `Request` and `OnTrip`. It does not make any decisions about how to behave once we are on the `OnTrip` screen. 

Not all state can be stored by the addition or removal of the RIBs. For example, when a user’s profile settings change, no RIB gets attached or detached. Typically, we store this state inside the streams of immutable models that re-emit values when the details change. For example, the user’s name may be stored in a `ProfileDataStream` that lives inside the `LoggedIn` scope. Only network responses have write access to this stream. We pass an interface that provides read access to these streams down the DI graph.

There is nothing in RIBs that forces a single source of truth for the RIB state. This is in contrast to what more opinionated frameworks, like React, already provide out of the box.  Within the context of each RIB, you can choose to adopt patterns that promote unidirectional data flow, or you can allow business state and view state to temporarily diverge in order to take advantage of efficient platform animation frameworks.


## Communication Between RIBs
When an Interactor makes a business logic decision, it may need to inform another RIB of events, like completion, and send data. The RIB framework does not include a single way to pass data between RIBs. Nonetheless, it is built to facilitate some common patterns.

Typically, if communication is downward to a child RIB, we pass this information as emissions into Rx streams. Or, the data may be included as a parameter to a child RIB’s `build()` method, in which case this parameter becomes an invariant for the lifetime of the child.

<p align="center">
<img src="https://github.com/uber/ribs/blob/assets/documentation/stream.png" width="450" alt="RIBs"/><br/>Example of downwards communication via Rx. Lines denote RIB hierarchy.
</p>

If communication is going up the RIB tree to a parent RIB’s Interactor, then the communication is done via a listener interface since the parent can outlive the child. The parent RIB, or some object on its DI graph, implements the listener interface and places it on its DI graph so that its children RIBs can invoke it. Using this pattern to pass data upwards instead of having parents directly subscribe to Rx streams from their children has a few benefits. It prevents memory leaks, allows parents to be written, tested and maintained without knowledge of which children are attached, and reduces the amount of ceremony needed to attach/detach a child RIB. No Rx streams or listeners need to be unregistered/re-registered when attaching a child RIB this way.

<p align="center">
<img src="https://github.com/uber/ribs/blob/assets/documentation/listener.png" width="250" alt="RIBs"/><br/>
Example of upwards communication with a listener interface. Lines denote RIB hierarchy.
</p>


## RIB Tooling
In order to ensure smooth adoption of the RIB architecture across our four apps (per platform), we have invested in tooling to make RIBs easier to use and take advantage of the invariants created by adopting the RIBs architecture. Some of this tooling has been open sourced and will be discussed in tutorials. 

The RIB related tooling that we have so far open sourced includes:
* **Code generation:** IDE plugins for creating new RIBs and accompanying tests.
  * [iOS code generation templates for Xcode](https://github.com/uber/RIBs/tree/master/ios/tooling)
  * [Android code generation IDE plugin](https://github.com/uber/RIBs/tree/master/android/tooling/rib-intellij-plugin)
* **NPE Static analysis (Android):** [NullAway](https://github.com/uber/NullAway) is a static analysis tool that makes NullPointerExceptions a thing of the past.
* **Autodispose Static Analysis (Android):** Prevents the most common RIB memory leak.

Tooling that we are planning to open source in the future are:
* Static analysis that prevents a variety of RIB memory leaks
* RIB integration with runtime leak detection
* (Android) Annotation processors for making testing easier
* (Android) RxJava static analysis that ensures RIBs don’t mutate views off the main thread


## Where to Go From Here
We highly encourage you to run through each of the tutorials on the platform that you're developing for. You can find them in the side pane of this wiki.

For platform-specific questions, refer to the [iOS-specific questions](ios-specific-questions) and [Android-specific questions](android-specific-questions) pages.