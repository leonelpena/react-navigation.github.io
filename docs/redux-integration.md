---
id: redux-integration
title: Redux integration
sidebar_label: Redux integration
---

Some folks like to have their navigation state stored in the same place as the rest of their application state. Using Redux to handle your state enables you to write custom actions that manipulate the navigation state directly, to be able to dispatch navigation actions from anywhere (sometimes in a "thunk" or "saga"), and to persist the navigation state in the same way you would other Redux state (your mileage may vary on this). You can read more about other use cases in the replies to [this tweet](https://twitter.com/satya164/status/952291726521024512).

## Warning!

*You probably do not need to do this!* Storing your React Navigation state in your own Redux store is likely to give you a very difficult time if you don't know what you're doing. You lose out on some performance optimizations that React Navigation can do for you, for example. Please do not integrate your state into Redux without first checking if you can do what you need to do without it!

## Overview

1. To handle your app's navigation state in Redux, you can pass your own [`navigation`](navigation-prop.html) prop to a navigator. This `navigation` prop is an object that must contain the navigation [`state`](navigation-prop.html#state-the-screen-s-current-state-route), [`dispatch`](navigation-prop.html#dispatch-send-an-action-to-the-router), and `addListener` properties.

2. The aforementioned navigation `state` will need to be kept updated using React Navigation's navigation reducer. You will call this reducer from your Redux master reducer.

3. React Navigation's `dispatch` will just be Redux's default `dispatch`, which you pass in to React Navigation as part of the `navigation` prop. As such, you will be able to dispatch normal Redux actions using `this.props.navigation.dispatch(ACTION)`.

4. A middleware is needed so that any events that mutate the navigation state properly trigger the event listeners. To properly trigger the event listeners with the initial state, a call to `initializeListeners` is also necessary.

## Step-by-step guide

First, you need to add the `react-navigation-redux-helpers` package to your project.

  ```bash
  yarn add react-navigation-redux-helpers
  ```

  or

  ```bash
  npm install --save react-navigation-redux-helpers
  ```

The following is a minimal example of how you might use navigators within a Redux application:

```es6
import {
  createStackNavigator,
} from 'react-navigation';
import {
  createStore,
  applyMiddleware,
  combineReducers,
} from 'redux';
import {
  createNavigationPropConstructor,       // handles #1 above
  createNavigationReducer,               // handles #2 above
  createReactNavigationReduxMiddleware,  // handles #4 above
  initializeListeners,                   // handles #4 above
} from 'react-navigation-redux-helpers';
import { Provider, connect } from 'react-redux';
import React from 'react';

const AppNavigator = createStackNavigator(AppRouteConfigs);

const navReducer = createNavigationReducer(AppNavigator);
const appReducer = combineReducers({
  nav: navReducer,
  ...
});

// Note: createReactNavigationReduxMiddleware must be run before createNavigationPropConstructor
const middleware = createReactNavigationReduxMiddleware(
  "root",
  state => state.nav,
);
const navigationPropConstructor = createNavigationPropConstructor("root");

class App extends React.Component {

  componentDidMount() {
    initializeListeners("root", this.props.nav);
  }

  render() {
    const navigation = navigationPropConstructor(
      this.props.dispatch,
      this.props.nav,
    );
    return <AppNavigator navigation={navigation} />;
  }

}

const mapStateToProps = (state) => ({
  nav: state.nav,
});

const AppWithNavigationState = connect(mapStateToProps)(App);

const store = createStore(
  appReducer,
  applyMiddleware(middleware),
);

class Root extends React.Component {
  render() {
    return (
      <Provider store={store}>
        <AppWithNavigationState />
      </Provider>
    );
  }
}
```

Once you do this, your navigation state is stored within your Redux store, at which point you can fire navigation actions using your Redux dispatch function.

Keep in mind that when a navigator is given a `navigation` prop, it relinquishes control of its internal state. That means you are now responsible for persisting its state, handling any deep linking, [Handling the Hardware Back Button in Android](#handling-the-hardware-back-button-in-android), etc.

Navigation state is automatically passed down from one navigator to another when you nest them. Note that in order for a child navigator to receive the state from a parent navigator, it should be defined as a `screen`.

Applying this to the example above, you could instead define `AppNavigator` to contain a nested `TabNavigator` as follows:

```es6
const AppNavigator = createStackNavigator({
  Home: { screen: MyTabNavigator },
});
```

In this case, once you `connect` `AppNavigator` to Redux as is done in `AppWithNavigationState`, `MyTabNavigator` will automatically have access to navigation state as a `navigation` prop.

## Full example

There's a working example app with Redux [here](https://github.com/react-community/react-navigation/tree/master/examples/ReduxExample) if you want to try it out yourself.

## Mocking tests

To make jest tests work with your React Navigation app, you need to change the jest preset in the `package.json`, see [here](https://facebook.github.io/jest/docs/tutorial-react-native.html#transformignorepatterns-customization):


```json
"jest": {
  "preset": "react-native",
  "transformIgnorePatterns": [
    "node_modules/(?!(jest-)?react-native|react-navigation|react-navigation-redux-helpers)"
  ]
}
```

## Under the hood

### Creating your own navigation reducer

If you want to replace `createNavigationReducer` reducer creator this is how you would do it yourself:

```es6
const AppNavigator = StackNavigator(AppRouteConfigs);

const initialState = AppNavigator.router.getStateForAction(AppNavigator.router.getActionForPathAndParams('Login'));

const navReducer = (state = initialState, action) => {
  const nextState = AppNavigator.router.getStateForAction(action, state);

  // Simply return the original `state` if `nextState` is null or undefined.
  return nextState || state;
};
```

## Handling the Hardware Back Button in Android

By using the following snippet, your nav component will be aware of the back button press actions and will correctly interact with your stack. This is really useful on Android.

```es6
import React from "react";
import { BackHandler } from "react-native";
import { NavigationActions } from "react-navigation";

/* your other setup code here! this is not a runnable snippet */

class ReduxNavigation extends React.Component {
  componentDidMount() {
    BackHandler.addEventListener("hardwareBackPress", this.onBackPress);
  }

  componentWillUnmount() {
    BackHandler.removeEventListener("hardwareBackPress", this.onBackPress);
  }

  onBackPress = () => {
    const { dispatch, nav } = this.props;
    if (nav.index === 0) {
      return false;
    }

    dispatch(NavigationActions.back());
    return true;
  };

  render() {
    /* more setup code here! this is not a runnable snippet */ 
    return <AppNavigator navigation={navigation} />;
  }
}
```
