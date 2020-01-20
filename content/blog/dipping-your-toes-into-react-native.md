---
path: dipping-your-toes-into-react-native
date: 2016-04-15T18:24:56.570Z
title: "Dipping Your Toes into React Native"
description: "When I took over as CTO at CompanyCam we had a universal app written in Xamarin that was horrendous. My first call was to rewrite both apps natively for Android and iOS using Java and…"
---

When I took over as CTO at CompanyCam we had a universal app written in Xamarin that was horrendous. My first call was to rewrite both apps natively for Android and iOS using Java and Objective-C/Swift respectively. Upon completion the performance of the apps improved and the bugs decreased significantly.

A constant problem as a small startup is that you are always short on developers. In our case this meant our apps varied in the features they provided. On top of handling our webapp, I also write our iOS app, which for our users means that Android gets features late if at all.

## Benefits of React Native

In case you aren’t familiar with React Native some benefits include:

- The node/npm ecosystem: Use packages provided by npm allowing you to write less code. This also allows you to write your own packages for use across iOS/Android.
- Hot Reloading: As soon as you save your file it reloads on the device; in most cases it preserves your spot and state in the app.
- Flexbox: Let your designers do their thing and style the app using technologies they are already familiar with. The flexbox model is a little different but Jared, our designer, figured it out in 10 minutes.
- Chrome Debug Tool: You can debug your code and log to the console from within your JS.
- Easy of Integration: It is almost too easy integrating it with an existing app. Add `RCT_EXPORT_MODULE();` to the file and expose methods you would like to call from JS with `RCT_EXPORT_METHOD`.
- It’s just JavaScript: If you have team members familiar with JavaScript they can start to contribute to iOS and Android. Granted some parts require some knowledge of the platform, but in general they don’t.

## Testing the Waters

To test how easy it was to integrate we decided to rewrite our login screen in React Native. The screen is fairly simple and looks like:

<figure>

![screenshot](/assets/dipping-your-toes-into-react-native/screenshot.jpeg)

</figure>

The trick with this is that for now we wanted to keep using our API helper to make the HTTP requests. To make that function we added:

```objective-c
RCT_EXPORT_METHOD(startLoginWithUsername:(NSString *)username password:(NSString *)password callback:(RCTResponseSenderBlock)callback).
```

The callback allows us to perform work that may take an undetermined amount of time. One thing to note is that this method gets called off of the main thread so if you require the main thread use:

```objective-c
dispatch_async(dispatch_get_main_queue(), ^{/* Code here */});
```

This will allow you to do whatever you need on the main thread. Calling this method from React is as simple as creating a function in your component that is called when the user performs a specific action, in our case tapping the “Sign In” button. Our function looks like:

```javascript
submitForm = () => {
  // Get the class you exposed with RCT_EXPORT_MODULE();
  var LoginViewController = NativeModules.LoginViewController;
```

```javascript
 // Method are changed to parameters for the JS function
  LoginViewController.startLoginWithUsername(this.state.email, this.state.password, function() {
     console.log(‘Login Failed’);
   });
 }
```

In our case since this is our first screen written in React Native. If the login is successful we replace the current view controller with a native iOS view controller. However, if the call fails we show an alert and then call the callback allowing us to clear the password field.

One of my favorite parts is that React Native is intuitive and makes simple things simple. For example, when a user taps to enter text into a textfield on native iOS you are required to manually calculate the height of the keyboard and then adjust scrollview contents to accommodate scrolling the textfield back into view. React Native handles this nearly automatically by using:

```javascript
let scrollResponder = this.refs.scrollView.getScrollResponder()
scrollResponder.scrollResponderScrollNativeHandleToKeyboard(
  React.findNodeHandle(this.refs[refName]), // The ref of the input
  110, //additionalOffset
  true
)
```

That small snippet of code handles everything that used to require calculating heights, modifying layout constraints and scrollview content sizes.

## Wrap Up

I know this doesn’t seem overly complex or dynamic, but that is the point! We wanted to take a simple screen and see how easy it was to do marginally complex tasks such as working with native Objective-C. We are getting ready to do a major over-hall of our app and are planning on using React Native to speed it up. Our long term goal is to share most of the code between our iOS and Android apps.
