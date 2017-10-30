# Create your first RIB

<img align="right" src="https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial1-create-a-rib/images/image1.jpg" width="200" />



Welcome to the tutorials, which have ben designed to give you a hands-on walkthrough through the core concepts of RIBs. As part of the tutorials, you'll be building a simple TicTacToe game using the RIBs architecture and associated tooling. 

We've provided you with a project that contains the boilerplate for a new RIBs application. You can find the source files [here](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial1). Follow the [README](https://github.com/uber/RIBs/tree/master/ios/tutorials/tutorial1/README.md) to install and open the project before reading any further.


## Goal

The goal of this exercise is to understand the various pieces of a RIB, and more importantly, how they interact and communicate with each other. At the end of this exercise, we should have an app that launches into a screen where the user can type in player names and tap on a login button. Tapping the button should then print player names to the Xcode console.  

## Create LoggedOut RIB

Select New File menu item on the LoggedOut group.  
<img src="https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial1-create-a-rib/images/image2.jpg" width="200" /> 

Select the Xcode RIB templates.  
<img src="https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial1-create-a-rib/images/image3.jpg" width="400" /> 

Name the new RIB "LoggedOut" and check the "Owns corresponding view" checkbox.  
<img src="https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial1-create-a-rib/images/image4.jpg" width="400" />  

LoggedOut RIB has its own view to display a "Sign Up" button and player name text fields.  

Create the files and make sure it’s in the "TicTacToe" target and in the "LoggedOut" folder.  
<img src="https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial1-create-a-rib/images/image5.jpg" width="400" /> 

Now we can delete the DELETE_ME.swift file. It was only temporarily needed so the project can compile without the LoggedOut RIB we just created.  

### Understanding generated code

![image7](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial1-create-a-rib/images/image7.jpg)

* LoggedOutBuilder conforms to LoggedOutBuildable so other RIBs that uses the builder can use a mocked instance that conforms to the buildable protocol.
* LoggedOutInteractor uses the protocol LoggedOutRouting to communicate with its router. This is based on the dependency inversion principle where the interactor declares what it needs, and some other unit, in this case the LoggedOutRouter, provides the implementation. Similar to the buildable protocol, this allows the interactor to be unit tested. LoggedOutPresentable is the same concept that allows the interactor to communicate with the view controller.
* LoggedOutRouter declares what it needs in LoggedOutInteractable to communicate with its interactor. It uses the LoggedOutViewControllable to communicate with the view controller.
* LoggedOutViewController uses LoggedOutPresentableListener to communicate with its interactor following the same dependency inversion principle.

### LoggedOut UI

Below is the UI we want to build. To save time, you can also use the provided UI code [here](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial1-create-a-rib/source/source1.swift?raw=true).

![image1](https://github.com/uber/ribs/blob/assets/tutorial_assets/ios/tutorial1-create-a-rib/images/image1.jpg)

### Login Logic

LoggedOutViewController calls to its listener, LoggedOutPresentableListener, to perform the business logic of login, by passing in the player 1 and 2 names.

```swift
protocol LoggedOutPresentableListener: class {
    func login(withPlayer1Name player1Name: String?, player2Name: String?)
}
```

* Notice that both player names are optional, since the user may not have entered anything. We could disable the Login button until both names are entered, but for this exercise, we’ll let the LoggedOutInteractor deal with the business logic of handling nil names.
* If player names are empty, we default to "Player 1" and "Player 2".