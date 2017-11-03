High-level architectural principles are fine and dandy. How do RIBs cope with the reality of the Android framework?

#### Question: How come none of the RIB diagrams show an activity in them?
RIB apps only have one activity in them. There is an implied RootActivity at the top of each diagrams that does nothing beyond calling attach(rootRouter).

#### Question: What about when you need to break the single activity model in order to open a second activity that can not be ported to RIBs (perhaps 3rd party activity)? How do you handle the onActivityForResult()?
Any RIB can start a second activity. When SecondActivity calls finishWithResult() the result is immediately delivered to the root of the FirstActivity. Since this data likely needs to be transmitted to the initiating RIB we use Workflows (see tutorial 4). This kills two birds with one stone: (1) it gets the data from the top of RootActivity down to the RIB that started the request (2) the workflow causes the RIB hierarchy to be recreated in its original form in case the SecondActivity had been killed. The benefit of using this pattern is that the same code path gets executed regardless of whether MainActivity had been killed while waiting for SecondActivity to finishWithResult().

If the SecondActivity is first party, you could also create an AppScope that both activities can be injected from. However, creating long lived app scopes like this is problematic and discouraged.

#### Question: How does this relate to MVP or MVI patterns that are popular in the Android community?
Mostly orthogonal. RIB framework provides flexibility around how you communicate between Views and Interactors. Some RIBs may use MVI. Some may not.

#### Question: Should we be diligent about persisting view state into databases or savedInstanceState?
You could. RIBs do have first party support for savedInstanceState. However this is less important when working in a single activity app.

#### Question: What about backstack?
Each RIB chooses how it wants to handle the back stack or pass it onto children. This is flexible since different backstack strategies make sense for different apps. Nonetheless, there are utility classes open sourced alongside that encapsulate common strategies. See `RouterNavigator` and `ScreenStack` utilities.

#### Question: What about screen rotation?
You can handle saving view state in RIBs with savedInstanceState or custom data stores. Uber rarely supports rotation in order to increase engineering and design efficiency. Therefore there aren’t strongly held best practices around screen rotation yet.

#### Question: Why the coupling with RxJava2? Seems like it is only needed for lifecycle handling.
We don’t think this architecture makes much sense without some reactive framework. If you’re using a reactive framework you should be using Rx2. So we haven’t prioritized removing this dependency.

#### Question: What about when you need services? For example, suppose MusicPlaybackService needs to subscribe to state inside LoggedInRIB’s scope. How would this work?
You can solve this multiple ways. 
* You could solve this by elevating the shared state into an AppScope that outlives the lifespan of the activity or service. If your service only needs a little bit of shared state with the RIB tree this works great. Just be careful to clean up the AppScope state during logout! 
* Alternatively, if you application is heavily based around services you could design a portion of your RIB hierarchy that exists independent of whether activities or views exist. Once you do this you no longer write any code inside Services. Instead you write code inside the RIB hierarchy. Once you do this Services are nothing more than a mechanism for keeping the app alive when backgrounded.
