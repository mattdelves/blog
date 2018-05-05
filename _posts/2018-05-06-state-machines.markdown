---
layout: post
title:  "State Machines and the view"
date:   2018-05-06 10:00:00 +1000
categories: adventure
---

The idea of a state that represents what a view is about is something that comes up time and time again in development of an application. But the question remains, what exactly is the state of a view? Is it the data of the view? Is it the visual appearance? Is it some combination? Is it the position within a complex flow of user behaviour that identifies what's going on?

It is all of the above. When people talk about the state of a view, they tend to mean the data that populates the view. The details of the objects on the screen. This is typical in a react like application, but that's not the focus of this article. This article is about the position within a complex flow. That is, how does a finite state machine get used in representing the behaviour of the app and what is represented on screen.

What is a finite state machine? A finite state machine is a thing (for lack of a better term) that defines a limited set of states that can exist. The state machine also defines the possible transitions between the states. This sounds very abstract and confusing so lets try and be a bit more concrete about what's going on.

First, let's define the states. We'll be working with a fictional onboarding flow and this can be defined as such.

``` swift
enum OnboardingFlow: State {
  case initial
  case pushNotificationsAsk
  case pushNotificationsDisplay
  case locationAsk
  case locationDisplay
  case onboarded
}
```

As the state is define as an enum, we can be sure at compile time that the state can't be different from the defined options. We have defined the parts of the flow as destinct interactions that are possible. But just having defined these isn't enough. There needs to be a definition of the possible transitions between states. This is where a state machine becomes useful.

What is a state machine then? We have defined what a finite state is, so how does a state machine fit in. A state machine defines the transitions and behaviour of the state. That is, whether it is possible for a transition to occur between states. Let's start to flesh out what happens in our state machine to determine whether a transition is possible.

Let's build out our state machine. Our machine needs to have some kind of type that it operates on. This is defined in the delegate of the state machine. We are using the delegate pattern to keep any information about the specifics of a state or business rules out of the mechanics of the state machine. The state machine will hand off all specifics to the delegate. That is, it delegates.

``` swift
protocol StateMachineDelegate: class {
  associatedtype StateType: State

  func shouldTransition(from: StateType, to: StateType) -> Bool
  func didTransition(from: StateType, to: StateType)
}

class StateMachine<DelegateType: StateMachineDelegate> {
  weak var delegate: DelegateType?

  init(initialState: DelegateType.StateType) {
    currentState = initialState
  }

  private var currentState: DelegateType.StateType

  var state: DelegateType.StateType {
    get {
      return currentState
    }
    set {
      guard let delegate = delegate, delegate.shouldTransition(from: self.state, to: newValue) else { return }

      let oldValue = currentState
      currentState = newValue
      delegate.didTransition(from: oldValue, to: newValue)
    }
  }
}
```

It's a very simple device. Let's break it apart by first examining what's going on in the delegate. There are two functions which need to be implemented and also an associated type. Those two functions are what lets us determine whether a transition is possible (`shouldTransition`) and also respond to a transition having taken place (`didTransition`). The mechanics of the state machine make use of these functions to change state. The state machine will first check whether the transition is possible and if so, set the value of the state. If a transition has occured (new value set) then the `didTransition` function is called.

Continuing to build out our example lets us see how this all fits together. We want to prompt the user at steps in the flow and also make sure that they are transitioned cleanly between the screens. The example is just going to focus on the delegate side and show how it can be used to define the onboarding flow.

Next up is the delegate:

``` swift
struct OnboardingFlowStateMachineDelegate {
  typealias StateType = OnboardingFlow

  func shouldTransition(from: OnboardingFlow, to: OnboardingFlow) -> Bool {
    switch (from, to) {
    case (.initial, .pushNotificationsAsk):
      return true
    case (.pushNotificationsAsk, .pushNotificationsDisplay):
      return true
    case (.pushNotificationsAsk, .locationAsk):
      return true
    case (.pushNotificationDisplay, .locationAsk):
      return true
    case (.locationAsk, .locationDisplay):
      return true
    case (.locationDisplay, .onboarded):
      return true
    case (.locationAsk, .onboarded):
      return true
    default:
      return false
    }
  }

  func didTransition(from: BasicState, to: BasicState) {
    switch (from, to) {
    case (.initial, .pushNotificationAsk):
      showPushNotificationScreen()
    case (.pushNoficiationAsk, .pushNotificationDisplay):
      requestNotificationPermissions()
    case (_, .locationAsk):
      showLocationScreen()
    case (.locationAsk, .locationDisplay):
      requestLocationPermissions()
    case (_, .onboarded):
      showOnboarded()
    default:
      break
    }
  }
}
```

The above demonstrates what can be done. With regard to having limited code, it's often best to be expressive and this is what's been done. There's the ability to prevent the user from navigating outside of the defined flow. This means that if there's any business rules which must happen at a particular step or if there's some setup required it can be cleanly defined. By placing the navigation into the result of the transition, we can cleanly define that as well.

The use of a state machine is a powerful tool for handling complex and intricate interactions in an app. I encourage you to use it wisely. It's not a solution that fits everything, it's something that makes sense for a particular type of situation. Remember, with great power comes a tendency to misuse it.
